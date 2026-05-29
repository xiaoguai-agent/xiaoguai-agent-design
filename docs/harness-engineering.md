# Harness Engineering Philosophy — Xiaoguai

| | |
|---|---|
| Document ID | `PHILO-XIAOGUAI-001` |
| Type | Architecture philosophy / design rationale |
| Status | `draft` |
| Source | Mitchell Hashimoto / OpenAI "Harness Engineering" (2026); TRAE — "万字干货：理解 Harness Engineering" (2026-04-10); implementation repo |
| Updated | 2026-05-28 (enriched from TRAE long-form) |
| Companion docs | [`hld.md`](hld.md), [`guardrails.md`](guardrails.md), [`lld/lld-agent.md`](lld/lld-agent.md) |

> **Purpose.** Capture the engineering philosophy that has implicitly guided
> Xiaoguai's design across the past four months. The R.E.S.T model and
> REPL-container abstraction below are not new architecture — they are a
> *vocabulary* for what we have already built, which makes future design
> decisions easier to argue from first principles.

> **At a glance.** Five lenses on the same thing. If you only have time
> for one section, read §2 (R.E.S.T) for the product goals or §3 (REPL
> container) for the runtime shape. The other lenses are reframings.
>
> ```
> ┌────────────────────────────────────────────────────────────────┐
> │  R.E.S.T          §2   what we optimise for                    │
> │  REPL container   §3   runtime shape (Read/Eval/Print/Loop)    │
> │  PPAF cycle       §4   the same loop, named for the agent      │
> │  4-D harness      §5   the four engineering disciplines        │
> │  Strategic matrix §6   where on the maturity grid you are      │
> │  Constraints      §7   the three things that bite              │
> │  Principles       §8   six design rules to negotiate them      │
> │  Components       §9   three formal positions: Context-Mgr /   │
> │                        Call-Interceptor / Feedback-Assembler   │
> │  Control / Data   §10  the runtime planes                      │
> │  Token pipeline   §11  Reduction Rules + Injection Boundary    │
> │  Function Calling §12  the 4-step lifecycle that breaks most   │
> │  Planning modes   §13  ReAct / Plan-and-Execute / Multi-agent  │
> │  Sandbox tiers    §14  L1–L4, when to upgrade                  │
> │  Policy gateway   §15  the chokepoint before execution         │
> │  Metrics          §16  four buckets that drive iteration       │
> │  Anti-patterns    §17  what we deliberately don't do           │
> └────────────────────────────────────────────────────────────────┘
> ```

---

## 1. What Harness Engineering is (and isn't)

Harness Engineering ("驾驭工程") is the systematic naming of an existing
practice: **everything besides the LLM itself that makes an agent actually
work**. The metaphor — "horse and reins" — frames the LLM as a powerful but
directionless animal, and the harness as the gear that turns raw capability
into reliable production work.

Three corollaries follow:

1. **Harness Engineering is not a better prompt and not a better model.** It
   is the engineering of the environment around the model.
2. **It is not a new invention.** Prompt Engineering, Context Engineering,
   tool-use frameworks, sandboxing, audit, policy gateways — they were all
   already harness components before the term existed.
3. **Harness components have a lifecycle.** As model capability grows, some
   harnesses are absorbed into the model and retire; new application
   surfaces create new harnesses. *We accept that some of our current
   investment will become legacy as Claude 5 / GPT-6 land.*

---

## 2. R.E.S.T — the four product goals

A useful agent system simultaneously satisfies four properties. Treat them
as **goals**, not features: every Xiaoguai PR should be readable as
contributing to at least one of them.

| Goal | Definition | What this looks like in Xiaoguai today |
|---|---|---|
| **R**eliability | The system continues to do what it claims across expected/unexpected inputs, partial failures, and environment drift. Failure-recovery, idempotency, behavioral consistency. | sd-notify watchdog (`xiaoguai-core.service`); supervisor restart in `xiaoguai-orchestrator::supervisor`; in-process cache fallback (#60) when Valkey absent; ReAct loop bounded by `window` trim (`crates/xiaoguai-agent/src/history.rs`). |
| **E**fficiency | Resource use (tokens, API quota, compute, latency) is bounded and predictable. | HotL budget gate (#61) per tool call; token-aware history trim; multi-provider LLM router (`xiaoguai-llm`) lets the cheapest backend be the default; pgvector instead of a separate vector DB. |
| **S**ecurity | Least privilege, sandboxed execution, IO filtering, auth/audit, defense in depth. | env-scrubbed mcp-exec sandbox (#64); HMAC audit chain (`xiaoguai-audit`); Casbin RBAC + Postgres RLS; OIDC; `_sanitize()` on MCP tool output; PII redaction in audit + OTel (#57). |
| **T**raceability | Every decision and side-effect is auditable; full-chain tracing; reproducible state at any past point. | OTel spans on every agent step; outcome chain (`xiaoguai outcomes`); audit log with chained HMAC so tamper is detectable; per-tool span events for tool selection + dispatch. |

**Negotiation rule.** When two goals conflict, name the trade explicitly in
an ADR. The default ordering in Xiaoguai is **S ≥ T ≥ R ≥ E** — we sacrifice
efficiency before reliability, reliability before traceability,
traceability before security. The PII-redaction-before-HMAC ordering in
audit (#57) is one example of T being subordinate to S (we redact, then
chain — making the chain over redacted data is acceptable because operator
debuggability is replaced by structured trace metadata).

---

## 3. The harness as a REPL container

Conceptually, the harness wraps the (non-deterministic) LLM inside a
deterministic Read-Eval-Print-Loop with boundary control:

```
   ┌─────────────────────  Harness (REPL container)  ──────────────────────┐
   │                                                                       │
   │   READ   ── Context Manager ────────────┐                              │
   │             • user input                │                              │
   │             • short-term memory         │                              │
   │             • long-term memory (RAG)    │                              │
   │             • tool schemas              │                              │
   │             • policy preamble           │                              │
   │                                         ▼                              │
   │                                  ┌────────────┐                        │
   │                                  │    LLM     │  ← non-deterministic   │
   │                                  └────────────┘                        │
   │                                         │                              │
   │   EVAL   ── Call Interceptor ──────────┤                              │
   │             • parse tool calls          │                              │
   │             • HotL gate verdict         │                              │
   │             • sandbox / RBAC / quota    │                              │
   │             • dispatch + capture        │                              │
   │                                         ▼                              │
   │   PRINT  ── Feedback Assembler ─────────┤                              │
   │             • structured ToolResult     │                              │
   │             • outcome event             │                              │
   │             • audit row + OTel span     │                              │
   │                                         │                              │
   │   LOOP   ── until terminal state ───────┘                              │
   │                                                                       │
   └───────────────────────────────────────────────────────────────────────┘
```

**Mapping to Xiaoguai modules:**

| REPL phase | Module |
|---|---|
| READ — Context Manager | `xiaoguai-agent::history`, `xiaoguai-rag`, `xiaoguai-memory` (with [[memory-bridge]]: Ollama embedder by default) |
| READ — Policy preamble | `xiaoguai-personas`, `xiaoguai-api::hotl` |
| EVAL — Call Interceptor | `xiaoguai-agent::react::dispatch_tools` + `HotlGate` (#61) |
| EVAL — Sandbox | `xiaoguai-mcp-exec` (#64); for non-code tools, MCP client `xiaoguai-mcp` |
| PRINT — Feedback Assembler | `xiaoguai-agent::loop_`, `xiaoguai-audit`, `xiaoguai-observability::redaction` (#57) |
| LOOP — Terminal/abort | `xiaoguai-orchestrator::supervisor` for retries; ReAct-loop `max_steps` for budgets |

### State separation invariant (CRITICAL)

> **The LLM is a stateless CPU. All cross-turn state lives in harness
> storage.**

This is non-negotiable. We never ask the model to "remember" a fact across
turns — we *retrieve* it. Practically:

- Conversation history → `xiaoguai-storage` (Postgres `messages` table)
- Long-term facts → `xiaoguai-memory` (pgvector, with optional Ollama
  embedder)
- Tool definitions → `mcp_servers` + `tools` tables (loaded fresh per
  request)
- HotL budget counters → `hotl_usage_log` (per tenant + tool)
- Outcome ledger → `outcome_event` chain

Anti-pattern (rejected in code review): "tell the model in the system
prompt that the user prefers Celsius and trust it to remember." We store
the preference in `memory.facts` and re-inject it via RAG at the top of
each turn.

---

## 4. PPAF — the agent cycle viewed from outside

REPL is the *harness's* view. **PPAF** is the same loop viewed from the
*agent's* side:

| Phase | Name | What the agent does | Where it touches the harness |
|---|---|---|---|
| **P**erception | Observe | Read user input, tool results, recent state | READ — Context Manager |
| **P**lanning | Think | Decide the next step (tool call, message, or terminate) | EVAL — LLM call |
| **A**ction | Act | Emit a tool call or a user-facing message | EVAL — Call Interceptor |
| **F**eedback / Reflection | Reflect | Incorporate the result into the next Perception | PRINT — Feedback Assembler |

REPL describes *how the runtime is implemented*; PPAF describes *what the
agent experiences*. They name the same four positions; using both makes
it easier to say where a bug lives ("the model has bad Perception" vs
"the Context Manager dropped a field" — the second is actionable).

---

## 5. The four-dimensional harness — 造缰 / 驭马 / 相马 / 育马

The TRAE writeup frames the engineering disciplines as four
horse-handling roles. They map cleanly to Xiaoguai sub-projects:

| Dimension | Literal | Engineering meaning | Xiaoguai realisation |
|---|---|---|---|
| **造缰** | Bridle-making | Define what the agent is allowed to do — tool schemas, policy buckets, RBAC, sandbox boundaries | `mcp_servers` + `tools` tables · `xiaoguai-auth` (Casbin) · `xiaoguai-api::hotl::policy` · `xiaoguai-mcp-exec` env allow-list |
| **驭马** | Driving | Steer the agent at runtime — context assembly, dispatch, mid-flight intervention | `xiaoguai-agent::react` loop · `xiaoguai-agent::dispatch_tools` · `HotlGate` · `xiaoguai-orchestrator::supervisor` |
| **相马** | Horse-judging | Evaluate how the agent is doing — telemetry, outcomes, evals | `xiaoguai-observability` (Prometheus + OTel) · `xiaoguai-eval` regression+capability suites · outcome ledger |
| **育马** | Cultivating | Improve the agent over time — eval-driven model swaps, prompt/skill tuning, feedback loops | Ralph-loop review · Outcome→eval pipeline · `xiaoguai-llm::router` provider swaps · skill-pack marketplace |

If you can name which of the four a piece of work falls under, the rest
of the design — owner, success metric, blast radius — falls out.

---

## 6. The strategic 2D matrix — where on the maturity grid are you?

A way to *position* an agent application, with implications for what
harness investment to prioritise.

```
                  Context Efficiency
                  (sandboxed / auto-injected)
                                 ▲
                                 │
                    ┌────────────┼────────────┐
                    │  Q2        │  Q1        │
                    │  passive   │  proactive │
                    │  + efficient│  + efficient│
                    │            │            │ ← target
                    ├────────────┼────────────┤
                    │  Q3        │  Q4        │
                    │  passive   │  proactive │
                    │  + manual  │  + manual  │
                    │            │            │
                    └────────────┴────────────┘──────►
                                                 Cognitive Loop
                                                 (proactive plan
                                                  & reflect)
```

- **Cognitive Loop axis** — from passive single-shot (ReAct on user
  trigger) to proactive multi-step plan-and-reflect (scheduler-driven
  agents that wake themselves up).
- **Context Efficiency axis** — from manual / per-prompt context
  feeding to sandboxed and auto-injected (file system, API gateway,
  state engine).

Harness maturity is the journey from Q3/Q4 (toy demos) to Q1/Q2
(production). **Xiaoguai today sits in Q1/Q2**: the scheduler +
proactive checker push the cognitive-loop axis right; pgvector RAG +
the supervisor's auto-spawn of MCP servers push the context-efficiency
axis up. New work should be scored on which axis it moves.

---

## 7. Three core constraints

The article names six design principles as responses to **three core
constraints**. Stating the constraints explicitly clarifies *why* each
principle exists:

| # | Constraint | What it means | Principles that respond |
|---|---|---|---|
| 1 | **LLM non-determinism** | The same prompt can produce different output. Tool calls can be malformed. The model can refuse, hallucinate, or loop. | Design-for-Failure · Contract-first · Decision/Execution separation |
| 2 | **Finite linear context window** | Transformers see a bounded token sequence; world state is unbounded. Information at middle positions is under-attended. | Contract-first (structured prompts) · Everything-is-measurable (per-token spend) |
| 3 | **Latency / cost ceiling** | Every LLM call costs time and money; every tool call adds a network or compute round-trip. Budgets are real. | Secure-by-Default (kill on budget) · Everything-is-measurable · Data-driven-Evolution (model swaps) |

These three constraints will not go away — they are properties of the
underlying technology, not bugs to be fixed. Harness Engineering is the
discipline of *negotiating* with them.

---

## 8. Six design principles

| # | Principle | How Xiaoguai applies it |
|---|---|---|
| 1 | **Design for Failure** | Every external call has timeout + retry + circuit-breaker. ReAct loop has `max_steps`. Supervisor restarts crashed workers. We test failure paths (`tests/hotl_gate.rs` deny path, MCP timeout test, compaction backend-error fallback). |
| 2 | **Contract-first** | OpenAPI 3 for HTTP. JSON Schema for tool definitions. SQL migrations are forward-only. Casbin policy is a written contract. PRD/HLD/LLD all checked into `xiaoguai-agent-design/`. |
| 3 | **Secure by default** | Audit signing key required at boot (`smoke` refuses to start without it). PII redaction defaults ON (`XIAOGUAI_AUDIT_REDACT_PII=true` by default). Outbound MCP defaults to TLS-verify-on. systemd unit uses `NoNewPrivileges`, `ProtectSystem=strict`, capability bounding set empty. |
| 4 | **Decision ↔ Execution separation** | The planner (LLM + ReAct) decides *what* tool to call. The executor (MCP client / sandbox) decides *how*. The HotL gate sits between them as the policy boundary. Three-way separation prevents a captured planner from also being the executor. |
| 5 | **Everything is measurable** | Per-tool latency, per-tool success rate, per-tenant token spend, per-policy denial count, per-session compaction savings. Prometheus exporter + Grafana dashboard. Outcome ledger lets us A/B model choices. |
| 6 | **Data-driven evolution** | Eval suite (`xiaoguai-eval`) replays canonical agent transcripts against new models. Outcome events feed retrospective analysis. Ralph-loop review is mandatory before merging anything that touches the agent loop. |

---

## 9. The three formal harness components

The article gives the harness *positions* concrete names. Use these
when writing LLDs so the role is unambiguous:

| Component | English / Chinese | Position | Xiaoguai modules |
|---|---|---|---|
| **Context Manager** | 上下文管理器 | READ. Translates user input + memory + tool schemas + policy preamble into a structured prompt the LLM can consume. Owns the Token Transformation Pipeline (§11). | `xiaoguai-agent::history` (slide + compact) · `xiaoguai-rag` · `xiaoguai-memory` · prompt assembly in `loop_::build_prompt` |
| **Call Interceptor** | 调用拦截器 | EVAL. Captures the model's tool-call intent, validates it against the contract, gates it (HotL/RBAC/quota), and dispatches to the executor under timeout + resource caps. | `xiaoguai-agent::react::dispatch_tools` · `HotlGate` (#61) · `xiaoguai-mcp::McpClient` · `xiaoguai-mcp-exec` (#64) |
| **Feedback Assembler** | 反馈汇编器 | PRINT. Captures the executor's structured result (success or failure), normalises it, redacts PII, audits it, and re-injects it into the next turn's context as a `Tool` role message. | `xiaoguai-agent::loop_` (tool-result wrapping) · `xiaoguai-observability::redact` (#57) · `xiaoguai-audit::PgAuditSink` · OTel span emission |

Naming the positions makes onboarding cheap: a new contributor can be
told "your change lives in the Call Interceptor" and immediately knows
*which crate* and *which trade-offs* are in play.

---

## 10. Control plane / data plane split

Borrowed from the Hashimoto framing — and matches what we've already
built:

| Plane | What it owns | Xiaoguai modules |
|---|---|---|
| **Control** | Scheduling, quotas, planning, policies, permissions | `xiaoguai-scheduler`, `xiaoguai-orchestrator`, `xiaoguai-api::hotl`, `xiaoguai-auth`, `xiaoguai-watch`, `xiaoguai-anomaly` |
| **Data** | Agent runtime instances, conversation state, memory stores, sandbox execution | `xiaoguai-agent`, `xiaoguai-storage`, `xiaoguai-memory`, `xiaoguai-rag`, `xiaoguai-mcp-exec`, `xiaoguai-runtime` |

A control-plane change (new HotL policy) takes effect on the next
data-plane request — no agent restart, no migration. This is enforced by
`hotl_policies` being read on every dispatch (with a small in-process
cache, TTL 5s).

The TRAE writeup frames the harness as "intelligent glue": it sits
**above the model API gateway** (talking to one or many LLM providers
via `xiaoguai-llm::router`) and **below the sandbox / external services**
(MCP servers, IM gateways, scheduler triggers). The glue is what makes
the LLM API and the messy external world look the same to the agent
loop.

---

## 11. Token transformation pipeline

The LLM operates on a finite, linear token sequence. The harness's job is
to compress the (effectively unbounded) world state down to that window.
Two engineering decisions sit at the heart of this pipeline:

- **Reduction Rules** (规约规则) — the *what gets dropped under Token
  pressure* policy. Tool schemas are usually kept (they're contract);
  ancient history is usually dropped (it's stale); RAG results are
  filtered by similarity threshold; personas are kept verbatim because
  they're short.
- **Injection Boundary** (注入边界) — *where in the assembled prompt*
  retrieved information goes. The standard mistake is to inject RAG
  results in the middle of a long history, where Transformer attention
  under-weights them (the "Lost in the Middle" effect). Xiaoguai puts
  high-leverage information (current task, latest tool result, key
  facts from memory) at the *ends* of the assembled prompt.

Xiaoguai's pipeline:

1. **Source collection** — collect user input, last-N turns, RAG results,
   tool schemas. (Implemented in `xiaoguai-agent::loop_::build_prompt`.)
2. **Relevance ranking** — pgvector cosine for RAG, recency for turns,
   tool-affinity scoring for schemas.
3. **Compression / summarization** — `history::compact` (Plan D.2,
   v0.5.4.1) generates an LLM summary of older turns when the estimate
   crosses the trigger threshold; falls back to `slide` on failure.
4. **Budget allocation** — per category quota (system: small, RAG: medium,
   history: large, tools: bounded by count) under a global model-context cap.
5. **Template assembly** — explicit role-tagged sections so the model can't
   confuse retrieved knowledge with the user's instruction.

Codified in `xiaoguai-agent::loop_::assemble_messages` and
`xiaoguai-agent::history`.

---

## 12. Function Calling lifecycle

Function Calling is the bridge from LLM planning to physical action. The
TRAE writeup breaks it into a strict four-step lifecycle, each step a
distinct failure surface:

```
   ┌─ harness ──────────────────────────────────────────────────┐
   │                                                            │
   │  1. Schema serialisation     ToolSpec → JSON Schema → Prompt
   │              │                                              │
   │              ▼                                              │
   │           ┌────────┐                                        │
   │           │  LLM   │  2. Trigger generation                 │
   │           └────────┘     model emits a tool_call            │
   │              │                                              │
   │              ▼                                              │
   │  3. Deterministic deserialisation                           │
   │     parse name + arguments_json → typed call                │
   │              │                                              │
   │              ▼                                              │
   │  4. Observation injection                                   │
   │     wrap result in Role::Tool, append to messages           │
   │              │                                              │
   │              ▼                                              │
   │            (next turn's READ phase)                         │
   │                                                            │
   └────────────────────────────────────────────────────────────┘
```

**The brittle step is #3** — LLMs emit malformed JSON often enough that
a robust harness MUST plan for it. Failure modes and Xiaoguai's
responses:

| Step | Typical failure | Xiaoguai response |
|---|---|---|
| 1 — Serialisation | Tool description is too long / too vague; model can't tell when to use it | Tool descriptions are explicitly `[READ]` / `[WRITE]`-tagged and answer When/What/Gotchas |
| 2 — Trigger | Model emits no tool call when one is obviously needed (or vice-versa) | Eval suite asserts tool-selection correctness on canonical prompts; metric `tool_use_correct_total` tracks drift |
| 3 — Deserialisation | Invalid JSON / wrong argument type | `dispatch_tools` returns a synthetic failed `ToolResult` so the LLM observes the error and can self-correct in the next turn; we never panic the loop |
| 4 — Observation injection | `Role::Tool` message lacks a matching `tool_call_id` after history trim → provider 400 | `history::slide` (and `compact`) walk back the split boundary past dangling Tool messages — codified invariant |

Re-planning on persistent failure is the supervisor's job (§13).

---

## 13. Planning modes

Xiaoguai today implements:

- **ReAct (single-step)** — default. Cheap, fast, good for "look up X and
  reply".
- **Plan-and-Execute** — when `task.kind = plan` is set; the planner emits
  a JSON plan, the executor runs it. Used for orchestrator workflows.
- **Multi-agent** — `xiaoguai-orchestrator` spawns sub-agents with their
  own ReAct loops; budget enforcement composes (sub-agent budgets ≤ parent
  remaining budget).

Re-planning on failure is supported via the supervisor's "challenger" path
(`xiaoguai-orchestrator::challenger`) — the supervisor can substitute a
fresh planner if the original gets stuck.

**Default to Plan-and-Execute for enterprise workflows; default to ReAct
for chat.** Add multi-agent only when the task is genuinely
parallelisable. The article's advice — *"for most enterprise scenarios
Plan-and-Execute + exception-triggered replanning is sufficient; reach
for multi-layer planning and multi-agent only for open-ended, long-lived
tasks"* — matches our observed production patterns.

---

## 14. Sandbox tiering

| Level | Mechanism | Xiaoguai usage |
|---|---|---|
| L1 (Python) | Process isolation (`ulimit`, env scrub, tempdir) | `xiaoguai-mcp-exec` (#64). Adequate for trusted-tenant internal code. |
| L1 (JavaScript) | Process isolation + Deno `--allow-none` | `xiaoguai-mcp-exec-js` (#75, T6). Separate trust boundary from Python — see DEC-017. |
| L2 | Container isolation (Docker, containerd) | Operator can wrap L1 binaries in a container for additional isolation; not the default. |
| L3 | wasmtime + pyodide (Python) / wasmtime + QuickJS-WASM (JS) | **Decision committed in ADR-0020 (PR #76).** Implementation deferred 2–3 sprints. Sibling crate `xiaoguai-mcp-exec-wasm` will extract an `ExecBackend` trait from the L1 crates and become an opt-in via tenant config. |
| L4 | Full VM (KVM/QEMU) | Not planned. Cost outweighs benefit for the target deployment shape. |

**Trade.** L1 is *fast* (3 ms cold start Python; ~10 ms JS due to Deno
init) and *cheap* (no daemons). L3 (when shipped) is *safer* but adds
~10 ms even with engine cache, and pyodide CPython is 2–4× slower than
native on CPU-heavy work. We default to L1 + a strict tool allow-list
+ per-language HotL bucket (`tool_call.execute_python`,
`tool_call.execute_javascript`); operators move to L3 when running
untrusted tenant code.

**Why L1 ships TWO crates** (per DEC-017): trust-boundary physical
separation. A V8 JIT 0-day and a CPython interpreter 0-day are
independent attack surfaces; sharing the binary would let either CVE
compromise both. Separate crates → separate binaries → separate
process spaces → composable failures stay isolated. Same argument
will apply at L3 — `xiaoguai-mcp-exec-wasm` will host both pyodide
and QuickJS in the same `wasmtime::Engine` (capability-confined per
module) but operators can still register them as two MCP servers if
they want belt-and-braces isolation.

---

## 15. Policy Gateway

A policy gateway sits between the planner and the executor. In Xiaoguai
this is the **HotL gate** (`crates/xiaoguai-agent/src/hotl_gate.rs`) plus
the Casbin enforcer. Per the article's pattern, every action passes
through:

1. **Permission check** — RBAC (tenant/role) + ABAC (resource attributes,
   e.g., tool name, target host).
2. **Sensitive-data filtering** — PII redaction on tool args *before* they
   are persisted to audit, and on tool output *before* it re-enters the
   LLM prompt.
3. **Prompt-injection defense** — `_sanitize()` on all foreign text;
   tool-result text is wrapped in explicit role markers so the model
   cannot interpret it as user instruction.
4. **Audit log** — every gateway decision (Allow / Deny / Escalate) is a
   row in `hotl_usage_log` and `audit_log`; the chain is HMAC-signed.

The Escalate verdict is xiaoguai-specific: it lets the gate ask a human
operator before allowing — used for high-blast-radius actions like
`vm_delete` or `payment.transfer`.

---

## 16. Metrics that drive the harness

We instrument across four buckets matching the article's framing:

| Bucket | Xiaoguai metrics (Prometheus names) |
|---|---|
| Task effectiveness | `xiaoguai_task_success_total`, `xiaoguai_tool_use_correct_total`, `xiaoguai_instruction_followed_total`, `xiaoguai_skill_proposals_total{outcome}` (T3) |
| Service quality | `xiaoguai_request_duration_seconds` (histogram, with first-token-time variant), `xiaoguai_error_total{kind=...}`, `xiaoguai_mcp_oauth_refresh_duration_seconds` (T4) |
| Resource efficiency | `xiaoguai_tokens_consumed_total`, `xiaoguai_tool_calls_per_task`, `xiaoguai_llm_provider_cost_usd`, `xiaoguai_compaction_token_savings` |
| Security & compliance | `xiaoguai_policy_denial_total`, `xiaoguai_security_incident_total`, `xiaoguai_audit_chain_break_total`, `xiaoguai_compaction_fallback_total`, `xiaoguai_audit_export_chain_broken_total` (T5), `xiaoguai_mcp_oauth_refresh_failed_total` (T4) |

These drive iteration: when `task_success_total / task_started_total` drops
on a model upgrade, we either revert or fix the planner. When
`tokens_consumed_total` spikes, we trigger compaction earlier. When
`compaction_fallback_total / compaction_triggered_total > 5 %`, the
summary model is unreliable and we swap it.

---

## 17. What this philosophy rejects

A few patterns we have deliberately not adopted, with reasons:

- **"Prompt the model into being safe."** Soft constraints in the system
  prompt are documentation, not enforcement. The harness enforces.
- **One giant Agent that does everything.** We compose smaller agents with
  bounded scope; the orchestrator coordinates. Easier to test, easier to
  RBAC.
- **Storing state in the prompt.** Discussed in §3 — state lives in the
  harness, not in tokens.
- **Trusting tool output as user input.** Always sanitize, always tag with
  role markers.
- **Letting the LLM author its own escapes.** Tool definitions are
  declarative; agents CAN propose new tools at runtime via
  `propose_skill` (T3, PR #72), but every proposal passes a strict
  three-gate flow: **validator** (whitelist-only — references existing
  tools by name, cannot declare MCP servers or native code) +
  **HotL gate** (`skill_author` bucket, 5/tenant/day default) +
  **admin approval** (Casbin `skill.approve`). The agent gets
  flexibility within a hard-coded escape envelope; the harness
  enforces, not prompts.
- **Bypassable compliance gates.** When the audit chain has a break,
  the export refuses (T5, PR #74) — there is no `--skip-verify` flag
  by design. Auditors get the `ChainProof` as evidence; the answer to
  a broken chain is to investigate, not to ship a tainted report.
- **Wide OAuth scope grants.** Outbound MCP OAuth (T4, PR #73) accepts
  the scope list verbatim from `xiaoguai mcp register` — we do NOT
  expand or canonicalise it. Operators are responsible for narrowing
  scopes per server. This is documented in the runbook as a "least
  scope" practice rather than enforced; future work may add a
  per-tenant scope allow-list.
- **Sharing trust boundaries across languages.** Even when two
  features share infrastructure (Python and JS sandboxes share the
  L1 contract), the binaries stay separate (DEC-017). Cross-runtime
  composition of CVEs is a non-goal.
- **Treating Harness Engineering as a destination.** It's not. As models
  absorb more capability, today's harness components retire and new ones
  arise. We accept this churn explicitly (§1, corollary 3).

---

## 18. How this document is used

- New PRs that touch the agent loop should reference §2 (R.E.S.T) and §8
  (six principles) in their description.
- New LLD docs use the §3 REPL phase taxonomy and the §9 formal-component
  names to locate the new feature.
- Ralph-loop reviewers use §17 as a checklist of anti-patterns.
- When negotiating R.E.S.T trade-offs in an ADR, cite the §2 negotiation
  rule.
- When defending a piece of work to a sceptical reviewer, state which of
  the §5 four dimensions it falls under and which of the §7 three
  constraints it addresses. If neither answer is clear, the work is
  probably premature.

---

## References

- Hashimoto / OpenAI "Harness Engineering" article (input that prompted
  this writeup, 2026-05-28). The "horse and reins" metaphor and the
  control-plane / data-plane split come from there.
- TRAE — *万字干货：理解 Harness Engineering，看这一篇就够了* (2026-04-10).
  Source for the §4 PPAF naming, the §5 four-dimensional 造缰/驭马/相马/育马
  framing, the §6 strategic 2D matrix, the §7 three core constraints,
  the §9 formal-component names (上下文管理器 / 调用拦截器 / 反馈汇编器),
  the §11 Reduction Rules + Injection Boundary terminology, the §12
  four-step Function Calling lifecycle, and the §16 four-bucket metrics
  taxonomy. Local copy at
  `/Users/zw/testany/myskills/xiaoguai-agent-design/理解 Harness Engineering.docx`.
- `hld.md` §1 system context — for the corresponding box diagram.
- `guardrails.md` §3 (security defaults) — concrete enforcement of §8
  principle 3.
- `lld/lld-agent.md` — REPL phase details + Call Interceptor implementation.
- Implementation handoff:
  `/Users/zw/testany/myskills/xiaoguai/docs/HANDOFF-2026-05-28-session5.md`.
- Plan D.2 (compaction) PR: `xiaoguai-agent/xiaoguai#66` — the §11
  token-transformation pipeline's compression/summarisation step.
