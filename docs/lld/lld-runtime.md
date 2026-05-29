# LLD — `xiaoguai-runtime`

| | |
|---|---|
| Document ID | `LLD-RUNTIME-001` |
| Refines | `FLOW-CANCEL-001` |
| Verifies | `REQ-CHAT-002`, `MR-CACHE-001` |

## 1. Module purpose

Process-wide async runtime utilities:

- `WorkerPool` for bounded concurrency on background tasks (scheduler ticks, IM webhook fan-out).
- `CancellationRegistry` keyed by `SessionId` for `POST /v1/sessions/{id}/cancel`.
- `Cache` (`DashMap` or `redis://`) used by rate limiter, IM session locks, scheduler nonces.
- `ShutdownSignal` plumbing for graceful drain on SIGTERM.

## 2. Public interface

```rust
pub struct WorkerPool { /* bounded by config */ }
impl WorkerPool {
    pub fn spawn<F: Future<Output = ()> + Send + 'static>(&self, name: &'static str, f: F);
}

pub struct CancellationRegistry { /* DashMap<SessionId, CancellationToken> */ }
impl CancellationRegistry {
    pub fn token(&self, id: SessionId) -> CancellationToken;
    pub fn cancel(&self, id: SessionId);
}

pub struct Cache { /* opaque */ }
impl Cache {
    pub async fn connect(url: &str, prefix: &str) -> Result<Cache, CacheError>;
    pub async fn get(&self, key: &str) -> Result<Option<Bytes>, CacheError>;
    pub async fn set(&self, key: &str, value: &[u8], ttl: Duration) -> Result<(), CacheError>;
    pub async fn incr(&self, key: &str, by: i64, ttl: Duration) -> Result<i64, CacheError>;
}

pub struct ShutdownSignal(broadcast::Receiver<()>);
```

## 3. Module structure

```
crates/xiaoguai-runtime/
├── src/
│   ├── lib.rs
│   ├── worker_pool.rs
│   ├── cancel.rs
│   ├── cache/
│   │   ├── mod.rs            # Cache enum + dispatch
│   │   ├── in_process.rs     # DashMap impl with sub-second TTL
│   │   └── redis.rs          # redis crate impl, TTL clamped to 1s
│   ├── shutdown.rs
│   └── error.rs
└── tests/
    ├── cache_backend_matrix.rs
    └── cancel_propagation.rs
```

## 4. Key flows

### 4.1 Cache backend selection

```
Cache::connect("")           => InProcess(DashMap::new())
Cache::connect("redis://...") => Redis(client)
Cache::connect("rediss://...") => Redis(client, tls=true)
```

No live failover; restart required to switch.

### 4.2 Cancel propagation

```
HTTP POST /v1/sessions/{id}/cancel
   ├─ load session, check ownership
   ├─ registry.cancel(id)
   └─ 200 { "cancelled": true }

Agent loop sees token cancelled at next iteration boundary -> Final(cancelled).
```

### 4.3 Graceful shutdown

```
sigterm received
   ├─ ShutdownSignal broadcast()
   ├─ HTTP server stops accepting new connections
   ├─ in-flight sessions drain (bounded by max_iterations × tool timeout)
   ├─ scheduler stops accepting new fires
   └─ Storage pool closes
```

## 5. Error handling

```rust
pub enum CacheError {
    Connect(String),
    InvalidUrl(String),
    Redis(String),
    Serialization(String),
}
```

`InvalidUrl` is fatal at startup; in-process boots on any non-redis URL except `""` is also valid (selects in-process).

## 6. Concurrency / transactions

- `Cache::incr` is atomic in both backends (DashMap entry API; Redis `INCR`).
- `CancellationRegistry` uses `tokio_util::sync::CancellationToken` so consumers can `select!` on it.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | TTL precision (in-process honours sub-second; redis clamps); URL parser |
| Integration | `cache_backend_matrix.rs` — same key-value cycle across both backends (with feature flag `with_redis` for CI nightly); `cancel_propagation.rs` — token observed across spawn |
| System | Test strategy L1: rate limiter survives backend swap; smoke `MR-CACHE-001` |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-RUNTIME-001
  type: LLD
  title: xiaoguai-runtime LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents: [HLD-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-RT-001, title: "No live cache failover", statement: "Cache backend selected at boot; switching modes requires restart.", status: approved, scope: in, decision: "Boot-time selection.", rationale: "Avoids in-flight-state migration complexity; ephemeral state acceptable to lose." }
  flows:
    - { id: FLOW-LLD-RT-001, title: "Graceful shutdown", statement: "broadcast signal -> stop accept -> drain in-flight -> close pools.", status: approved, scope: in, kind: state_transition }
  test_cases: []
relations:
  - { id: REL-LLD-RT-001, type: refines, from: DEC-LLD-RT-001, to: DEC-HLD-002, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
