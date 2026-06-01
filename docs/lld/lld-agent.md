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

### 4.5 HotL suspension wiring (sprint-12, S11-3a.2)

> **Status:** new section added in sprint-12 design pass. Sprint-11 closed the chat-ui inline buttons + the `POST /v1/hotl/decisions` route (see [`lld-chat-ui.md`](lld-chat-ui.md) §4.3.1 and [`api-contract.md`](../api-contract.md) §2.6.2). This section specifies the agent-side wiring that flips the route's `resumed: false` seam to a live waiter.

Sprint-11 shipped the seam: `EnforcerGate` maps upstream `HotlVerdict::Escalate` to `HotlGateVerdict::Allow` plus a `tracing::warn`. The loop **does not pause** on escalation — see `crates/xiaoguai-core/src/hotl_bridge.rs:361`. Operators reviewed pending escalations out-of-band; their decisions were recorded but had no live effect.

Sprint-12 keeps that fall-through as the legacy compatibility path and adds a second adapter that **does** suspend. The choice between adapters is per-tenant (config flag `agent.hotl.suspend_on_escalate: bool`, default `false` in v1.8.x for backward compatibility, default `true` from v1.9 onwards).

**Additive verdict variant** (extends `HotlGateVerdict` in `crates/xiaoguai-agent/src/hotl_gate.rs`):

```rust
pub enum HotlGateVerdict {
    Allow,
    Deny(String),
    /// New (sprint-12). Loop must NOT dispatch the tool yet — it
    /// must emit `AgentEvent::HotlPending`, then await `ticket.await`
    /// to receive an operator verdict from `DecisionRegistry`.
    Suspend {
        request_id: Uuid,
        scope: String,
        ticket: HotlSuspensionTicket,
    },
}

/// One-shot receiver paired with a sender held by `DecisionRegistry`
/// on `AppState`. Future-like: awaits a `HotlDecisionVerdict { verdict, decided_by }`
/// or times out at `expires_at` (24h default, configurable per scope).
pub struct HotlSuspensionTicket { /* inner: oneshot::Receiver<...> + Instant */ }
```

The variant is **additive** — existing `EnforcerGate` keeps returning `Allow`/`Deny` and ignores the new path; only the new `SuspendingHotlGate` (in `xiaoguai-core::hotl_bridge`) ever emits `Suspend`. Adapter selection happens in `xiaoguai-core::run_serve` based on the config flag.

**Additive `AgentEvent` variants** (extends `crates/xiaoguai-agent/src/event.rs`):

```rust
pub enum AgentEvent {
    // ... existing variants ...

    /// New (sprint-12). Tool dispatch paused; waiting on operator decision.
    /// SSE wire shape: see api-contract §2.6.3.
    HotlPending {
        request_id: Uuid,
        tool: String,
        args_redacted: JsonValue,
        scope: String,
        expires_at: DateTime<Utc>,
    },

    /// New (sprint-12). Emitted after the ticket resolves (operator verdict
    /// OR server-side timeout). Always follows a HotlPending with the same
    /// `request_id`. The tool's eventual outcome appears as the normal
    /// ToolCallFinished event that follows (when verdict = "allow") or as a
    /// synthetic ToolCallFinished with ok=false (when verdict = "deny" / "timeout").
    HotlResolved {
        request_id: Uuid,
        verdict: HotlResolution,   // Allow | Deny | Timeout
        decided_by: Option<String>,
        recorded_at: DateTime<Utc>,
    },
}
```

**Loop integration** — extends §4.1's `match gate_result` arm. The key design choice (DEC-LLD-AGENT-004 below): suspension blocks **within the existing iteration**, it does not introduce a new outer state machine.

```
iteration:
  ...
  if tool_calls present:
    gated_calls = []
    for call in tool_calls:
      gate_result = hotl.check(tenant, scope=tool_call.<name>, amount=1)
      match gate_result:
        Allowed                     => gated_calls.push((Allowed, call))
        Suspend{request_id, ticket} =>                                   # NEW (sprint-12)
          emit HotlPending{request_id, tool, args_redacted, scope, expires_at}
          match ticket.await:                                            # blocks the iteration
            Ok(HotlDecisionVerdict{ Allow, decided_by }) =>
              emit HotlResolved{request_id, verdict=Allow, decided_by, recorded_at=now}
              gated_calls.push((Allowed, call))
            Ok(HotlDecisionVerdict{ Deny(reason), decided_by }) =>
              emit HotlResolved{request_id, verdict=Deny, decided_by, recorded_at=now}
              synthesise failed ToolResult { error: reason }; do NOT dispatch
            Err(Timeout) =>
              emit HotlResolved{request_id, verdict=Timeout, decided_by=None, recorded_at=now}
              synthesise failed ToolResult { error: "operator decision timed out" }
            Err(Cancelled) =>                                            # cancellation registry fired
              # do NOT emit HotlResolved (the cancel path emits Final(cancelled))
              return
        Escalated{ to } => synthetic_result + audit + push escalate channel  # legacy, suspend_on_escalate=false
        Denied(reason)  => synthesise failed ToolResult
        PolicyUnavailable => synthetic_result + audit + agent stops
    futures = gated_calls.map(|c| toolbox.call(c)).join_all
    ...
```

**Why block inside the iteration (not lift to an outer state machine)**:

- The session is already serialised — §6 guarantees one loop per session. There is no concurrent work that suspension can starve.
- Operator decisions are slow (seconds to hours), but so are LLM calls (seconds to minutes). The existing `await` discipline accommodates this. A separate state machine would have to re-derive the same per-tool-call locality the loop already owns.
- The cancellation registry already covers the "operator never decides + user closes tab" case via the existing iteration-boundary poll plus a select between `ticket.await` and `cancel.observe()`. No new cancel surface.

**DecisionRegistry** (lives on `AppState` in `xiaoguai-api`, depended on by both the route handler and the gate adapter — see `lld-chat-ui.md` §4.3 for the frontend contract):

```rust
pub struct DecisionRegistry {
    waiters: DashMap<Uuid, oneshot::Sender<HotlDecisionVerdict>>,
}

impl DecisionRegistry {
    /// Called by `SuspendingHotlGate::check` immediately before
    /// returning `Suspend{...}`. Returns the receiver half wrapped
    /// in a `HotlSuspensionTicket`.
    pub fn register(&self, request_id: Uuid) -> HotlSuspensionTicket;

    /// Called by `POST /v1/hotl/decisions` handler after recording
    /// the decision. Returns `true` iff a live waiter consumed the verdict —
    /// this is the value that flips the route's `resumed` response field
    /// (see api-contract §2.6.2). Returns `false` when no waiter exists
    /// (decision arrived late; race; or the loop was never suspended).
    pub fn resolve(&self, request_id: Uuid, verdict: HotlDecisionVerdict) -> bool;
}
```

**Timeout** — the gate adapter spawns a `tokio::time::sleep_until(expires_at)` companion future when it registers. On timeout, it sends a `HotlDecisionVerdict::Timeout` to the receiver and unregisters the entry. The route handler's `409 Conflict` semantics keep working: a late `POST /v1/hotl/decisions` after the timeout sees an empty registry and falls through to `resumed: false` + records the decision (the operator's intent is preserved in audit even if it had no live effect).

**Persona suffix** — when the orchestrator runs multi-peer patterns, the gate scope already carries `.persona.<persona_id>` (see [`lld-orchestrator.md`](lld-orchestrator.md) §4.2). The `request_id` is freshly minted per-call so cross-persona collision is impossible at the registry layer. The persona suffix matters for **which policy fires**, not for waiter routing.

**Test design**: see §7's "Integration" row — sprint-12 adds `hotl_suspend.rs` (suspend → resolve happy path), `hotl_suspend_timeout.rs` (no operator → timeout synthesises Deny), `hotl_suspend_cancel.rs` (user cancels mid-suspension → cancel wins over operator).

### 4.6 HotL hardening (sprint-13)

> **Status:** ✅ shipped in sprint-13 (xiaoguai PRs #139, #141, #144, #145, #146, #147, #148, #142, #143, #149). Section retained for the design rationale. Refines §4.5 in four directions: registry persistence (DEC-HLD-013), policy-driven args redaction (DEC-HLD-014), per-scope timeouts (DEC-HLD-015), and the `escalation_id` rename + parent-table split (DEC-HLD-016). The §4.5 in-iteration suspension model is unchanged; this section specifies what changes around it.

**Identifier rename.** Everywhere in `xiaoguai-agent` and the cross-crate wire — `HotlGateVerdict::Suspend { request_id, ... }`, `AgentEvent::HotlPending { request_id, ... }`, `AgentEvent::HotlResolved { request_id, ... }`, `DecisionRegistry::register(request_id)`, `DecisionRegistry::resolve(request_id, ...)` — the field is renamed to `escalation_id`. The `#[serde(alias)]` shim is removed. SSE consumers that previously accepted both names now see only `escalation_id`. The new authoritative type is:

```rust
pub enum HotlGateVerdict {
    Allow,
    Deny(String),
    Suspend {
        escalation_id: Uuid,        // renamed from request_id (sprint-12)
        scope: String,
        ticket: HotlSuspensionTicket,
    },
}
```

**DecisionRegistry persistence.** The registry gains a storage dependency:

```rust
pub struct DecisionRegistry {
    waiters: DashMap<Uuid, oneshot::Sender<HotlDecisionVerdict>>,
    store: Arc<dyn HotlEscalationStore>,   // sprint-13: PG-backed in production
}

#[async_trait]
pub trait HotlEscalationStore: Send + Sync {
    /// Called when SuspendingHotlGate registers a new escalation.
    /// Persists the parent hotl_escalations row + the child hotl_pending row.
    async fn insert_pending(&self, parent: HotlEscalationRow, child: HotlPendingRow) -> Result<(), StorageError>;

    /// Called by POST /v1/hotl/decisions before the in-process oneshot fires.
    /// Atomic update of status + decided_by + decided_at.
    async fn record_decision(&self, escalation_id: Uuid, verdict: HotlDecisionVerdict) -> Result<(), StorageError>;

    /// Called by run_serve at boot: returns all pending+unexpired rows.
    async fn list_pending_unexpired(&self, now: DateTime<Utc>) -> Result<Vec<HotlPendingRow>, StorageError>;
}
```

`xiaoguai-core::run_serve` calls `list_pending_unexpired` immediately after `AppState` construction and before the HTTP server begins accepting requests. For each row it: (a) mints a fresh `oneshot::channel`; (b) inserts the sender into `waiters`; (c) spawns a `tokio::time::sleep_until(row.expires_at)` companion future bound to the same `escalation_id`. The replay path is logged at `info` level and the count is exported via `xiaoguai_hotl_registry_replayed_total{outcome}`.

Live oneshot delivery latency is unchanged from §4.5 (DashMap O(1) lookup). The persistence dependency adds one PG write per `register` and one per `resolve`; both are inside the existing per-iteration transaction in §6 so they do not introduce a new transaction boundary.

**Args redaction.** The `args_redacted` field on `AgentEvent::HotlPending` is computed by a new policy hook:

```rust
// Lives in xiaoguai-auth::redaction
pub struct RedactionRules { /* opaque, loaded per-tenant per-request */ }

impl RedactionRules {
    pub fn from_storage(s: &Storage, tenant_id: TenantId) -> impl Future<Output = Result<Self, AuthError>>;
    pub fn apply(&self, scope: &str, args: &serde_json::Value) -> serde_json::Value;  // replaces matched JSONPath nodes with "***"
}
```

`SuspendingHotlGate` resolves the rules from `AppState` and calls `apply(scope, args)` before constructing the `HotlPending` event. Empty rules degrade to `args.clone()` and emit a single `tracing::warn!` per tenant per boot ("no HotL redaction policy configured — args emitted verbatim"). The audit row that pairs with the `HotlPending` carries the resolved `redaction_policy_id` so the audit chain can prove which policy version was applied.

**Per-scope timeouts.** `SuspendingHotlGate::check` no longer reads a single `default_expiry`. The new lookup:

```rust
fn resolve_expiry(cfg: &HotlConfig, scope: &str) -> Duration {
    let class = scope.split_once('.').map(|(c, _)| c).unwrap_or(scope);
    cfg.expiry.get(class).copied().unwrap_or(cfg.default_expiry)
}
```

The `default_expiry` field stays in `HotlConfig` as the documented fallback (DEC-HLD-015). Lookup is per-call, not cached — tenants editing their config at runtime are honoured on the next escalation.

**Why parent/child schema matters for the loop.** The split into `hotl_escalations` (parent) + `hotl_pending` (child) does not change the loop's per-tool-call locality (one `Suspend` verdict → one ticket → one `resolve`). It only changes what `insert_pending` writes: an iteration that triggers multiple gates (e.g., DEC-021 triangle with per-peer HotL) writes one parent row plus N children sharing that `escalation_id` lineage. The loop code remains §4.5's single-`Suspend` arm; the registry's `register` call carries the parent id when one already exists for the iteration.

**Test design** — see §7's "Integration" row: sprint-13 adds `hotl_persistence_replay.rs` (5 pending rows survive a simulated restart and resolve correctly), `hotl_args_redaction.rs` (password field masked before SSE emit), `hotl_per_scope_expiry.rs` (mcp.* scope picks 4h over default 24h), `hotl_escalation_id_rename.rs` (regression — old `request_id` payload rejected with 400), `hotl_decide_scope.rs` (operator without `hotl:decide` scope gets 403).

### 4.7 API scope-gate extractor (sprint-14 — DEC-HLD-018)

Sprint-13's S13-10 wired the `hotl:decide` Casbin scope into `POST /v1/hotl/decisions` as an inline check inside the route body. Sprint-14 extracts the check into a reusable axum extractor — a precondition for adding the same gate to the new `/v1/admin/hotl-redaction-policies` routes (DEC-HLD-017) and the queued `POST /v1/audit/exports/approve` route. The extractor lives in a new module `crates/xiaoguai-api/src/middleware/require_scope.rs`:

```rust
// xiaoguai-api::middleware::require_scope
pub struct RequireScope<const SCOPE: &'static str>(pub Claims);

#[async_trait]
impl<const SCOPE: &'static str, S> FromRequestParts<S> for RequireScope<SCOPE>
where
    S: Send + Sync,
    AppState: FromRef<S>,
{
    type Rejection = ApiError;

    async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
        let claims = Claims::from_request_parts(parts, state).await?;     // sprint-11 JWT layer
        if !claims.scopes.iter().any(|s| s == SCOPE) {
            return Err(ApiError::scope_required(SCOPE));                  // 403 with ErrorEnvelope.code = "scope_required"
        }
        Ok(RequireScope(claims))
    }
}
```

Handler composition becomes:

```rust
async fn decide(
    RequireScope(claims): RequireScope<"hotl:decide">,
    State(state): State<AppState>,
    Json(body): Json<HotlDecisionRequest>,
) -> Result<Json<HotlDecisionResponse>, ApiError> {
    // claims.sub is the operator user_id; can thread through to record_decision when sprint-14
    // S14-? graduates from request-body decided_by (sprint-13 carry-forward #6).
    …
}
```

**Why an extractor, not tower middleware.** axum extractors compose with the handler signature and surface in the route's type, so the scope requirement is part of the OpenAPI generation surface and `cargo doc`. A tower layer would route-attach the gate at router-construction time, which is one layer of indirection away from the handler. The existing `RequireRole<&'static str>` and `RequireTenant` extractors set the precedent.

**Why JWT claims, not Casbin per-request.** The hybrid CSV+DB Casbin adapter from sprint-13's S13-10 merges policy at boot. Scopes for a role are stable for the lifetime of a token; reading them off the verified JWT is one in-memory `Vec::contains` rather than a DB round-trip per request. The trade-off is hot-reload: admin-ui rule edits don't take effect until the issuer re-mints tokens. Sprint-14 explicitly defers hot-reload (DEC-HLD-018 "Hot-reload" subsection); the carry-forward to sprint-15 is a SIGHUP-driven re-merge plus a token-revocation list.

**Existing inline check.** `routes/hotl_decisions.rs` after sprint-13 has roughly:

```rust
async fn decide(State(state): State<AppState>, claims: Claims, Json(body): Json<…>) -> … {
    if !claims.scopes.contains(&"hotl:decide".to_string()) {
        return Err(ApiError::Forbidden);
    }
    …
}
```

Sprint-14 migrates this to `RequireScope<"hotl:decide">` in the same PR that lands the extractor — the route's signature changes but the behaviour is byte-identical (same 403 status, same error envelope code).

**Test design** — see §7's "Integration" row: sprint-14 adds `require_scope_extractor.rs` covering: (a) operator with scope passes; (b) operator without scope gets 403 with `ErrorEnvelope.code = "scope_required"` and `scope: "hotl:decide"`; (c) anonymous request gets 401 (auth layer runs first); (d) two different scope-gated routes in the same router compile and route correctly (regression for the `const SCOPE` parameter).

### 4.8 Small carry-forwards from sprint-13 (sprint-14 — same PR series)

Two sprint-13 deferrals are small enough to land in sprint-14 alongside the DEC-030/031/032 work; both touch §4.5 and §4.6 surfaces and reuse the same test fixtures.

**Unknown escalation → 404.** S13-5 currently degrades `POST /v1/hotl/decisions` to `resumed=false` for unknown `escalation_id` (back-compat with sprint-12 routes that returned 200 + a flag). Sprint-13's parent-table presence assertion (DEC-HLD-016) now lets the route hard-distinguish "unknown id" from "known but already resolved/expired":

```rust
match store.lookup_parent(escalation_id).await? {
    None => Err(ApiError::not_found("escalation_id"))   // 404 — id never existed
    Some(row) if row.is_terminal() => Err(ApiError::conflict("escalation already resolved"))   // 409
    Some(_) => /* proceed to in-process oneshot delivery */
}
```

The `resumed=false` flag is removed from the response shape; chat-ui's HotlBanner already treats absent flag as "live". This is a breaking change in the response envelope; the chat-ui rename PR (sprint-13 S13-9) is the only known consumer.

**Replay batch cap.** S13-5's boot replay is unbounded — `list_pending_unexpired` reads every row matching the filter. For tenants accumulating thousands of pending escalations (R13-4 in sprint-13 plan), this is one big query + one big DashMap insertion at boot. Sprint-14 introduces a cap with cursor-based pagination:

```rust
pub trait HotlEscalationStore {
    async fn list_pending_unexpired_page(&self, cursor: Option<EscalationCursor>, limit: usize)
        -> Result<(Vec<HotlPendingRow>, Option<EscalationCursor>), StorageError>;
}
```

`run_serve` calls in a loop until the cursor returns `None`, yielding to the tokio runtime between pages so the HTTP server can start accepting traffic on the first page (subsequent pages replay in background). Default page size: 256 rows. Configurable via `agent.hotl.replay_page_size`. The total replayed count metric stays unchanged — the per-page invocation just increments incrementally.

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
| Integration | `hotl_escalate.rs` — exceeded budget yields synthetic result + audit row; `cancel.rs` — cancel mid-tool returns Final(cancelled) within slowest-tool-deadline. **`compaction_integration.rs`** (v0.5.4.1): `compaction_shrinks_large_history_via_summary` — 100-turn synthetic conversation with 30 tool-call cycles compacts to ≤ 50 % of `max_context_tokens`; `compaction_preserves_recent_turns_verbatim` — last 6 messages bit-identical post-compaction. **Sprint-12 (S11-3a.2)**: `hotl_suspend.rs` — gate returns Suspend → loop emits HotlPending → registry.resolve(Allow) → loop emits HotlResolved + dispatches tool; `hotl_suspend_timeout.rs` — ticket times out → HotlResolved(Timeout) + synthetic Deny ToolResult; `hotl_suspend_cancel.rs` — cancellation registry fires during ticket.await → Final(Cancelled), no HotlResolved emitted. **Sprint-13 (HotL hardening)**: `hotl_persistence_replay.rs` — 5 pending rows with mixed expiry survive an `AppState` rebuild; all 5 oneshot waiters reattach and a subsequent `POST /v1/hotl/decisions` resolves each; expired-on-replay rows synthesise verdict=timeout exactly once. `hotl_args_redaction.rs` — tool call with `{password: "x"}` arg under a tenant policy matching `$.password` emits `HotlPending.args_redacted = {password: "***"}`; audit row carries non-null `redaction_policy_id`. `hotl_per_scope_expiry.rs` — config sets `expiry.mcp = 4h`, default 24h; an `mcp.oauth.consent` escalation's `expires_at` is now+4h; a `tool_call.execute_python` escalation's `expires_at` is now+24h. `hotl_escalation_id_rename.rs` — `POST /v1/hotl/decisions` with body using legacy `request_id` field returns 400 with `field: escalation_id` discriminator error. `hotl_decide_scope.rs` — operator JWT without `hotl:decide` scope receives 403 on `POST /v1/hotl/decisions`. **Sprint-14 (admin-CRUD + extractor)**: `require_scope_extractor.rs` — operator with `hotl:decide` passes through extractor; operator without it gets 403 + `code: "scope_required"`; anonymous request gets 401; two scope-gated routes coexist in one router. `hotl_redaction_policy_crud.rs` — create policy → escalation banner shows masked args; PUT creates new revision and old `policy_id` becomes inactive but audit row pointing at old id remains resolvable; DELETE only deactivates; 409 on concurrent identical inserts; 409 on deactivate-of-last-active-rule when `redaction_policy_required=true`. `hotl_unknown_escalation_404.rs` — decision on unknown id returns 404; decision on terminal id returns 409. `hotl_replay_pagination.rs` — 1000 pending rows replay across 4 pages of 256; HTTP server starts after page 1; first request served while pages 2-4 still replaying. |
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
  updated_at: 2026-05-31
  source_documents: [HLD-XIAOGUAI-001, API-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-AGENT-001, title: "HotL gate before tool spawn", statement: "Every tool call passes through HotlGate.check before dispatch. Verdicts are Allow / Deny / Suspend (sprint-12+); Suspend blocks the iteration on a DecisionRegistry ticket before either dispatching (Allow) or synthesising a failed ToolResult (Deny/Timeout).", status: approved, scope: in, decision: "Gate-first with optional in-iteration suspension.", rationale: "Prevents budget evasion via fast-spawn races; suspension keeps operator decisions on the same per-tool-call locality the loop already owns." }
    - { id: DEC-LLD-AGENT-002, title: "Cancel polled at boundaries", statement: "Cancellation polled at iteration and tool-call boundaries.", status: approved, scope: in, decision: "Coarse-grained cancel.", rationale: "Avoids mid-LLM-stream tear-down complexity." }
    - { id: DEC-LLD-AGENT-003, title: "Compaction deferred to Tier-2d", statement: "History compaction not implemented in v1.4.", status: approved, scope: in, decision: "Truncate-only for now.", rationale: "Compaction policy design is open (PRD Q5)." }
    - { id: DEC-LLD-AGENT-004, title: "HotL suspension is in-iteration, not a new state machine", statement: "On HotlGateVerdict::Suspend the loop blocks within the existing iteration on a HotlSuspensionTicket (oneshot receiver + timeout) before dispatching tools, rather than lifting suspension to an outer state machine.", status: approved, scope: in, decision: "In-iteration await.", rationale: "Sessions are already serialised (one loop per session, §6); a separate state machine would re-derive the per-tool-call locality the loop already owns. Cancellation registry already covers the ‘operator never decides + user closes tab’ case via a select between ticket.await and cancel.observe()." }
    - { id: DEC-LLD-AGENT-005, title: "HotL hardening — persistence + redaction + per-scope expiry + escalation_id rename", statement: "DecisionRegistry depends on HotlEscalationStore for boot-time replay; SuspendingHotlGate resolves RedactionRules from xiaoguai-auth before emitting HotlPending; expiry lookup is per-scope-class with default fallback; all wire fields rename request_id→escalation_id (no alias).", status: approved, scope: in, decision: "PG-backed registry + policy-driven redaction in xiaoguai-auth + per-scope expiry config + clean rename.", rationale: "Sprint-12 MVP loses live waiters on restart, leaks raw args to operators, uses one timeout for fundamentally different scope cadences, and carries a two-name surface for one identifier. Sprint-13 closes all four in one cohesive hardening pass." }
    - { id: DEC-LLD-AGENT-006, title: "Scope-gate enforcement is a reusable axum extractor reading JWT claims", statement: "xiaoguai-api::middleware::require_scope::RequireScope<&'static str> extracts the operator scope from the verified JWT claims and short-circuits with 403 + ErrorEnvelope.code='scope_required' before the handler runs; sprint-13's inline check in routes/hotl_decisions.rs migrates to the extractor.", status: approved, scope: in, decision: "axum extractor over tower middleware; const-generic scope name; JWT claims, not Casbin per-request.", rationale: "Sprint-14 queues at least two more scope-gated routes; inline copies would diverge within a sprint; the extractor surfaces the scope requirement in the handler signature for OpenAPI + cargo doc visibility. JWT claims avoid per-request DB round-trips, accepting the hot-reload trade-off (deferred to sprint-15)." }
    - { id: DEC-LLD-AGENT-007, title: "Sprint-13 carry-forwards bundled into sprint-14 (unknown escalation 404, replay batch pagination)", statement: "POST /v1/hotl/decisions returns 404 on unknown escalation_id and 409 on terminal escalation_id (removing the resumed=false flag); boot replay paginates pending+unexpired rows with a configurable page size (default 256) and starts the HTTP server after page 1 with subsequent pages replaying in background.", status: approved, scope: in, decision: "Bundle two small sprint-13 carry-forwards alongside DEC-LLD-AGENT-006 so they share the same routes/storage diff review.", rationale: "Both touch §4.5/§4.6 surfaces and reuse the existing test fixtures; landing them with sprint-14 avoids a sprint-15 micro-PR series for changes that are individually too small to warrant their own sprint." }
  flows:
    - { id: FLOW-LLD-AGENT-001, title: "ReAct iteration with HotL", statement: "Stream LLM, gate tool calls, parallel dispatch, append history, loop or finalise.", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-AGENT-002, title: "Cancellation propagation", statement: "Registry poll at boundaries; bounded by slowest tool.", status: approved, scope: in, kind: state_transition }
    - { id: FLOW-LLD-AGENT-003, title: "HotL suspend/resume (sprint-12)", statement: "On Suspend verdict emit HotlPending, await DecisionRegistry ticket (or timeout), emit HotlResolved, then dispatch or synthesise failed ToolResult based on operator verdict.", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-AGENT-004, title: "HotL registry boot replay (sprint-13)", statement: "On AppState build, HotlEscalationStore.list_pending_unexpired drives per-row oneshot mint + sleep_until re-arm; HTTP server starts after replay completes.", status: approved, scope: in, kind: state_transition }
    - { id: FLOW-LLD-AGENT-005, title: "RequireScope axum extractor (sprint-14)", statement: "FromRequestParts reads Claims (sprint-11 layer) → claims.scopes.contains(SCOPE) → on miss return 403 with ErrorEnvelope.code=scope_required + scope name; on hit yield Claims to handler.", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-AGENT-006, title: "Paginated HotL registry boot replay (sprint-14)", statement: "list_pending_unexpired_page yields cursor-tagged pages of N rows; run_serve replays page 1 sync (HTTP server then accepts) and spawns a tokio task to drain remaining pages; per-page replay metrics increment incrementally.", status: approved, scope: in, kind: state_transition }
  test_cases: []
relations:
  - { id: REL-LLD-AGENT-001, type: refines, from: DEC-LLD-AGENT-001, to: DEC-HLD-006, status: active }
  - { id: REL-LLD-AGENT-002, type: refines, from: FLOW-LLD-AGENT-001, to: FLOW-CHAT-001, status: active }
  - { id: REL-LLD-AGENT-003, type: refines, from: FLOW-LLD-AGENT-002, to: FLOW-CANCEL-001, status: active }
  - { id: REL-LLD-AGENT-004, type: refines, from: DEC-LLD-AGENT-005, to: DEC-HLD-013, status: active }
  - { id: REL-LLD-AGENT-005, type: refines, from: DEC-LLD-AGENT-005, to: DEC-HLD-014, status: active }
  - { id: REL-LLD-AGENT-006, type: refines, from: DEC-LLD-AGENT-005, to: DEC-HLD-015, status: active }
  - { id: REL-LLD-AGENT-007, type: refines, from: DEC-LLD-AGENT-005, to: DEC-HLD-016, status: active }
  - { id: REL-LLD-AGENT-008, type: refines, from: FLOW-LLD-AGENT-004, to: DEC-HLD-013, status: active }
  - { id: REL-LLD-AGENT-009, type: refines, from: DEC-LLD-AGENT-006, to: DEC-HLD-018, status: active }
  - { id: REL-LLD-AGENT-010, type: refines, from: DEC-LLD-AGENT-006, to: DEC-HLD-016, status: active }
  - { id: REL-LLD-AGENT-011, type: refines, from: FLOW-LLD-AGENT-005, to: DEC-HLD-018, status: active }
  - { id: REL-LLD-AGENT-012, type: refines, from: DEC-LLD-AGENT-007, to: DEC-HLD-013, status: active }
  - { id: REL-LLD-AGENT-013, type: refines, from: FLOW-LLD-AGENT-006, to: DEC-HLD-013, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
