# LLD — `xiaoguai-mcp-exec`

| | |
|---|---|
| Document ID | `LLD-MCPEXEC-001` |
| Refines | `DEC-HLD-005` (sandbox via ulimit + tokio timeout) |
| Verifies | `REQ-MCP-004`, `BEH-MCPEXEC-001`, `BEH-MCPEXEC-002`, `GR-SEC-07` |

## 1. Module purpose

Standalone MCP server binary that exposes the `execute_python` tool. It is a separate crate + bin so that it can be supervised, container-isolated, and replaced (e.g., wasmtime+pyodide per DEC-018) without touching the rest of the system.

**Sibling crate**: `xiaoguai-mcp-exec-js` (PR #75, T6, DEC-017) ships the same L1 contract for JavaScript via Deno (default) or Node. See [`lld-mcp-exec-js.md`](lld-mcp-exec-js.md). The two crates intentionally share NO code at the binary boundary so a CVE in either language runtime cannot compose. The shared L1 contract (env scrub allow-list, ulimit, tokio deadline, output cap, redaction) lives in PHILO §14.

**Future L3 path**: per ADR-0020, the wasmtime+pyodide implementation will extract an `ExecBackend` trait from this crate's `sandbox.rs`; L1 becomes the default impl and `WasmtimeBackend` becomes an opt-in via a new sibling crate `xiaoguai-mcp-exec-wasm`. Implementation deferred 2–3 sprints.

## 2. Public interface

This crate exposes a single binary `xiaoguai-mcp-exec` plus a library entry point for in-tree tests:

```rust
pub struct ExecConfig {
    pub timeout_secs: u32,        // 1..=300, hard cap
    pub memory_mb: u32,           // ulimit -v
    pub workdir_parent: PathBuf,  // OS temp by default
    pub python: PathBuf,          // python3
    pub redact_stderr: bool,      // default true
}

pub async fn run_stdio_server(cfg: ExecConfig) -> Result<(), ExecError>;
```

## 3. Module structure

```
crates/xiaoguai-mcp-exec/
├── src/
│   ├── main.rs            # parses CLI flags + envar; delegates to run_stdio_server
│   ├── lib.rs             # run_stdio_server + ExecConfig
│   ├── sandbox.rs         # spawn helper: env scrub, ulimit -v, tempdir
│   ├── tool.rs            # execute_python tool definition (schema + handler)
│   ├── redact.rs          # re-export from xiaoguai-audit
│   └── error.rs
└── tests/
    ├── timeout.rs         # infinite loop killed at deadline
    ├── env_scrub.rs       # XIAOGUAI_AUDIT_SIGNING_KEY not visible
    ├── stderr_redact.rs   # PII redacted
    └── output_cap.rs      # 64 KB cap
```

## 4. Key flows

### 4.1 `execute_python` invocation

```
client (xiaoguai-mcp Supervisor as MCP client)
  ── JSON-RPC: tools/call name=execute_python args={code,timeout_secs}
       │
       ▼
xiaoguai-mcp-exec
  ├─ validate args
  ├─ effective_timeout = min(args.timeout_secs, cfg.timeout_secs)
  ├─ create tempdir under cfg.workdir_parent
  ├─ build env from allowlist (PATH, LANG, LC_*)
  ├─ build command:
  │     /bin/sh -c "ulimit -v $(memory_mb*1024); exec $python -c '$code'"
  ├─ spawn with kill_on_drop=true, cwd=tempdir, env=scrubbed
  ├─ tokio::select! {
  │     timeout(effective_timeout) => kill_process_group();
  │     output = wait_with_output() => normal path;
  │   }
  ├─ cap stdout/stderr each at 64 KiB; set truncated flags
  ├─ if cfg.redact_stderr: xiaoguai-audit::redact(&mut stderr_str)
  ├─ drop tempdir (recursive remove)
  └─ JSON-RPC response: { exit_code, stdout, stderr, duration_ms, truncated, timed_out }
```

### 4.2 Timeout semantics

- On deadline: `exit_code=null`, `timed_out=true`, `stdout`/`stderr` may be empty because the process was killed before pipes were drained.
- Per-call `timeout_secs` is clamped to `cfg.timeout_secs` to prevent caller-supplied DoS.

### 4.3 Memory cap

- `ulimit -v` applied via the `sh -c` wrapper so the cap is in effect before `python` `exec`s.
- On macOS the cap is best-effort; production deploys MUST be Linux containers (HLD §7.1, GR-OPS-04 deferred).

## 5. Error handling

```rust
pub enum ExecError {
    SpawnFailed(io::Error),
    SerializationFailed(serde_json::Error),
    WorkdirCreate(io::Error),
    Internal(String),
}
```

Tool-level errors (Python traceback) are returned as a normal tool result with `exit_code != 0`, not as an `ExecError`. The MCP framing layer treats `ExecError` as a server-side fault.

## 6. Concurrency / transactions

- Tokio multi-thread runtime; concurrent calls each get their own tempdir + process group.
- No shared state between calls; the server is stateless.
- HotL is **not** consulted here — the gate is upstream in `xiaoguai-agent`.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | env allowlist computation, output cap arithmetic, JSON-RPC tool schema serialisation |
| Integration | `timeout.rs` — `while True: pass` killed at 1 s; `env_scrub.rs` — print(os.environ.get('XIAOGUAI_AUDIT_SIGNING_KEY')) emits None; `stderr_redact.rs` — print('Bearer xxx', file=sys.stderr) redacted; `output_cap.rs` — print('x'*200_000) returns 65536-byte stdout, truncated=true |
| System | Test strategy L1: registered as MCP server, agent calls `execute_python`, HotL counter increments |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-MCPEXEC-001
  type: LLD
  title: xiaoguai-mcp-exec LLD
  status: draft
  owners: [engineering.xiaoguai, security.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents: [HLD-XIAOGUAI-001, GUARDRAILS-XIAOGUAI-001]
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions:
    - { id: DEC-LLD-MCPEXEC-001, title: "Defence in depth at five layers", statement: "Sandbox combines tempdir, env scrub, ulimit, tokio timeout, output cap, stderr redaction.", status: approved, scope: in, decision: "Five-layer defence.", rationale: "Each layer addresses a distinct threat class; failure of any one does not collapse the threat model." }
    - { id: DEC-LLD-MCPEXEC-002, title: "HotL out of scope here", statement: "HotL gating is upstream in agent ReAct loop; server remains stateless.", status: approved, scope: in, decision: "Stateless server.", rationale: "Avoids per-tenant state in the sandbox; supports horizontal scaling." }
  flows:
    - { id: FLOW-LLD-MCPEXEC-001, title: "execute_python lifecycle", statement: "validate -> spawn with sandbox -> wait/timeout -> cap+redact -> drop tempdir.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-MCPEXEC-001, type: refines, from: DEC-LLD-MCPEXEC-001, to: DEC-HLD-005, status: active }
  - { id: REL-LLD-MCPEXEC-002, type: mitigates, from: DEC-LLD-MCPEXEC-001, to: RISK-SEC-003, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
