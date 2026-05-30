# LLD — `xiaoguai-orchestrator`

| | |
|---|---|
| Document ID | `LLD-ORCH-001` |
| Refines | `DEC-HLD-011` (orchestrator is multi-agent dispatch), `FLOW-MULTI-001` |
| Verifies | `REQ-CHAT-004`, generalised to peer agents |

## 1. Module purpose

Coordinates multiple `xiaoguai-agent::AgentLoop` peers cooperating on a single task. Each peer is parameterised by a persona from `xiaoguai-personas`. The orchestrator is NOT a BPMN workflow engine; it is a structured-conversation dispatcher.

Supported patterns at v1.4:

- **Plan-then-execute**: planner persona produces a JSON plan; executor persona consumes it step by step.
- **Researcher + critic**: researcher persona answers; critic persona validates; up to N rounds.
- **Round-robin debate**: list of personas converse turn by turn until a stop condition.

v1.6+ (DEC-021, sprint-9) adds the formal **planner-worker-critic triangle** as a fourth pattern. The first three remain — they cover lighter use cases; the triangle is the canonical pattern for non-trivial heterogeneous workflows. The triangle's three roles are:

| Role | Responsibility | LLM call shape |
|---|---|---|
| **Planner** | Decompose a goal into a JSON `Plan` of sub-tasks; assign each to one Worker | One LLM call per re-plan; small context (goal + tool list + persona) |
| **Worker** | Execute one sub-task as a full ReAct loop; report `WorkerResult { artefact, citations, confidence, cost }` to a private `Scratchpad` | N LLM calls per sub-task (the ReAct loop) |
| **Critic** | Review every Worker result; either Approve (promote to session memory), RequestRevision (Worker re-runs with feedback), or Reject (hand back to Planner) | One LLM call per Worker result; small context (result + acceptance criteria) |

## 2. Public interface

```rust
pub struct Orchestrator {
    storage: Storage,
    personas: PersonaRepo<'static>,
    spawn_agent: Arc<dyn AgentSpawner>,
    config: OrchestratorConfig,
}

#[async_trait]
pub trait AgentSpawner: Send + Sync {
    async fn spawn(&self, persona_id: PersonaId, session_id: SessionId)
        -> Result<Arc<AgentLoop>, OrchError>;
}

pub enum OrchestrationPattern {
    PlanThenExecute { planner: PersonaId, executor: PersonaId },
    ResearchCritic  { researcher: PersonaId, critic: PersonaId, rounds: u32 },
    RoundRobin      { peers: Vec<PersonaId>, max_rounds: u32 },
    /// v1.6+ (DEC-021): planner/worker/critic triangle. The three
    /// `PersonaId`s name the configured personas; the optional `budget`
    /// overrides the default 50/40/10 split (Worker/Planner/Critic) of
    /// the parent budget.
    Triangle {
        planner: PersonaId,
        worker: PersonaId,
        critic: PersonaId,
        budget: Option<TriangleBudget>,
        max_replans: u32,
    },
}

/// Per-role budget split for `OrchestrationPattern::Triangle`. Percentages
/// must sum to 100; the orchestrator rejects bad values at config load.
/// Defaults to `{ worker: 50, planner: 40, critic: 10 }` (DEC-021).
pub struct TriangleBudget {
    pub worker_pct: u32,
    pub planner_pct: u32,
    pub critic_pct: u32,
}

impl Orchestrator {
    pub fn run(&self, pattern: OrchestrationPattern, req: AgentRequest)
        -> impl Stream<Item = OrchEvent>;
}

pub enum OrchEvent {
    PeerSpawned { persona: PersonaId },
    PeerEvent { persona: PersonaId, event: AgentEvent },
    RoundCompleted { round: u32 },
    Final { stop_reason: OrchStopReason, summary: String },
}
```

## 3. Module structure

```
crates/xiaoguai-orchestrator/
├── src/
│   ├── lib.rs
│   ├── patterns/
│   │   ├── plan_then_execute.rs
│   │   ├── research_critic.rs
│   │   ├── round_robin.rs
│   │   └── triangle.rs          # v1.6+ (DEC-021) — planner/worker/critic
│   ├── triangle/                # v1.6+ supporting modules
│   │   ├── plan.rs              # Plan, Task, TaskId, acceptance criteria
│   │   ├── roles.rs             # Role enum + PlannerAgent / WorkerAgent / CriticAgent wrappers
│   │   ├── verdict.rs           # Verdict { Approve(reason), RequestRevision(feedback), Reject(reason) }
│   │   ├── scratchpad.rs        # Scratchpad — per-task ephemeral writes, task_id-quarantined
│   │   ├── memory_view.rs       # MemoryView trait + MemorySnapshot
│   │   └── budget.rs            # TriangleBudget + BudgetEnforcer (50/40/10 default)
│   ├── aggregator.rs            # final-message synthesis
│   ├── config.rs
│   └── error.rs
└── tests/
    ├── pattern_round_robin.rs
    ├── persona_budget_isolation.rs
    ├── triangle_happy_path.rs                # v1.6+
    ├── triangle_critic_request_revision.rs   # v1.6+
    ├── triangle_critic_reject_triggers_replan.rs # v1.6+
    ├── triangle_scratchpad_quarantine.rs     # v1.6+ — cross-Worker contamination test
    ├── triangle_budget_split_enforced.rs     # v1.6+
    └── triangle_replan_cap_terminates.rs     # v1.6+
```

## 4. Key flows

### 4.1 Plan-then-execute

```
run(PlanThenExecute{planner, executor}, req)
   ├─ planner_agent = spawn(planner, session_id)
   ├─ planner_stream = planner_agent.stream(req)
   ├─ collect planner Final -> parse JSON plan
   ├─ executor_agent = spawn(executor, session_id)
   ├─ for step in plan:
   │     executor_stream = executor_agent.stream(step)
   │     forward PeerEvent
   ├─ aggregate final results
   └─ emit Final
```

### 4.2 Per-persona HotL isolation

Each peer's tool calls flow through HotL with `scope = "tool_call.<name>.persona.<persona_id>"` so personas with different trust levels can be budgeted separately. Default mode (single budget) is the no-suffix form.

**Suspend interaction (sprint-12, S11-3a.2)** — when `SuspendingHotlGate` is active (see [`lld-agent.md`](lld-agent.md) §4.5), each suspended waiter is keyed by a freshly minted `request_id`, so cross-persona collision at the `DecisionRegistry` is impossible at the routing layer. The persona suffix determines **which policy fires** (and therefore whether the verdict is `Allow`/`Deny`/`Suspend`); it does not need to flow into the registry key. Operator UX (chat-ui `<HotlBanner>`, §4.3 of `lld-chat-ui.md`) shows the scope including the persona suffix so the reviewer knows which peer requested the tool call.

### 4.4 Planner-Worker-Critic triangle (v1.6+, DEC-021)

The triangle is a structured loop: Planner emits a `Plan` of `Task`s, each `Task` runs through Worker → Critic. The Critic's verdict drives the next step.

```
run(Triangle{planner, worker, critic, budget, max_replans}, req)
   ├─ plan_round = 0
   ├─ loop:
   │     ├─ planner_agent = spawn(planner)
   │     ├─ plan = planner_agent.plan(req.goal, session_memory.snapshot())   // 1 LLM call
   │     ├─ for task in plan.tasks:
   │     │     ├─ revision = 0
   │     │     ├─ loop:
   │     │     │     ├─ worker_agent = spawn(worker)
   │     │     │     ├─ scratchpad = Scratchpad::new(task.id)
   │     │     │     ├─ result = worker_agent.execute(task, scratchpad)      // N LLM calls (ReAct)
   │     │     │     ├─ critic_agent = spawn(critic)
   │     │     │     ├─ verdict = critic_agent.review(result, task.acceptance_criteria) // 1 LLM call
   │     │     │     ├─ match verdict:
   │     │     │     │     Approve(reason) → promote to session_memory; break
   │     │     │     │     RequestRevision(feedback) if revision < cfg.max_revisions
   │     │     │     │                       → revision += 1; loop back to Worker with feedback
   │     │     │     │     Reject(reason)   → hand back to Planner; break outer task loop
   │     │     ├─ if any Reject and plan_round < max_replans:
   │     │     │     plan_round += 1; continue outer planner loop
   │     │     ├─ else: emit Final
```

Emits OrchEvent variants `PlanProduced(plan)`, `TaskStarted(task_id)`, `WorkerScratchpadUpdate(task_id, partial)`, `CriticVerdict(task_id, verdict, reason)`, `Replan(reason)`, `Final`.

### 4.5 Shared memory view + private scratchpads (DEC-021)

Three roles must read the **same per-session memory snapshot** to stay consistent, but Workers must NOT see each other's intermediate state. The new `MemoryView` trait captures both invariants:

```rust
#[async_trait]
pub trait MemoryView: Send + Sync {
    /// Read-only snapshot of the session's promoted memory.
    /// All three roles see the same view for the lifetime of one
    /// plan→execute round; the snapshot is taken at plan time and
    /// frozen until the next replan.
    async fn snapshot(&self) -> MemorySnapshot;
}

pub struct MemorySnapshot {
    pub facts: Vec<MemoryFact>,
    pub captured_at: DateTime<Utc>,
    pub round: u32,
}

/// Per-task ephemeral writes. The Worker writes here as it iterates;
/// the Critic reads BUT cannot write. Other Workers cannot see this
/// scratchpad — quarantined per `task_id`. On Critic Approve, the
/// scratchpad contents migrate into session memory (and become visible
/// to the next round's snapshot).
pub struct Scratchpad {
    pub task_id: TaskId,
    pub entries: Vec<ScratchEntry>,    // append-only
    pub cost_tokens: u64,               // tracked for budget enforcement
}
```

The quarantine matters because cross-Worker context contamination is a known multi-agent failure mode — Worker A succeeds with one approach; Worker B sees A's intermediate notes and adopts them as ground truth before A's Critic has approved them. The `Scratchpad` is `task_id`-keyed so even concurrent Workers cannot read each other's drafts.

### 4.6 Budget split enforcement (DEC-021)

The supervisor splits the parent budget at the top of `Orchestrator::run` for a Triangle pattern:

```rust
let parent_budget = req.budget;                       // tokens
let TriangleBudget { worker_pct, planner_pct, critic_pct } =
    pattern.budget.unwrap_or(TriangleBudget::DEFAULT); // 50/40/10
let worker_cap  = parent_budget * worker_pct  / 100;
let planner_cap = parent_budget * planner_pct / 100;
let critic_cap  = parent_budget * critic_pct  / 100;
```

Each role's spawned `AgentLoop` is wrapped with a `BudgetEnforcer` that aborts when the cap is reached and returns an `OrchEvent::BudgetExhausted(role)`. The Critic's small share is intentional — it makes accept/reject decisions, not artefacts; if the Critic budget is exhausted, the loop ends with `Final { stop_reason: BudgetExhausted }`, not with the Critic silently going over.

### 4.7 Failure modes

| Failure | Detection | Recovery |
|---|---|---|
| Planner emits malformed JSON | Plan parser returns `Err` | Retry once with the error injected as context; on second failure, emit `Final { stop_reason: PlannerFailed }` |
| Worker ReAct hits `MaxIterations` | Existing ReAct stop reason | Surface as a `WorkerResult { artefact: None, error: MaxIterations }`; Critic sees it and typically Rejects → Planner re-plans |
| Worker timeouts (slowest-tool deadline) | Existing per-iteration deadline | Same as MaxIterations |
| Critic in revision loop (more than `cfg.max_revisions`) | Counter in §4.4 | After max revisions, emit `OrchEvent::CriticVerdict(task, Reject(too_many_revisions))`; Planner re-plans |
| `max_replans` exhausted | Counter | Emit `Final { stop_reason: MaxReplansReached }`; surface every Critic Reject reason in the final summary |
| Parent cancellation | `CancellationRegistry` poll between roles | All in-flight agents cancelled; emit `Final { stop_reason: Cancelled }` |

The orchestrator listens on the same `CancellationRegistry` token as its session; cancelling the session cancels every peer.

## 5. Error handling

```rust
pub enum OrchError {
    PersonaNotFound(PersonaId),
    PatternMalformed(String),
    PeerAgent(AgentError),
    PlanParseFailed(serde_json::Error),
    MaxRoundsReached,
}
```

## 6. Concurrency / transactions

- Peers spawn sequentially in plan-then-execute; in round-robin they run sequentially by design (the next persona reads the previous outputs).
- Researcher + critic can be parallelised; v1.4 implementation runs them sequentially for determinism.
- Final aggregation is one transaction per orchestration run.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | Pattern state machines; plan parsing; `TriangleBudget` percentages-sum-to-100 validation; `Verdict` round-trip; `Scratchpad` append-only semantics (v1.6+); `MemorySnapshot` round-frozen contract (v1.6+) |
| Integration | `pattern_round_robin.rs` — N peers converge or hit max_rounds. `persona_budget_isolation.rs` — exhausting one persona's budget does not affect another's. **Triangle (v1.6+)**: `triangle_happy_path` — Planner → 2 Workers → Critic Approve → Final. `triangle_critic_request_revision` — Critic asks for revision; Worker re-runs once with feedback; Approve. `triangle_critic_reject_triggers_replan` — Critic Reject → Planner re-plans with reject reason as context. `triangle_scratchpad_quarantine` — Worker A's scratchpad NOT visible to Worker B even when both run concurrently. `triangle_budget_split_enforced` — Planner exceeds its 40% cap → BudgetExhausted event surfaces. `triangle_replan_cap_terminates` — after `max_replans` Plan→Reject loops, Final with stop_reason=MaxReplansReached. |
| System | Test strategy L4 capability: orchestrated debate produces a final consensus or escalates. v1.6+ adds: triangle pattern produces an Approved artefact for a research-synthesis goal, with the final summary citing both Worker contributions and the Critic's accept reason. |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-ORCH-001
  type: LLD
  title: xiaoguai-orchestrator LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents: [HLD-XIAOGUAI-001, PRD-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-ORCH-001, title: "Structured patterns only", statement: "v1.4 ships three structured patterns; free-form DAGs deferred.", status: approved, scope: in, decision: "Three patterns.", rationale: "Reduces config surface and bug class." }
    - { id: DEC-LLD-ORCH-002, title: "Per-persona HotL scope suffix", statement: "Tool-call scope includes persona_id suffix so personas budget independently.", status: approved, scope: in, decision: "Suffixed scope.", rationale: "Operator can run a low-trust persona on a tight budget without affecting a trusted persona." }
  flows:
    - { id: FLOW-LLD-ORCH-001, title: "Plan-then-execute", statement: "Planner emits plan JSON; executor consumes step-by-step.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-ORCH-001, type: refines, from: FLOW-LLD-ORCH-001, to: FLOW-MULTI-001, status: active }
  - { id: REL-LLD-ORCH-002, type: refines, from: DEC-LLD-ORCH-002, to: DEC-HLD-006, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
