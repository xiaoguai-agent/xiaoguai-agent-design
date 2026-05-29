# LLD — `xiaoguai-agent`

| | |
|---|---|
| Document ID | `LLD-AGENT-001` |
| Refines | `DEC-HLD-006` (HotL allow-then-escalate), `FLOW-CHAT-001` (chat lifecycle) |
| Verifies | `REQ-CHAT-001`, `REQ-CHAT-002`, `REQ-CHAT-004`, `REQ-CHAT-005`, `REQ-HOTL-001`, `REQ-HOTL-002`, `REQ-HOTL-003`, `BEH-CHAT-001`, `BEH-HOTL-001` |

## 1. Module purpose

Implements the ReAct loop. Consumes:

- `LlmRouter` (xiaoguai-llm) for streaming LLM responses.
- `Toolbox` (xiaoguai-mcp) for tool calls.
- `HotlGate` for budget enforcement.
- `Storage` for message persistence + audit append.
- `CancellationRegistry` (xiaoguai-runtime) for session cancel.

Emits an `agent::Stream` consumed by `xiaoguai-api` for SSE.

## 2. Public interface

```rust
pub struct AgentLoop {
    llm: Arc<LlmRouter>,
    toolbox: Toolbox,
    hotl: Arc<HotlGate>,
    storage: Storage,
    cancel: Arc<CancellationRegistry>,
    config: AgentConfig,
}

pub struct AgentConfig {
    pub max_iterations: u32,        // default 25; per-tenant override
    pub history_window: usize,      // sliding window before compaction
    pub compact_threshold_pct: u8,  // default 70 — trigger LLM-summary compaction
}

impl AgentLoop {
    pub fn stream(self: Arc<Self>, req: AgentRequest) -> impl Stream<Item = AgentEvent>;
}

pub enum AgentEvent {
    Delta(String),
    ToolCall { id: String, name: String, args: serde_json::Value },
    ToolResult { id: String, output: serde_json::Value, duration_ms: u64 },
    Final { message_id: MessageId, stop_reason: StopReason },
    Error(AgentError),
}
```

## 3. Module structure

```
crates/xiaoguai-agent/
├── src/
│   ├── lib.rs               # AgentLoop, stream entry point
│   ├── react.rs             # ReAct loop state machine
│   ├── pre_processor.rs     # history compaction, system prompt injection
│   ├── post_processor.rs    # token usage, message persistence
│   ├── hotl.rs              # HotlGate trait + StorageHotlGate impl
│   ├── parallel.rs          # tokio::join_all over allowed tool calls
│   ├── config.rs
│   └── error.rs
└── tests/
    ├── react_loop.rs
    ├── hotl_escalate.rs
    └── cancel.rs
```

## 4. Key flows

### 4.1 ReAct loop iteration

```
iteration:
  ├─ if cancel.check(session_id): emit Final(cancelled); break
  ├─ stream = llm.chat(history)
  ├─ relay Delta events until stream yields tool_calls or stop
  ├─ if tool_calls present:
  │     gated_calls = []
  │     for call in tool_calls:
  │       gate_result = hotl.check(tenant, scope=tool_call.<name>, amount=1)
  │       match gate_result:
  │         Allowed         => gated_calls.push((Allowed, call))
  │         Escalated{ to } => synthetic_result + audit + push escalate channel
  │         PolicyUnavailable => synthetic_result + audit + agent stops
  │     futures = gated_calls.map(|c| toolbox.call(c)).join_all
  │     emit ToolCall + ToolResult events as they complete
  │     append results to history
  ├─ else: emit Final(stop); persist assistant message; record token_usage; break
  └─ iteration += 1; if iteration >= max_iterations: emit Final(max_iterations); break
```

### 4.2 Parallel tool dispatch

`tokio::join_all` over the gated set. Tool failures do not abort the iteration; their errors are surfaced as `ToolResult { output: { error: ... } }` so the LLM can recover.

### 4.2a `propose_skill` synthetic tool (v0.5.4.2, T3, PR #72)

`crates/xiaoguai-agent/src/skill_author_tool.rs` registers a synthetic `McpClient` impl exposing **one tool** — `propose_skill` — with a strict JSON schema (`additionalProperties: false`, required fields enforced). The client is NOT loaded from `mcp_servers`; it's constructed in `xiaoguai-core::run_serve` only when `AppState.skill_proposals + tenant_settings + skill_author_gate` are all `Some`.

```rust
pub struct ProposeSkillClient<B: SkillAuthorBackend> {
    inner: B,                        // typically xiaoguai-tasks::skill_author bridge
    tenant_id: String,
    proposed_by: String,
}

impl<B: SkillAuthorBackend> McpClient for ProposeSkillClient<B> {
    fn descriptors(&self) -> Vec<ToolDescriptor> {
        vec![Self::descriptor()]   // returns the propose_skill ToolDescriptor
    }
    async fn call_tool(&self, name: &str, args: Value) -> McpResult<ToolResult> {
        if name != "propose_skill" { return Err(McpError::Protocol(...)); }
        let parsed: ProposeSkillArgs = serde_json::from_value(args)?;
        self.inner.invoke(&self.tenant_id, &self.proposed_by, parsed).await
    }
}
```

Schema enforces:
- `name` (string, required)
- `description` (string, required)
- `version` (string matching semver, required)
- `system_prompt` (string, required, ≤ 8 KB)
- `tool_allowlist` (array of strings, required, non-empty)
- `reason` (string, required, ≤ 1 KB)
- `additionalProperties: false` — any extra field is rejected by Pydantic/Zod-style validators on the LLM side.

The agent loop treats this tool like any other MCP tool: the per-tool HotL gate (PR #61) runs first under scope `tool_call.propose_skill`, and *separately* the skill-author flow consults its own bucket `skill_author`. Two gates, two budgets — `tool_call.propose_skill` controls whether the agent can even attempt; `skill_author` (in `xiaoguai-tasks`) caps daily proposals after attempting.

The route from a `Role::Tool` reply back to the model carries the lifecycle status (`Pending` / `Denied(reason)`) — the agent observes the outcome and can revise its proposal in the next turn (validated by the E2E "revise after deny" test).

### 4.3 Cancellation

`CancellationRegistry` is checked at every iteration boundary AND at every tool-call boundary (before spawn). Bounded latency is the slowest in-flight tool.

### 4.4 Compaction (shipped in v0.5.4.1, Tier-2d)

`crates/xiaoguai-agent/src/history.rs` (PR #66) ships LLM-summarisation compaction alongside the existing `slide()`. The ReAct loop now decides per iteration:

```rust
// crates/xiaoguai-agent/src/react.rs (inside run_inner)
if let Some(cfg) = config.compaction {
    if should_compact(&messages, cfg) {
        let (next, outcome) = compact(messages, backend.as_ref(), cfg).await;
        messages = next;
        // emit Prometheus + tracing per outcome
    } else {
        messages = slide(messages, config.history_window);
    }
} else {
    messages = slide(messages, config.history_window);  // legacy
}
```

**`CompactionConfig` (public)**:

| Field | Default | Meaning |
|---|---|---|
| `max_context_tokens` | `30_000` | Hard ceiling the model can accept. Set per model — see `runbooks/compaction.md` for the per-model table. |
| `trigger_at_pct` | `75` | Percentage of `max_context_tokens` at which compaction kicks in. |
| `keep_recent` | `6` | Recent non-system messages kept verbatim (≈ 3 user/assistant pairs). |
| `summary_model` | `"qwen2.5-coder"` | Model identifier passed to `backend.chat_stream` for summarisation. |

**Algorithm (six steps, executes within `compact()`):**

1. Partition `messages` into `system` (kept verbatim) and `tail`.
2. If `tail.len() <= keep_recent` → return original, `CompactionOutcome::NoOp`.
3. Compute split = `tail.len() - keep_recent`. **Walk the split backwards** past any assistant-`tool_calls` ↔ `Role::Tool` pair so the split never lands between a tool call and its reply (preserves `tool_call_id` invariant).
4. Render the older head as plain text and request a ≤ 500-token summary from `backend` with a fixed dense-summary system prompt (keep concrete facts, drop pleasantries, never invent).
5. **Success path** — replace the older head with one synthetic `Role::System` message tagged `[Compacted summary of N earlier messages]\n\n…` followed by `recent` verbatim. Strip leading `Tool` messages from `recent` (same invariant as `slide`).
6. **Fallback path** — on network/empty/backend error, emit `tracing::warn!` and fall through to `slide(messages, keep_recent + 1)`. Returns `CompactionOutcome::FellBack`.

**Outcome enum** — `Compacted` / `FellBack` / `NoOp`. Each variant has a distinct Prometheus emission (see §7).

**Tool-pair preservation** — `walk_split_back_past_tool_pairs` looks back from the proposed split and pulls it left until both sides of every tool_calls↔Tool pair land on the same side of the cut. Codified in unit test `compact_preserves_tool_pairing`.

**Token estimation** — `xiaoguai-llm::estimate_message_tokens` is a conservative 4-chars/token whitespace estimator with a 4-token per-message overhead. Counts `char` not `byte` so CJK content is not double-counted. Rounds up.

**Opt-in semantics** — `AgentConfig.compaction: Option<CompactionConfig>` defaults to `None`. Existing callers see byte-identical behaviour to v0.5.4. Operators enable per tenant via `agent.compaction.enabled` in `local.yaml` (also `local.yaml.example`). Anthropic / OpenAI tenants should typically keep this off and rely on provider-native caching instead.

**Cross-references**: PHILO-XIAOGUAI-001 §11 (token transformation pipeline) and §12 (Function Calling lifecycle step 4 invariant) motivate the tool-pair guard. DEC-013 in HLD codifies the decision.

## 5. Error handling

```rust
pub enum AgentError {
    Llm(LlmError),
    Tool(McpError),
    Hotl(HotlError),
    Cancelled,
    MaxIterations,
}
```

`MaxIterations` and `Cancelled` are not errors — they end the stream cleanly via `Final` event.

## 6. Concurrency / transactions

- One loop per session per request; sessions are serial (no concurrent agent runs on the same session id).
- Within an iteration, tool calls are parallel.
- Persistence (message + audit row) is one transaction per iteration.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | `react.rs` state machine: one-shot reply, multi-tool reply, escalated reply, cancelled reply. `history.rs` slide: noop_when_under_window, drops_oldest_non_system, drops_dangling_tool_message_at_new_head, window_zero_is_unbounded. `history.rs` compact: compact_noop_when_under_keep_recent, compact_replaces_head_with_summary, compact_falls_back_to_slide_on_backend_error, compact_preserves_tool_pairing, should_compact_threshold. |
| Integration | `hotl_escalate.rs` — exceeded budget yields synthetic result + audit row; `cancel.rs` — cancel mid-tool returns Final(cancelled) within slowest-tool-deadline. **`compaction_integration.rs`** (v0.5.4.1): `compaction_shrinks_large_history_via_summary` — 100-turn synthetic conversation with 30 tool-call cycles compacts to ≤ 50 % of `max_context_tokens`; `compaction_preserves_recent_turns_verbatim` — last 6 messages bit-identical post-compaction. |
| System | Test strategy L2: SSE event sequence shape; BEH-CHAT-001. Compaction metrics: `xiaoguai_compaction_triggered_total` increments per trigger; `xiaoguai_compaction_fallback_total / triggered` < 5 % in healthy deployments. |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-AGENT-001
  type: LLD
  title: xiaoguai-agent LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents: [HLD-XIAOGUAI-001, API-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-AGENT-001, title: "HotL gate before tool spawn", statement: "Every tool call passes through HotlGate.check before dispatch.", status: approved, scope: in, decision: "Gate-first.", rationale: "Prevents budget evasion via fast-spawn races." }
    - { id: DEC-LLD-AGENT-002, title: "Cancel polled at boundaries", statement: "Cancellation polled at iteration and tool-call boundaries.", status: approved, scope: in, decision: "Coarse-grained cancel.", rationale: "Avoids mid-LLM-stream tear-down complexity." }
    - { id: DEC-LLD-AGENT-003, title: "Compaction deferred to Tier-2d", statement: "History compaction not implemented in v1.4.", status: approved, scope: in, decision: "Truncate-only for now.", rationale: "Compaction policy design is open (PRD Q5)." }
  flows:
    - { id: FLOW-LLD-AGENT-001, title: "ReAct iteration with HotL", statement: "Stream LLM, gate tool calls, parallel dispatch, append history, loop or finalise.", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-AGENT-002, title: "Cancellation propagation", statement: "Registry poll at boundaries; bounded by slowest tool.", status: approved, scope: in, kind: state_transition }
  test_cases: []
relations:
  - { id: REL-LLD-AGENT-001, type: refines, from: DEC-LLD-AGENT-001, to: DEC-HLD-006, status: active }
  - { id: REL-LLD-AGENT-002, type: refines, from: FLOW-LLD-AGENT-001, to: FLOW-CHAT-001, status: active }
  - { id: REL-LLD-AGENT-003, type: refines, from: FLOW-LLD-AGENT-002, to: FLOW-CANCEL-001, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
