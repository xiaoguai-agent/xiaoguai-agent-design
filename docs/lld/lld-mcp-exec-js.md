# LLD — `xiaoguai-mcp-exec-js`

| | |
|---|---|
| Document ID | `LLD-MCPEXECJS-001` |
| Type | Low-Level Design |
| Status | `draft` |
| Refines | `DEC-HLD-017` (polyglot sandbox tier — separate crates per language), PHILO §14 (sandbox tiering) |
| Verifies | `REQ-TOOL-EXEC-002` (Hermes parity, JavaScript path) |
| Updated | 2026-05-29 |

## 1. Module purpose

Standalone MCP server binary that exposes the `execute_javascript` tool. Sibling crate to [`lld-mcp-exec.md`](lld-mcp-exec.md) for the Python path. The crates share **no code at the binary boundary** by design — see DEC-017 for the trust-isolation rationale.

**L3 path now shipping**: per ADR-0020 + DEC-020, the wasmtime + QuickJS-WASM backend lives in [`xiaoguai-mcp-exec-wasm`](lld-mcp-exec-wasm.md). This crate's `runtime` module (added in sprint-8 PR #78) defines `ExecBackend` and provides the L1 default `ProcessL1JavaScript`; the L3 binary `xiaoguai-mcp-exec-wasm-js` constructs `WasmtimeJavaScriptBackend` and passes it to `ExecServer::with_backend(cfg, backend)`.

The trait surface is documented in [`lld-mcp-exec.md` § ExecBackend trait](lld-mcp-exec.md#exec-backend-trait). It is **re-defined in this crate too** (not imported from a shared utility) per the trust-boundary argument.

Default JavaScript runtime is **Deno** (`--allow-none` for capability-denied execution); `--runtime node` is an operator opt-in. The choice is documented in `docs/designs/tier2-mcp-exec-js.md`; rejected alternative `boa_engine` is also recorded there.

## 2. Public interface

```rust
pub struct ExecJsConfig {
    pub timeout_secs: u32,         // 1..=300, hard cap, default 30
    pub memory_mb: u32,            // ulimit -v, default 1024 (V8 needs more headroom than CPython)
    pub workdir_parent: PathBuf,   // OS temp by default
    pub runtime: JsRuntime,        // Deno (default) | Node
    pub redact_stderr: bool,       // default true
}

pub enum JsRuntime {
    Deno {
        deno_bin: PathBuf,                  // "deno"
        extra_allow: Vec<String>,           // strict default is ["--allow-none"]
    },
    Node {
        node_bin: PathBuf,                  // "node"
        extra_flags: Vec<String>,           // strict default is ["--no-deprecation"]
    },
}

pub async fn run_stdio_server(cfg: ExecJsConfig) -> Result<(), ExecJsError>;
```

CLI binary `xiaoguai-mcp-exec-js` parses these via clap + env vars:
- `XIAOGUAI_MCP_EXEC_JS__TIMEOUT_SECS`
- `XIAOGUAI_MCP_EXEC_JS__MEMORY_MB`
- `XIAOGUAI_MCP_EXEC_JS__WORKDIR_PARENT`
- `XIAOGUAI_MCP_EXEC_JS__RUNTIME` (`deno` | `node`)
- `XIAOGUAI_MCP_EXEC_JS__NO_REDACT`

## 3. Module structure

```
crates/xiaoguai-mcp-exec-js/
├── src/
│   ├── main.rs            # clap + envar; delegates to run_stdio_server
│   ├── lib.rs             # run_stdio_server + ExecJsConfig
│   ├── exec.rs            # subprocess wrapper (mirrors xiaoguai-mcp-exec/sandbox.rs)
│   ├── tools.rs           # execute_javascript tool definition (schema + handler)
│   ├── server.rs          # ExecServer: ServerHandler + stdio binding
│   └── error.rs
└── tests/
    ├── ...                # 15 pure-Rust unit + 8 gated-spawn (skip if runtime not on PATH)
```

## 4. Key flows

### 4.1 `execute_javascript` invocation

1. Tool call arrives via `rmcp::ServerHandler::call_tool`.
2. Reject snippet > 64 KB before spawn (same as Python path).
3. Create fresh `tempfile::TempDir`; write snippet to `main.js`.
4. Build subprocess command:
   - **Deno**: `/bin/sh -c "ulimit -v ${memory_mb}*1024; exec deno run --allow-none main.js"`
   - **Node**: `/bin/sh -c "ulimit -v ${memory_mb}*1024; exec node --no-deprecation main.js"`
5. `env_clear()`, then re-add only `PATH`, `LANG`, `LC_ALL`, `LC_CTYPE`. **Explicitly excluded**: `OLLAMA_HOST`, `DATABASE_URL`, `XIAOGUAI_AUDIT_SIGNING_KEY`.
6. `kill_on_drop(true)` so tokio's deadline → future drop → SIGKILL to the child shell, which reaps the runtime.
7. `tokio::timeout(deadline, child.wait_with_output())`.
8. Stdout / stderr captured; cap at 64 KB / stream with truncation marker; `redact_str` stderr if `redact_stderr=true`.
9. Return `ExecJsResult { exit_code, stdout, stderr, duration_ms, truncated, timed_out }`.

### 4.2 Why Deno default

Per DEC-017, Deno's `--allow-none` flag means the runtime itself enforces capability denial — the snippet cannot open network connections, read files outside its workdir, or spawn processes, **regardless of what the snippet tries to call**. Node.js requires our wrapper to enforce these (and the snippet can sometimes work around it via `child_process` even with `ulimit`).

### 4.3 Why separate trust boundary

A Python sandbox escape via CPython interpreter vulnerability and a JavaScript sandbox escape via V8 JIT vulnerability are independent attack surfaces. Sharing the binary means a single 0-day in either path compromises both. Separate crates → separate binaries → separate process spaces → composable failures stay isolated.

Operators also get independent HotL buckets: `tool_call.execute_python` and `tool_call.execute_javascript` have separate budgets and per-tool deny policies.

## 5. Error handling

```rust
pub enum ExecJsError {
    SnippetTooLarge(usize),
    TempdirCreate(std::io::Error),
    Spawn(std::io::Error),
    RuntimeNotFound(String),       // deno or node binary not on PATH
    Timeout,
    ChildIo(std::io::Error),
}
```

## 6. Concurrency / transactions

- Each invocation is a fresh subprocess with its own tempdir — no shared state.
- The MCP `Tool` handler is `Send + Sync`; concurrent invocations are allowed but each holds an independent `Child` and `TempDir`.
- No persistence: results return to the caller, caller persists if needed (via the agent loop's audit emission).

## 7. Test design

| Layer | Cases |
|---|---|
| Unit (pure Rust) | Schema validation, payload round-trip, `[WRITE]` description marker, output-cap policy, env allow-list contents (15 tests) |
| Integration (gated on `deno` or `node` on PATH) | Happy path, non-zero exit, timeout-kill (5 s sleep, 500 ms deadline → killed), stderr PII redaction, stdout cap, **env_secrets_do_not_leak_into_sandbox** (parent sets `XIAOGUAI_AUDIT_SIGNING_KEY=must-not-leak-into-sandbox` → sandbox sees "env scrubbed"), workdir fresh per call (8 tests) |
| System | Operator runs `xiaoguai mcp register --transport stdio --command $(which xiaoguai-mcp-exec-js)`, then `xiaoguai chat` triggers `execute_javascript` — covered by the follow-up agent-loop wiring PR (mirror PR #66 pattern) |

Gated tests SKIP with `"SKIPPED: deno not on PATH"` (or `node`) so CI runners without the runtime installed see a clean pass.

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-MCPEXECJS-001
  type: LLD
  title: xiaoguai-mcp-exec-js LLD
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-29
  refines: [DEC-HLD-017]
  verifies: [REQ-TOOL-EXEC-002]
```
<!-- TRACEABILITY-METADATA:END -->
