# LLD — `xiaoguai-mcp-exec-wasm`

| | |
|---|---|
| Document ID | `LLD-MCPEXECWASM-001` |
| Type | Low-Level Design |
| Status | `draft` |
| Refines | `DEC-HLD-018` (L3 path wasmtime+pyodide), `DEC-HLD-019` (`ExecBackend` trait extraction), `DEC-HLD-020` (single crate hosting both language modules) |
| Verifies | `REQ-TOOL-EXEC-002` extended to L3, ADR-0020 phasing §2 |
| Updated | 2026-05-30 |

## 1. Module purpose

Single crate providing the **L3 sandbox tier** of PHILO §14 for both Python
(`WasmtimePythonBackend`) and JavaScript (`WasmtimeJavaScriptBackend`).
Both backends implement the [`ExecBackend`](lld-mcp-exec.md#exec-backend-trait)
trait defined in the L1 crates so they slot in as runtime-swappable
backends per DEC-019.

Two binaries ship from this crate:

- `xiaoguai-mcp-exec-wasm-py` — Python via wasmtime + pyodide
- `xiaoguai-mcp-exec-wasm-js` — JavaScript via wasmtime + QuickJS-WASM

Operators register them in `mcp_servers` rows the same way they register
`xiaoguai-mcp-exec` (L1 Python) or `xiaoguai-mcp-exec-js` (L1 JS); the
supervisor decides which to spawn per tenant by reading
`tenant_settings.sandbox_tier` (see `TenantSettingsReader::sandbox_tier`
in [`lld-tasks.md`](lld-tasks.md)).

## 2. Public interface

```rust
pub struct WasmConfig {
    pub timeout_secs: u32,        // mapped to wasmtime epoch ticks (1 tick = 10 ms)
    pub memory_mb: u32,           // mapped to Store::limiter()
    pub workdir_parent: PathBuf,  // tempdir scope (no FS access from inside WASM)
    pub asset_path: PathBuf,      // operator-provided pyodide / QuickJS asset dir
}

pub struct WasmtimePythonBackend { /* shared Engine + cached pyodide Module */ }
pub struct WasmtimeJavaScriptBackend { /* shared Engine + cached QuickJS Module */ }

impl WasmtimePythonBackend {
    pub fn new(cfg: WasmConfig) -> Result<Self, WasmError>;
}
impl WasmtimeJavaScriptBackend {
    pub fn new(cfg: WasmConfig) -> Result<Self, WasmError>;
}

// Both implement xiaoguai_mcp_exec::runtime::ExecBackend (Python)
// or xiaoguai_mcp_exec_js::runtime::ExecBackend (JS) respectively.
```

Each binary also exposes `pub async fn run_stdio_server(cfg: WasmConfig)`
for in-tree integration tests.

## 3. Module structure

```
crates/xiaoguai-mcp-exec-wasm/
├── src/
│   ├── lib.rs
│   ├── engine.rs              # shared wasmtime::Engine singleton + epoch-tick thread
│   ├── assets.rs              # asset loader (pyodide.wasm / quickjs.wasm fetched via scripts/fetch-wasm-assets.sh)
│   ├── wasmtime_python.rs     # WasmtimePythonBackend impl
│   ├── wasmtime_javascript.rs # WasmtimeJavaScriptBackend impl
│   ├── config.rs              # WasmConfig + parsing
│   ├── error.rs               # WasmError enum
│   └── bin/
│       ├── wasm_py.rs         # main for xiaoguai-mcp-exec-wasm-py
│       └── wasm_js.rs         # main for xiaoguai-mcp-exec-wasm-js
└── tests/
    └── ...                    # 9 unconditional + 7 asset-gated × 2 languages
```

## 4. Key flows

### 4.1 Shared engine singleton (DEC-020 — memory amortisation)

A single `wasmtime::Engine` lives for the process lifetime. Both binaries
fetch it via `engine::get_engine()` on cold start. The engine config
enables **epoch-based interruption** (not deadline-based) because
pyodide's CPython interpreter does tight syscall loops that don't yield
to deadline checks. An epoch-tick thread runs at 100 Hz (`Duration::from_millis(10)`),
incrementing the engine epoch; per-call `Store::set_epoch_deadline(N)`
maps `cfg.timeout_secs * 100` ticks.

### 4.2 `WasmtimePythonBackend::run` (per-call lifecycle)

1. Snippet > 64 KB rejected (same threshold as L1).
2. Create a fresh `Store` with the engine reference + a `Limiter` enforcing `cfg.memory_mb`.
3. Set epoch deadline = `timeout_secs * 100`.
4. Instantiate the cached pyodide `Module` into the store (per-call instantiation; engine + module precompile are reused).
5. Write `snippet` into pyodide's virtual filesystem at `/main.py` (via wasmtime memory API).
6. Call pyodide's exported `runPython` entry point.
7. Capture stdout/stderr from pyodide's redirected streams.
8. On epoch-interruption (timeout), `Store` traps and we return `ExecResult { timed_out: true, exit_code: None, … }`.
9. On `Store::limiter` rejection (OOM), we return `ExecError::SnippetMemoryExceeded`.
10. `Drop` of the `Store` releases all wasmtime allocations.

Target cold-start (cached engine + cached module): **≤ 50 ms** mid-tier hardware. Hard fail in release-mode tests if observed > 200 ms.

### 4.3 `WasmtimeJavaScriptBackend::run`

Mirrors §4.2 with QuickJS-WASM instead of pyodide. QuickJS-WASM's `eval`
entry point is simpler than pyodide's `runPython`; the per-call delta is
~5 ms faster.

### 4.4 Asset provisioning (operator-side)

The crate does **not** fetch pyodide or QuickJS-WASM at build time
(per Track A sub-plan §6 — no `build.rs` network access in CI/sandbox).
Operators provision assets via:

```bash
bash scripts/fetch-wasm-assets.sh ~/.xiaoguai/wasm-assets/
export XIAOGUAI_WASM_ASSET_DIR=~/.xiaoguai/wasm-assets/
xiaoguai-mcp-exec-wasm-py
```

Tests follow the same gating idiom as the L1 JS Deno-on-PATH check (PR #75):
mark spawn-tests `#[ignore]` when `XIAOGUAI_WASM_ASSET_DIR` is unset.

## 5. Error handling

```rust
pub enum WasmError {
    EngineInit(wasmtime::Error),
    AssetLoad { path: PathBuf, cause: std::io::Error },
    AssetParse { name: &'static str, cause: wasmtime::Error },
    SnippetTooLarge(usize),
    StoreLimitExceeded,         // memory cap hit
    TrapDuringInstantiation(wasmtime::Error),
    Timeout,                    // mapped from epoch interruption
    HostInterfaceMissing(&'static str),
}
```

## 6. Concurrency / transactions

- Engine is `Arc<Engine>` shared; cheap to clone.
- Each invocation owns its own `Store` and `Instance` — no shared state between snippets.
- The epoch-tick thread is one tokio task per process, parked on `tokio::time::interval(10ms)`.
- No persistence at the crate boundary — caller (the agent loop) handles audit emission.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit (unconditional) | `WasmConfig` parsing, asset-path resolution, engine singleton idempotence, `ExecBackend::name` and `::capability_summary` stable labels. Python: 9 tests; JS: 10 tests; shared engine/asset: 6 tests. |
| Integration (asset-gated; skip if `XIAOGUAI_WASM_ASSET_DIR` unset) | Happy path (`print(7**7)` → `823543` for py; `console.log(7**7)` for JS), epoch-interruption timeout (5 s infinite loop → 500 ms deadline → `timed_out: true`), memory-limit reject (`x = " " * 10**9` → `StoreLimitExceeded`), env scrub (snippet can't see host env at all — verify both languages), snippet > 64 KB rejection. |
| System (release builds only) | `cold_start_under_200ms` — measures the cached-engine cold-start of `run` against the 200 ms ceiling; warns in debug, hard-fails in release. |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-MCPEXECWASM-001
  type: LLD
  title: xiaoguai-mcp-exec-wasm LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-30
  refines: [DEC-HLD-018, DEC-HLD-019, DEC-HLD-020]
  verifies: [REQ-TOOL-EXEC-002, ADR-0020-phase-2]
```
<!-- TRACEABILITY-METADATA:END -->
