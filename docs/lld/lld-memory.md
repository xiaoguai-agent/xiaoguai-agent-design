# LLD — `xiaoguai-memory`

| | |
|---|---|
| Document ID | `LLD-MEM-001` |
| Refines | `DEC-HLD-003` (boot-time embedder selection) |
| Verifies | `REQ-MEM-001`…`REQ-MEM-004`, `MR-MEM-001`, `BEH-MEM-001` |

## 1. Module purpose

Long-term episodic memory. Wraps `MemoryRepo` (storage) and an embedder, exposes:

- `put(memory)` — embed and persist.
- `recall(query, k, filter)` — embed query, ANN against pgvector HNSW.
- `similar(memory_id, k)` — neighbours of an existing row.
- `prune(now)` — delete rows past `expires_at` (manual; no auto-job in v1.4).

## 2. Public interface

```rust
#[async_trait]
pub trait Embedder: Send + Sync {
    fn dimensions(&self) -> usize;
    async fn embed(&self, text: &str) -> Result<Vec<f32>, EmbedError>;
}

pub struct OllamaEmbedder { /* http client, model="all-minilm" */ }
pub struct InMemoryEmbedder; // deterministic hash for smoke tests

pub struct MemoryService {
    storage: Storage,
    embedder: Arc<dyn Embedder>,
}

impl MemoryService {
    pub async fn select_embedder(cfg: &MemoryConfig) -> Arc<dyn Embedder>;
    pub async fn put(&self, tenant: TenantId, m: NewMemory) -> Result<Memory, MemoryError>;
    pub async fn recall(&self, tenant: TenantId, q: RecallRequest) -> Result<RecallResponse, MemoryError>;
}
```

## 3. Module structure

```
crates/xiaoguai-memory/
├── src/
│   ├── lib.rs
│   ├── embed/
│   │   ├── mod.rs           # Embedder trait
│   │   ├── ollama.rs
│   │   └── in_memory.rs     # 384-dim hash
│   ├── service.rs           # MemoryService
│   ├── recall.rs            # query shaping, persona/tags filter
│   ├── error.rs
│   └── config.rs
└── tests/
    ├── ollama_recall.rs     # requires running Ollama
    └── in_memory_smoke.rs
```

## 4. Key flows

### 4.1 Boot-time embedder selection

```
MemoryService::select_embedder(cfg):
   if cfg.ollama_host.is_some_and(non_blank):
       Arc::new(OllamaEmbedder::new(cfg.ollama_host))
   else:
       Arc::new(InMemoryEmbedder)
```

Boot log emits exactly one of:

```
memory: selected embedding backend choice=Ollama("http://localhost:11434")
memory: selected embedding backend choice=InMemory
```

### 4.2 Put

```
put(tenant, NewMemory{ content, tags, persona_id, expires_at })
   ├─ embedding = embedder.embed(content).await?
   ├─ assert embedding.len() == column dimensions (default 384)
   ├─ storage.with_tenant(tenant, |tx|
   │     MemoryRepo::insert(tx, row)
   │   )
   └─ audit memory.create
```

### 4.3 Recall

```
recall(tenant, RecallRequest{ query, limit, tags_filter, persona_id })
   ├─ q_embedding = embedder.embed(query).await?
   ├─ storage.with_tenant(tenant, |tx|
   │     MemoryRepo::recall(tx, q_embedding, limit, tags_filter, persona_id)
   │   )
   ├─ trace recall to recall_traces (sampled)
   └─ return items with scores
```

## 5. Error handling

```rust
pub enum MemoryError {
    Embed(EmbedError),
    Dimension { expected: usize, got: usize },
    Storage(StorageError),
    NotFound,
}

pub enum EmbedError {
    Http(reqwest::Error),
    ModelNotFound(String),
    Dimension { expected: usize, got: usize },
}
```

`ModelNotFound` is mapped to a 503 with the operator hint: `ollama pull all-minilm`.

## 6. Concurrency / transactions

- Embedder calls are concurrent (tokio); rate limited at the HTTP client level per-Ollama-endpoint.
- Put and recall use the storage repo's tenant-scoped transaction.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | Embedder dimensions match column, in-memory determinism, recall request shaping |
| Integration | `ollama_recall.rs` (gated) — real Ollama + pgvector; `in_memory_smoke.rs` — boot, put, recall, returns ≥ 1 row |
| System | Test strategy L1 + L2; MR-MEM-001 air-gap smoke |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-MEM-001
  type: LLD
  title: xiaoguai-memory LLD
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
    - { id: DEC-LLD-MEM-001, title: "Boot-time embedder selection (no runtime failover)", statement: "Embedder chosen at boot from OLLAMA_HOST.", status: approved, scope: in, decision: "No runtime swap.", rationale: "Avoids configuration drift; simple operator mental model." }
    - { id: DEC-LLD-MEM-002, title: "Dimension asserted at put", statement: "Mismatched embedding dimension fails fast at insert time.", status: approved, scope: in, decision: "Fail-fast.", rationale: "Prevents silent corruption when schema is altered without code change." }
  flows:
    - { id: FLOW-LLD-MEM-001, title: "Put and recall", statement: "Embed -> transactional insert; embed query -> ANN recall.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-MEM-001, type: refines, from: DEC-LLD-MEM-001, to: DEC-HLD-003, status: active }
  - { id: REL-LLD-MEM-002, type: mitigates, from: DEC-LLD-MEM-002, to: RISK-OPS-001, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
