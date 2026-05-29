# LLD — `xiaoguai-storage`

| | |
|---|---|
| Document ID | `LLD-STORAGE-001` |
| Type | Low-level design |
| Refines | `DEC-HLD-008` (RLS double-layer) |
| Verifies | `REQ-AUTH-003`, `MR-AUTH-002` |

## 1. Module purpose

`xiaoguai-storage` is the only crate that talks to Postgres. It owns:

- The sqlx connection pool.
- Migrations under `migrations/0001…0020`.
- A repository per domain table (`SessionRepo`, `MessageRepo`, `MemoryRepo`, `HotlPolicyRepo`, …).
- The RLS plumbing helper `Storage::with_tenant(tenant_id, async closure)`.
- The pgvector helper for memory recall.

It does NOT own business logic; repositories are CRUD-only with explicit transaction boundaries.

## 2. Public interface

```rust
pub struct Storage { /* opaque */ }

impl Storage {
    pub async fn connect(cfg: &StorageConfig) -> Result<Self, StorageError>;

    /// Execute a closure inside a tenant-scoped transaction.
    /// Sets `SET LOCAL app.current_tenant_id = $1` so Postgres RLS is enforced.
    pub async fn with_tenant<F, R>(&self, tenant_id: TenantId, f: F) -> Result<R, StorageError>
    where
        F: for<'tx> FnOnce(&'tx mut Tx<'tx>) -> BoxFuture<'tx, Result<R, StorageError>>;

    pub fn sessions(&self) -> SessionRepo<'_>;
    pub fn messages(&self) -> MessageRepo<'_>;
    pub fn memories(&self) -> MemoryRepo<'_>;
    pub fn hotl(&self) -> HotlPolicyRepo<'_>;
    pub fn audit(&self) -> AuditWriter<'_>;
    pub fn llm_providers(&self) -> ProviderRepo<'_>;
    pub fn token_usage(&self) -> TokenUsageRepo<'_>;
    pub fn mcp_servers(&self) -> McpServerRepo<'_>;
    pub fn personas(&self) -> PersonaRepo<'_>;
    pub fn skill_packs(&self) -> SkillPackRepo<'_>;
    pub fn tasks(&self) -> TaskRepo<'_>;
    pub fn outcomes(&self) -> OutcomeRepo<'_>;
    pub fn scheduler(&self) -> SchedulerRepo<'_>;
    pub fn workspaces(&self) -> WorkspaceRepo<'_>;
}
```

## 3. Module structure

```
crates/xiaoguai-storage/
├── src/
│   ├── lib.rs               # Storage struct, with_tenant
│   ├── config.rs            # StorageConfig (url, max_connections, sslmode)
│   ├── error.rs             # StorageError enum
│   ├── tx.rs                # Tx<'a> wrapper
│   ├── repos/
│   │   ├── sessions.rs
│   │   ├── messages.rs
│   │   ├── memories.rs      # pgvector recall queries
│   │   ├── hotl.rs
│   │   ├── llm_providers.rs
│   │   ├── token_usage.rs
│   │   ├── mcp_servers.rs
│   │   ├── personas.rs
│   │   ├── skill_packs.rs
│   │   ├── tasks.rs
│   │   ├── outcomes.rs
│   │   ├── scheduler.rs
│   │   └── workspaces.rs
│   └── audit_writer.rs      # consumed by xiaoguai-audit
├── migrations/
│   ├── 0001_init.sql
│   ├── … (20 total)
│   └── 0020_seed_ollama_default.sql
└── tests/
    ├── rls_isolation.rs     # adversarial cross-tenant tests
    └── recall_hnsw.rs       # pgvector index correctness
```

## 4. Key flows

### 4.1 Tenant-scoped read

```text
with_tenant(tenant_id, |tx| async move {
    tx.execute("SET LOCAL app.current_tenant_id = $1", &[&tenant_id]).await?;
    SessionRepo::list(tx, query).await
})
```

The `SET LOCAL` is per-transaction; the value is consumed by every RLS policy. The app-layer query also adds `WHERE tenant_id = $1` (defence in depth — GR-SEC-03).

### 4.2 Memory recall (pgvector HNSW)

```sql
SELECT id, content, tags, score
FROM (
  SELECT id, content, tags,
         1 - (content_embedding <=> $1) AS score
  FROM memories
  WHERE tenant_id = $2
    AND (expires_at IS NULL OR expires_at > now())
    AND ($3::text[] IS NULL OR tags && $3)
) sub
ORDER BY score DESC
LIMIT $4;
```

`content_embedding` is `vector(384)`; index is HNSW with cosine operator.

### 4.3 Migration apply at boot

`sqlx::migrate!()` runs at boot before any handler is mounted; if a migration fails the process exits non-zero.

## 5. Error handling

```rust
pub enum StorageError {
    Connect(sqlx::Error),
    Migration(sqlx::migrate::MigrateError),
    Query(sqlx::Error),
    NotFound { entity: &'static str, id: String },
    TenantMismatch,
    PgvectorMissing,
}
```

`TenantMismatch` is returned when a query crosses tenant boundaries unexpectedly (defensive check after RLS).

## 6. Concurrency / transactions

- One `PgPool` per `Storage`.
- Repositories take `&mut Tx<'_>` so callers must hold an explicit transaction.
- `audit().append()` MUST be called inside the same transaction as the state change that produced it; otherwise the chain can desync from the change.

## 7. Test design

| Layer | Owner | Cases |
|---|---|---|
| Unit | Crate | Query SQL strings compile (sqlx::query!), RLS helpers set the right `SET LOCAL`, error enum mappings |
| Integration | Crate | `tests/rls_isolation.rs` — tenant A cannot read tenant B's rows; `tests/recall_hnsw.rs` — HNSW index returns plausible top-k |
| System | Test strategy §9 | Adversarial harness across `/v1/**` |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-STORAGE-001
  type: LLD
  title: xiaoguai-storage LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents: [HLD-XIAOGUAI-001, API-XIAOGUAI-001, GUARDRAILS-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-STO-001, title: "Single PgPool per Storage", statement: "One sqlx PgPool owned by Storage; cloning is cheap (Arc).", status: approved, scope: in, decision: "Single pool.", rationale: "Avoid pool fragmentation across crates." }
    - { id: DEC-LLD-STO-002, title: "Tenant-scoped transactions via SET LOCAL", statement: "with_tenant sets SET LOCAL app.current_tenant_id; repositories also WHERE tenant_id=$1.", status: approved, scope: in, decision: "Double-layer.", rationale: "Defence in depth (GR-SEC-03)." }
  flows:
    - { id: FLOW-LLD-STO-001, title: "Tenant-scoped read", statement: "with_tenant sets SET LOCAL then runs query.", status: approved, scope: in, kind: module_interaction }
    - { id: FLOW-LLD-STO-002, title: "Memory recall via pgvector HNSW", statement: "ORDER BY (1 - cosine_distance) DESC LIMIT k.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-STO-001, type: refines, from: DEC-LLD-STO-002, to: DEC-HLD-008, status: active }
  - { id: REL-LLD-STO-002, type: refines, from: FLOW-LLD-STO-002, to: REQ-MEM-001, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
