# LLD — `xiaoguai-personas`

| | |
|---|---|
| Document ID | `LLD-PERSONA-001` |
| Refines | `DEC-HLD-012` (personas are role presets + persona-scoped memory views) |
| Verifies | `DEC-PERSONA-001`, `REQ-MEM-001` (persona filter), partial `REQ-CHAT-001` |

## 1. Module purpose

`xiaoguai-personas` owns the data model and lookup logic for personas (role presets) and the persona-scoped memory view. A persona bundles:

- A persona-specific system prompt template.
- Default toolbox restriction (allow-list of tool names).
- Default skill-pack set.
- A `persona_id` foreign key that other tables (memories, sessions) carry to enable persona-scoped filtering.

Personas are NOT user accounts; they are reusable role templates within a tenant.

**v1.6+ (DEC-021)**: the three roles of the planner/worker/critic triangle (see [`lld-orchestrator.md`](lld-orchestrator.md) §4.4) are **regular personas** — the orchestrator picks any three `PersonaId`s for `OrchestrationPattern::Triangle { planner, worker, critic, … }`. To make role-shaped personas discoverable, operators conventionally name them with a `role/` prefix (e.g., `role/planner-default`, `role/worker-coder`, `role/critic-strict`) and tag them via the optional `tags` column added in migration `0025_persona_role_tags.sql`. The roles are **soft conventions**, not enforced types — a critic-tagged persona run as a Worker in a different invocation still works; the agent loop reads only `system_prompt` and `allowed_tools`. This keeps personas composable while DEC-021's role distinction stays at the orchestrator pattern layer.

## 2. Public interface

```rust
pub struct Persona {
    pub id: PersonaId,
    pub tenant_id: TenantId,
    pub name: String,                       // unique per tenant
    pub system_prompt: String,
    pub allowed_tools: Option<Vec<String>>, // None = all tools
    pub default_skill_packs: Vec<SkillPackId>,
    pub created_at: DateTime<Utc>,
}

pub struct PersonaRepo<'a> { /* &mut Tx<'a> */ }
impl<'a> PersonaRepo<'a> {
    pub async fn list(&mut self, tenant: TenantId) -> Result<Vec<Persona>, StorageError>;
    pub async fn get(&mut self, id: PersonaId) -> Result<Persona, StorageError>;
    pub async fn upsert(&mut self, p: NewPersona) -> Result<Persona, StorageError>;
    pub async fn delete(&mut self, id: PersonaId) -> Result<(), StorageError>;
}
```

## 3. Module structure

```
crates/xiaoguai-personas/
├── src/
│   ├── lib.rs            # types
│   ├── repo.rs           # PersonaRepo
│   ├── prompt.rs         # prompt template helpers
│   └── error.rs
└── tests/
    └── persona_filter.rs
```

The `personas` table (migration 0016) holds the rows; memory `persona_id` column was added in migration 0019.

## 4. Key flows

### 4.1 Build agent request with persona

```
build_agent_request(tenant, persona_id?, user_message)
   ├─ persona = persona_id.map(|id| PersonaRepo::get(id))
   ├─ system_prompt = persona.map(|p| p.system_prompt).or(tenant.default_system_prompt)
   ├─ toolbox = if persona.allowed_tools: full_toolbox.restrict(allowed)
   │            else: full_toolbox
   └─ AgentRequest { system_prompt, toolbox, persona_id, ... }
```

### 4.2 Persona-scoped memory recall

```
recall(tenant, RecallRequest{ query, persona_id: Some(p), ... })
   ├─ MemoryRepo::recall with WHERE persona_id = $p OR persona_id IS NULL
   │     (rows without persona_id are tenant-global, visible to every persona)
   └─ return hits
```

### 4.3 Defaults

If a session is created without a persona, the tenant's `default_persona_id` (column on `tenants`) is used; if that is also null, a built-in `default` persona ("You are a helpful assistant.") is synthesised.

## 5. Error handling

Delegates to `StorageError`. A `Conflict` is returned on duplicate `(tenant_id, name)`.

## 6. Concurrency / transactions

- CRUD via repository transactions.
- `default_skill_packs` are immutable per persona version; renaming creates a new row.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | Persona prompt template rendering; toolbox restriction logic |
| Integration | `persona_filter.rs` — memory recall under persona P returns persona-tagged + global rows, never another persona's rows |
| System | Test strategy L2: chat with persona_id changes the system prompt observably in transcript |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-PERSONA-001
  type: LLD
  title: xiaoguai-personas LLD
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
    - { id: DEC-LLD-PERSONA-001, title: "Persona-scoped memory uses inclusive null", statement: "Recall with persona_id matches rows with that persona OR with NULL persona (tenant-global).", status: approved, scope: in, decision: "Inclusive null.", rationale: "Lets tenants store global preferences shared across personas without duplicating rows." }
    - { id: DEC-LLD-PERSONA-002, title: "Persona name unique per tenant", statement: "Duplicate (tenant_id, name) returns Conflict.", status: approved, scope: in, decision: "Unique constraint.", rationale: "Reduces operator confusion." }
  flows:
    - { id: FLOW-LLD-PERSONA-001, title: "Build agent request with persona", statement: "Resolve persona -> apply system_prompt + toolbox restriction.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-PERSONA-001, type: refines, from: DEC-LLD-PERSONA-001, to: DEC-HLD-012, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
