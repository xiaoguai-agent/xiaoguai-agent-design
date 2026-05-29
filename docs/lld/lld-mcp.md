# LLD — `xiaoguai-mcp`

| | |
|---|---|
| Document ID | `LLD-MCP-001` |
| Refines | `DEC-HLD-001` (single binary; supervisor in-process) |
| Verifies | `REQ-MCP-001`, `REQ-MCP-002`, `REQ-MCP-003`, `REQ-MCP-005` |

## 1. Module purpose

MCP client + supervisor. Provides:

- A `McpClient` trait with three transports: `Stdio`, `Http`, `Sse`.
- A `Supervisor` that holds a pool of per-tenant `McpClient` instances and serves the `Toolbox` consumed by `xiaoguai-agent`.
- A marketplace catalog loader.
- An optional MCP **server** mode mounted via `POST /v1/mcp/serve` that re-exposes the Toolbox outbound.

## 2. Public interface

```rust
#[async_trait]
pub trait McpClient: Send + Sync {
    async fn list_tools(&self) -> Result<Vec<ToolSpec>, McpError>;
    async fn call_tool(&self, name: &str, args: serde_json::Value)
        -> Result<ToolResult, McpError>;
}

pub struct Supervisor { /* ... */ }
impl Supervisor {
    pub async fn ensure_loaded(&self, tenant: TenantId, server_id: McpServerId)
        -> Result<Arc<dyn McpClient>, McpError>;
    pub async fn toolbox(&self, tenant: TenantId) -> Result<Toolbox, McpError>;
}

pub struct Toolbox { /* (tool_name -> Arc<dyn McpClient>) */ }
```

## 3. Module structure

```
crates/xiaoguai-mcp/
├── src/
│   ├── lib.rs                 # public re-exports
│   ├── supervisor.rs          # Supervisor + per-tenant pool
│   ├── toolbox.rs
│   ├── transport/
│   │   ├── stdio.rs           # spawn child + JSON-RPC over pipes
│   │   ├── http.rs            # request/response over HTTP
│   │   └── sse.rs             # server-sent events
│   ├── marketplace.rs
│   ├── servers/
│   │   └── github_pr.rs       # bundled in-tree example
│   └── error.rs
└── tests/
    ├── stdio_roundtrip.rs
    └── tool_isolation.rs
```

## 4. Key flows

### 4.1 Tenant tool dispatch

```
xiaoguai-agent ── Toolbox::call(tool_name, args) ──►
   Supervisor::ensure_loaded(tenant, server_id_for_tool)
      ├─ check cache; if missing:
      │     spawn child / open HTTP client / open SSE
      │     handshake; list_tools(); cache
      └─ return Arc<dyn McpClient>
   client.call_tool(name, args)
      ├─ enforce per-call timeout (default 30s, configurable per server)
      └─ return ToolResult
```

### 4.2 Stdio child lifecycle

- Child process spawned with `tokio::process::Command`, `kill_on_drop=true`.
- Stdin/stdout JSON-RPC framed by newline.
- Stderr scrubbed via `xiaoguai-audit::redact` then captured into structured logs.
- On crash: per-tenant supervisor restarts up to 3 times within 60 s; further crashes mark the server `unhealthy` until operator intervention.

### 4.3 Outbound MCP server mode

When mounted, the agent's `Toolbox` is exposed via the MCP wire protocol on an HTTP endpoint authenticated by bearer JWT. Tool list is filtered by the caller's role.

### 4.4 OAuth 2.1 PKCE for outbound HTTP transport (v0.5.4.2, T4, PR #73)

The HTTP transport (`HttpMcpClient`) consults a `TokenStore` before every request when the configured `auth` is `oauth2_pkce`. New module surface in `auth/oauth2_pkce.rs`:

```rust
pub enum AuthConfig {
    None,
    Bearer { token: String },
    OAuth2Pkce(OAuth2PkceConfig),
}

pub struct OAuth2PkceConfig {
    pub auth_url: String,
    pub token_url: String,
    pub client_id: String,
    pub scopes: Vec<String>,
    pub redirect_uri: String,   // bound by CLI to 127.0.0.1:<random>/callback
}

pub struct TokenBundle {
    pub access_token: String,
    pub refresh_token: Option<String>,
    pub expires_at: DateTime<Utc>,
}

#[async_trait]
pub trait TokenStore: Send + Sync {
    async fn get(&self, server_id: &str, tenant_id: &str) -> McpResult<Option<TokenBundle>>;
    async fn put(&self, server_id: &str, tenant_id: &str, bundle: &TokenBundle) -> McpResult<()>;
}

pub fn new_pkce_pair() -> PkcePair;     // hand-rolled: rand → base64url-no-pad → SHA256 challenge
pub async fn exchange_code(http: &Client, cfg: &OAuth2PkceConfig, code: &str, verifier: &str) -> McpResult<TokenBundle>;
pub async fn refresh_pkce(http: &Client, cfg: &OAuth2PkceConfig, bundle: &TokenBundle) -> McpResult<TokenBundle>;
```

Implementation choices (per DEC-015):

- **Hand-rolled PKCE** rather than the `oauth2` crate. ~120 LOC using existing `rand 0.10` + `sha2 0.11` + `base64 0.22` workspace deps. Avoids `getrandom 0.3` transitive-dep churn.
- **Verifier**: 64 random bytes → base64url-no-pad → 86-char string (within RFC 43..128 bound).
- **Challenge**: `base64url_no_pad(sha256(verifier))` → 43 chars. `code_challenge_method=S256`.
- **Refresh threshold**: 60 s. When `now + 60s >= expires_at`, `connect_with_store` invokes `refresh_pkce` and atomically persists the new bundle before attaching `Authorization: Bearer`.
- **Refresh-token rotation**: per RFC 6749 §6, if the token endpoint returns a new `refresh_token`, the stored bundle is updated. If not, the old `refresh_token` is preserved.
- **TLS**: on by default. `XIAOGUAI_MCP_OAUTH_INSECURE=1` toggles `reqwest::ClientBuilder::danger_accept_invalid_certs(true)`, logged at `warn` (operators-only escape hatch for self-signed dev environments).

CLI flow `xiaoguai mcp register --auth oauth2-pkce …` binds a `127.0.0.1:0/callback` listener with a single-route handler, prints the consent URL, waits ≤ 5 minutes for the redirect, validates `state`, exchanges the code synchronously, persists. No browser auto-open (works on headless servers).

## 5. Error handling

```rust
pub enum McpError {
    Transport(String),
    HandshakeFailed,
    ToolNotFound(String),
    ToolError { code: String, message: String },
    Timeout,
    Unhealthy,
    SignatureMismatch,    // for signed marketplace packs
    AuthRequired,         // v0.5.4.2 — OAuth flow not yet completed for this (server, tenant)
    AuthFailed(String),   // v0.5.4.2 — token refresh / exchange failed
}
```

## 6. Concurrency / transactions

- One `Supervisor` per process; tenants share spawn capacity (configurable global cap).
- Tool call dispatch is `Send + Sync`; multiple concurrent calls per server use the same JSON-RPC stream with id correlation.
- Restart counter is in-memory; resets on process restart.

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | JSON-RPC framing roundtrip, stderr redaction passes through `xiaoguai-audit::redact` |
| Integration | `stdio_roundtrip.rs` — spawn a fixture server, call a tool, assert result; `tool_isolation.rs` — tenant A cannot see tenant B's MCP server registrations |
| System | Test strategy L2: `/v1/mcp/servers` returns only the caller's tenant's servers; agent loop end-to-end via `xiaoguai-mcp-exec` |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-MCP-001
  type: LLD
  title: xiaoguai-mcp LLD
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
    - { id: DEC-LLD-MCP-001, title: "In-process supervisor with kill_on_drop", statement: "Supervisor spawns children with kill_on_drop=true so process termination always cleans up.", status: approved, scope: in, decision: "kill_on_drop.", rationale: "Avoid zombie children on operator-induced restarts." }
    - { id: DEC-LLD-MCP-002, title: "Stderr redaction in transport", statement: "Stdio transport scrubs child stderr through xiaoguai-audit::redact before logging.", status: approved, scope: in, decision: "Redact at transport edge.", rationale: "Prevents PII leak into logs from arbitrary MCP servers." }
  flows:
    - { id: FLOW-LLD-MCP-001, title: "Tool dispatch", statement: "Toolbox -> Supervisor -> McpClient::call_tool.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-MCP-001, type: refines, from: DEC-LLD-MCP-002, to: REQ-NFR-007, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
