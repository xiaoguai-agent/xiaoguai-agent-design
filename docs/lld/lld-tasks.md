# LLD — `xiaoguai-tasks`

| | |
|---|---|
| Document ID | `LLD-TASKS-001` |
| Type | Low-Level Design |
| Status | `draft` |
| Refines | `DEC-HLD-014` (agent-authored skills) + ADR-0019 (durable task board) |
| Verifies | `REQ-TASK-001`, `REQ-SKILL-AUTHOR-001` |
| Updated | 2026-05-29 |

## 1. Module purpose

Two distinct sub-modules under one crate:

1. **Kanban task board** (ADR-0019) — a durable work-queue with TRIAGE / TO-DO / READY / RUNNING / BLOCKED / DONE columns and auto-dispatcher for sub-agents. Predates this LLD revision; documented at the ADR.
2. **`skill_author`** (PR #72, T3) — agent-authored skill-pack manifest lifecycle (propose → HotL gate → admin approve/reject → materialise to disk). This LLD focuses on the new sub-module.

## 2. Public interface — `skill_author`

```rust
// Manifest the agent proposes
pub struct SkillManifest {
    pub name: String,
    pub description: String,
    pub version: String,           // semver: "MAJOR.MINOR.PATCH"
    pub system_prompt: String,
    pub tool_allowlist: Vec<String>,  // existing tool names only — no new MCP servers
    pub proposed_by: String,       // agent identity
    pub reason: String,            // why the agent wants this skill
}

// Lifecycle row in Postgres
pub struct SkillProposal {
    pub id: Uuid,
    pub tenant_id: String,
    pub manifest: SkillManifest,
    pub status: ProposalStatus,    // Pending | Approved | Rejected | Installed
    pub created_at: DateTime<Utc>,
    pub decided_at: Option<DateTime<Utc>>,
    pub decided_by: Option<String>,
    pub reason: Option<String>,    // rejection reason
}

pub enum ProposalStatus { Pending, Approved, Rejected, Installed }

#[async_trait]
pub trait SkillProposalRepository: Send + Sync {
    async fn create(&self, p: &SkillProposal) -> Result<()>;
    async fn list(&self, tenant_id: &str, status: Option<ProposalStatus>) -> Result<Vec<SkillProposal>>;
    async fn get(&self, id: Uuid) -> Result<Option<SkillProposal>>;
    async fn update_status(&self, id: Uuid, status: ProposalStatus, decided_by: &str, reason: Option<&str>) -> Result<()>;
}

#[async_trait]
pub trait TenantSettingsReader: Send + Sync {
    async fn allow_skill_authoring(&self, tenant_id: &str) -> Result<bool>;
}

#[async_trait]
pub trait SkillAuthorGate: Send + Sync {
    async fn check(&self, tenant_id: Uuid, scope: &str) -> Result<(), GateDeny>;
}

// Public functions — the three lifecycle stages
pub async fn propose(ctx: &Ctx, tenant_id: &str, proposed_by: &str, manifest: SkillManifest) -> Result<SkillProposal, SkillAuthorError>;
pub async fn approve(ctx: &Ctx, id: Uuid, decided_by: &str) -> Result<SkillProposal, SkillAuthorError>;
pub async fn reject(ctx: &Ctx, id: Uuid, decided_by: &str, reason: &str) -> Result<SkillProposal, SkillAuthorError>;

pub enum SkillAuthorError {
    Disabled,                                    // tenant_settings.allow_skill_authoring = false
    InvalidManifest(String),                     // validator failed
    InvalidTenant(String),                       // tenant_id doesn't parse as Uuid for HotL
    GateDenied { reason: String },               // HotL gate Deny verdict
    BudgetExceeded,                              // HotL daily budget hit
    NotFound,
    AlreadyDecided,
    StorageError(String),
}
```

## 3. Module structure

```
crates/xiaoguai-tasks/
├── src/
│   ├── lib.rs
│   ├── card.rs               # Kanban task card (ADR-0019)
│   ├── dispatcher.rs         # Auto-dispatch loop (ADR-0019)
│   ├── executor.rs           # Card → sub-agent execution (ADR-0019)
│   ├── mem.rs                # In-memory repos for testing
│   ├── pg.rs                 # Postgres repos
│   ├── store.rs              # Repo traits
│   ├── traits.rs             # Shared traits
│   ├── types.rs              # Card / Status / etc.
│   ├── metrics.rs            # Prometheus
│   └── skill_author.rs       # ★ T3 — manifest validator + lifecycle
└── tests/
    └── skill_author_e2e.rs   # ★ Mock backend → propose → deny → propose → allow → approve → YAML on disk
```

## 4. Key flows — `skill_author`

### 4.1 Validator (whitelist-only manifest)

Runs before HotL gate and before any audit emission. Per DEC-014, the manifest must be a strict subset of what the marketplace allows:

1. `name` matches `^[a-z][a-z0-9-]{1,63}$`; not in reserved list `{xiaoguai, system, internal}`.
2. `version` matches semver `^[0-9]+\.[0-9]+\.[0-9]+$`.
3. `tool_allowlist` is non-empty AND every entry is a known tool name (looked up via `KnownTools` injected dep). `propose_skill` itself is NOT permitted in the list (no recursion).
4. `system_prompt` is non-empty and ≤ 8 KB.
5. No fields referencing MCP servers, transports, native code, or filesystem paths.

A validation failure returns `SkillAuthorError::InvalidManifest(reason)` — gate is NOT consulted, no audit row is emitted (validation precedes gate, per cost-control and least-surprise).

### 4.2 `propose` flow

```
propose(ctx, tenant, proposed_by, manifest)
   ├─ 1. tenant_settings.allow_skill_authoring(tenant)?
   │       false → return Disabled  (no audit emission — feature is invisible per DEC-014)
   ├─ 2. validate(manifest)
   │       fail → return InvalidManifest(reason)  (no audit, no gate)
   ├─ 3. audit.append("skill.propose", manifest_summary)
   ├─ 4. tenant_uuid = Uuid::parse_str(tenant)?
   │       fail → return InvalidTenant   (fail-closed — don't bypass gate)
   ├─ 5. gate.check(tenant_uuid, "skill_author")
   │       Deny(reason) → audit("skill.hotl_gate", verdict=deny) → return GateDenied
   │       BudgetExceeded → audit("skill.hotl_gate", verdict=deny, reason=budget) → return BudgetExceeded
   ├─ 6. audit.append("skill.hotl_gate", verdict=allow)
   ├─ 7. repo.create(SkillProposal { status=Pending })
   └─ return Pending proposal
```

### 4.3 `approve` / `reject`

```
approve(ctx, id, decided_by)
   ├─ repo.get(id)?
   │     None → NotFound
   │     status ≠ Pending → AlreadyDecided
   ├─ repo.update_status(id, Approved, decided_by, None)
   ├─ write manifest YAML to skills_dir/<name>-<version>.yaml
   ├─ repo.update_status(id, Installed, decided_by, None)  -- two-phase so partial failure is recoverable
   ├─ audit.append("skill.approve", { id, decided_by })
   └─ return Installed proposal

reject(ctx, id, decided_by, reason)
   ├─ similar, but writes status=Rejected and emits "skill.reject"
```

### 4.4 Audit chain invariant

Every successful lifecycle produces **exactly three audit rows**: `skill.propose` → `skill.hotl_gate` → `skill.approve` (or `skill.reject`). A denied proposal produces two: `skill.propose` → `skill.hotl_gate`. A disabled-tenant proposal produces zero (per DEC-014, the feature is invisible).

Test `e2e_full_lifecycle_allow_path` asserts the exact sequence `["skill.propose", "skill.hotl_gate", "skill.approve"]`.

## 5. Error handling

See `SkillAuthorError` enum in §2. All variants are non-recoverable from the agent's perspective (returned as MCP `ToolResult { is_error: true }` so the LLM observes the failure and can adapt).

## 6. Concurrency / transactions

- `repo.create` and `repo.update_status` are individual SQL transactions; the lifecycle is **not** wrapped in a single transaction — `propose` and `approve` can be days apart.
- `skill_author.rs::approve` does two updates: `Pending → Approved` then YAML-write then `Approved → Installed`. If the process crashes between, the row stays `Approved` and a follow-up admin call can re-trigger materialisation.
- Audit emission is fire-and-forget at the `xiaoguai-audit` boundary — chain integrity is preserved by the audit module itself.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit (`skill_author.rs::tests`) | Validator boundary cases (semver parse, reserved names, tool-allowlist whitelist, no-recursion, content-size); audit sequence for allow path / deny path / disabled tenant; repository in-memory get/list/update; `propose_disabled_returns_disabled` (no audit emission). 19 tests. |
| Integration (`tests/skill_author_e2e.rs`) | 2 E2E tests: full happy lifecycle (propose → gate allow → admin approve → YAML on disk → round-trips to identical manifest); revise-after-deny (propose → gate deny → agent revises tool_allowlist → re-propose → gate allow → approve). |
| System | HTTP routes covered in `xiaoguai-api/tests/skill_proposals.rs` (8 tests). CLI commands covered in `xiaoguai-cli` smoke tests. |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-TASKS-001
  type: LLD
  title: xiaoguai-tasks LLD (skill_author sub-module)
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-29
  refines: [DEC-HLD-014, ADR-0019]
  verifies: [REQ-TASK-001, REQ-SKILL-AUTHOR-001]
```
<!-- TRACEABILITY-METADATA:END -->
