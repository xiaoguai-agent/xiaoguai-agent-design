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
│   │   └── round_robin.rs
│   ├── aggregator.rs       # final-message synthesis
│   ├── config.rs
│   └── error.rs
└── tests/
    ├── pattern_round_robin.rs
    └── persona_budget_isolation.rs
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

### 4.3 Cancellation

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
| Unit | Pattern state machines; plan parsing |
| Integration | `pattern_round_robin.rs` — N peers converge or hit max_rounds; `persona_budget_isolation.rs` — exhausting one persona's budget does not affect another's |
| System | Test strategy L4 capability: orchestrated debate produces a final consensus or escalates |

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
