# API Contract — Xiaoguai v1.4

| | |
|---|---|
| Document ID | `API-XIAOGUAI-001` |
| Type | API Contract (HTTP REST + SSE + MCP) |
| Status | `draft` |
| Author | Zhou Wei |
| Source | [`PRD-XIAOGUAI-001`](prd-xiaoguai.md), implementation repo `crates/xiaoguai-api/src/routes/` and `crates/xiaoguai-mcp/` |
| Updated | 2026-05-31 |

This contract is the **single source of truth** for the HTTP, SSE, and MCP surfaces. HLD and LLD reference identifiers from this document; they do not redefine endpoints.

---

## 1. Conventions

### 1.1 Base URL and versioning

- Default listen address: `0.0.0.0:7600`.
- All versioned routes live under `/v1/**`. The unversioned `/healthz` is reserved for liveness probes and is not subject to versioning.
- Breaking changes bump the path prefix (`/v2/**`). Additive (new field, new endpoint) changes are allowed within `/v1/**`. See [Deprecation policy](#13-deprecation-policy).

### 1.2 Authentication

- Bearer JWT in the `Authorization: Bearer <token>` header for all `/v1/**` routes.
- JWTs are minted via the OIDC flow; RS256/ES256, TTL ≤ 15 minutes; refresh tokens rotated and stored in cache backend.
- One exception: the **public scheduler webhook** `POST /v1/scheduler/webhooks/{route_id}` uses an opaque token in the `X-Xiaoguai-Token` header instead of a JWT.
- Inbound IM webhooks (`/v1/im/<platform>/event`) use the platform's native signature scheme (HMAC, Ed25519, AES) — no JWT required.

### 1.3 Tenancy

Every authenticated request carries a `tenant_id` claim inside the JWT. The server:

1. Sets `SET LOCAL app.current_tenant_id = '<claim>'` on every PG transaction for RLS;
2. Adds `WHERE tenant_id = $1` in every query (defence-in-depth);
3. Refuses to dispatch admin-only operations to non-admin roles via Casbin.

### 1.4 Content negotiation

- Default content type: `application/json; charset=utf-8`.
- SSE endpoints respond with `text/event-stream`.
- Request bodies that exceed **1 MiB** are rejected with `413 Payload Too Large`.

### 1.5 Pagination

List endpoints accept `?limit=<n>&cursor=<opaque>`. Default `limit=50`, maximum `limit=200`. Responses include:

```json
{
  "items": [ ... ],
  "next_cursor": "opaque-string-or-null"
}
```

Clients that omit the cursor receive the first page. The cursor is opaque and not parseable by clients.

### 1.6 Error envelope

All non-2xx responses use this shape:

```json
{
  "error": {
    "code": "snake_case_error_code",
    "message": "Human-readable explanation suitable for surfacing to operators.",
    "details": { "field": "value" },
    "trace_id": "opentelemetry-trace-id-hex"
  }
}
```

| HTTP status | `error.code` examples |
|---|---|
| 400 | `invalid_request`, `validation_failed` |
| 401 | `unauthenticated`, `invalid_token` |
| 403 | `forbidden`, `tenant_mismatch`, `policy_unavailable` |
| 404 | `not_found` |
| 409 | `conflict`, `optimistic_lock_failed` |
| 413 | `payload_too_large` |
| 422 | `unprocessable_entity` |
| 429 | `rate_limited` |
| 500 | `internal_error` |
| 503 | `dependency_unavailable` |

### 1.7 Idempotency

Mutation endpoints accept `Idempotency-Key: <uuid>`. The server stores the response for 24 hours; replays of the same key return the cached response without re-executing the side effect.

### 1.8 Rate limiting

Per-tenant rate limits are configured in `tenant_rate_limit_config`. Exhaustion returns `429` with `Retry-After: <seconds>`. The token-bucket counters live in the cache backend (Redis/Valkey in HA, DashMap in single-instance).

---

## 2. HTTP route table

### 2.1 Public

| Method | Path | Description | Auth | Verifies |
|---|---|---|---|---|
| `GET` | `/healthz` | Liveness probe; returns `200 {"status":"ok"}` whenever the process is healthy | — | REQ-NFR-005 |

### 2.2 Sessions and chat

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/sessions` | List sessions for the caller's user_id (paginated) | REQ-CHAT-001 |
| `POST` | `/v1/sessions` | Create a new session | REQ-CHAT-001 |
| `GET` | `/v1/sessions/{id}` | Fetch session metadata | REQ-CHAT-001 |
| `GET` | `/v1/sessions/{id}/messages` | List prior turns (paginated) | REQ-CHAT-001 |
| `POST` | `/v1/sessions/{id}/messages` | Send a user message and stream the agent reply (SSE) | REQ-CHAT-001, REQ-CHAT-004 |
| `POST` | `/v1/sessions/{id}/cancel` | Cancel the in-flight agent run on this session | REQ-CHAT-002 |
| `POST` | `/v1/sessions/{id}/fork` | Fork a new session from this session at a given message_id | REQ-CHAT-003 |

#### 2.2.1 SSE stream envelope

`POST /v1/sessions/{id}/messages` returns `text/event-stream`. Event types:

```
event: delta
data: {"chunk": "tok"}

event: tool_call
data: {"id": "call_01", "tool": "execute_python", "args": {...}}

event: tool_result
data: {"id": "call_01", "output": {...}, "duration_ms": 312}

event: final
data: {"message_id": "msg_...", "stop_reason": "stop"}

event: error
data: {"error": {"code": "...", "message": "..."}}
```

`stop_reason` ∈ `{stop, max_iterations, cancelled, escalated, policy_unavailable}`.

### 2.3 MCP management

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/mcp/servers` | List registered MCP servers visible to the caller's tenant | REQ-MCP-001 |
| `GET` | `/v1/mcp/marketplace` | List available marketplace entries | REQ-MCP-003 |
| `POST` | `/v1/mcp/marketplace/install` | Install a marketplace entry into the caller's tenant | REQ-MCP-003 |
| `POST` | `/v1/mcp/serve` | (Optional, operator opt-in) Mount the local Toolbox as an outbound MCP server | REQ-MCP-005 |

#### 2.3.1 MCP server record shape

```json
{
  "id": "uuid",
  "tenant_id": "uuid",
  "transport": "stdio" | "http" | "sse",
  "command": "xiaoguai-mcp-exec",
  "args": ["--timeout-secs", "30", "--memory-mb", "512"],
  "env": {"PATH": "/usr/local/bin:/usr/bin:/bin"},
  "enabled": true,
  "signature": "ed25519:base64...",
  "labels": ["sandbox", "first-party"]
}
```

### 2.4 Admin

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/admin/tenants` | List tenants | REQ-AUTH-004 |
| `GET` | `/v1/admin/audit` | Query audit log (filters: actor, action, time range, tenant) | REQ-AUD-001 |
| `GET` | `/v1/admin/audit/verify` | Verify the HMAC chain integrity for a tenant | REQ-AUD-002 |
| `GET` | `/v1/admin/today` | Today's event summary (used by chat-ui Today view) | REQ-OUT-003 |
| `GET` | `/v1/admin/eval/suites` | List eval suite catalog | REQ-EVAL-001 |
| `POST` | `/v1/admin/eval/run` | Execute an eval suite, returns run_id | REQ-EVAL-001 |
| `POST` | `/v1/admin/eval/case-from-session` | Materialise a `.eval.yaml` case from a session transcript | REQ-EVAL-002 |

#### 2.4.1 Audit verify response

```json
{
  "tenant_id": "uuid",
  "head_sequence": 4711,
  "head_hmac_b64": "abc...",
  "valid": true,
  "broken_at_sequence": null,
  "checked_at": "2026-05-28T08:30:00Z"
}
```

When tampering is detected `valid=false, broken_at_sequence=<n>` is set. No automated repair — append-only by design.

### 2.5 Scheduler

| Method | Path | Description | Verifies |
|---|---|---|---|
| `POST` | `/v1/scheduler/webhooks/{route_id}` | External webhook trigger (X-Xiaoguai-Token auth) | REQ-SCHED-003 |
| `POST` | `/v1/admin/scheduler/webhooks/{route_id}` | Internal webhook trigger (bearer JWT auth) | REQ-SCHED-001 |
| `GET` | `/v1/admin/scheduler/jobs` | List scheduled jobs | REQ-SCHED-001 |
| `POST` | `/v1/admin/scheduler/jobs` | Create or update a job | REQ-SCHED-001 |
| `POST` | `/v1/admin/scheduler/jobs/{id}/fire-now` | Trigger now (for testing) | REQ-SCHED-001 |
| `GET` | `/v1/admin/scheduler/tokens` | List webhook tokens | REQ-SCHED-003 |
| `POST` | `/v1/admin/scheduler/tokens` | Create a webhook token | REQ-SCHED-003 |
| `DELETE` | `/v1/admin/scheduler/tokens/{id}` | Revoke a webhook token | REQ-SCHED-003 |

#### 2.5.1 Trigger record shape

```json
{
  "id": "uuid",
  "type": "cron" | "interval" | "proactive" | "file_watch" | "webhook" | "git_push" | "db_poll",
  "cron_spec": "0 9 * * 1-5",
  "interval_secs": null,
  "route_id": null,
  "path_globs": null,
  "checker_prompt": null
}
```

Each job has at most one trigger plus one or more push sinks.

### 2.6 HotL policies + decisions

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/hotl/policies` | List HotL policies for caller's tenant | REQ-HOTL-004 |
| `POST` | `/v1/hotl/policies` | Create or upsert a policy | REQ-HOTL-001 |
| `DELETE` | `/v1/hotl/policies/{id}` | Delete a policy | REQ-HOTL-004 |
| `POST` | `/v1/hotl/decisions` | Record an operator verdict on a pending escalation; optionally raise a policy in the same transaction | REQ-HOTL-002, REQ-HOTL-003 |

#### 2.6.1 Policy shape

```json
{
  "id": "uuid",
  "tenant_id": "uuid",
  "scope": "tool_call.execute_python",
  "window_secs": 3600,
  "max_count": 50,
  "max_spend_usd": "10.00",
  "escalate_to": "ops@acme.com",
  "enabled": true
}
```

#### 2.6.2 Decision shape (`POST /v1/hotl/decisions`)

Operator verdicts on escalated tool calls. Sprint-11 (S11-3a.1) shipped the **record-only** path; sprint-12 (S11-3a.2) flipped the `resumed` field to dynamic when a suspended agent loop is waiting. Sprint-13 (DEC-HLD-016) finishes the contract cleanup: the canonical identifier is `escalation_id` end-to-end, authorisation is the dedicated Casbin scope `hotl:decide`, and `escalation_id` references the `hotl_escalations` parent row introduced by migration 0027.

> **Shipped:** xiaoguai PR #146 (backend `escalation_id` rename end-to-end), PR #147 (chat-ui rename), PR #143 (Casbin `hotl:decide` scope + DB-backed policy merge), PR #138 (migration 0027 — `hotl_escalations` parent + `redaction_policies` + Casbin row).

Request:

```json
{
  "escalation_id": "uuid",       // canonical (sprint-13). The legacy `request_id` field is no longer accepted; requests using it return 400.
  "verdict": "allow",            // "allow" | "deny"
  "decided_by": "ops@acme.com",  // sprint-12: ignored when Claims carries a subject
  "raise_policy": {              // optional — "Approve & remember" UX
    "scope": "tool_call.execute_python",
    "window_secs": 3600,
    "max_count": 50,
    "max_spend_usd": "10.00",
    "rationale": "Pre-approved for incident #4421"
  }
}
```

Response (`201 Created`):

```json
{
  "id": "uuid",                  // decision row id
  "escalation_id": "uuid",       // matches the request; FK to hotl_escalations(id)
  "verdict": "allow",
  "recorded_at": "2026-05-30T08:12:34Z",
  "resumed": false,              // sprint-11: always false (no waiter exists, loop never suspended).
                                 // sprint-12 (S11-3a.2): true when DecisionRegistry resolved a live oneshot
                                 // waiter; false when the decision arrived after the loop already timed out
                                 // or the agent was never suspended (e.g. policy returned Allow client-side).
                                 // sprint-13: also true when DecisionRegistry resolved a waiter that was
                                 // *replayed* on API boot (DEC-HLD-013) — the registry layer is opaque to the route.
  "policy_created": {            // present iff raise_policy was supplied AND policy was created in-tx
    "id": "uuid",
    "scope": "tool_call.execute_python",
    "max_count": 50,
    "enabled": true
  }
}
```

Status codes:

| Code | When |
|---|---|
| `201 Created` | Decision recorded (`resumed` may be `true` or `false`). |
| `400 Bad Request` | Body uses legacy `request_id` field (sprint-13 rename; see DEC-HLD-016); or `escalation_id` malformed. Response carries `{"error":"field","field":"escalation_id","message":"..."}`. |
| `401 Unauthorized` | Bearer missing or invalid. |
| `403 Forbidden` | Caller's Casbin scopes do not include `hotl:decide` (sprint-13: the sprint-11 path-based fallback `p, *, /v1/hotl/decisions, POST` is removed by migration 0027). Granted by the `tenant_admin` and `operator` roles. |
| `404 Not Found` | Unknown `escalation_id` — no row in `hotl_escalations` for this tenant. Sprint-12 added the parent table; sprint-13's migration 0027 makes it the authoritative parent for `hotl_pending` children, so 404 now means the parent does not exist (not merely that no pending row matches). |
| `409 Conflict` | Duplicate decision for the same `escalation_id`. Operators must inspect the existing record. |
| `503 Service Unavailable` | `HotlEscalationStore` not wired in production `AppState` (should not occur post-sprint-13; DEC-HLD-013 makes the PG impl the default). |

#### 2.6.3 SSE events — `hotl_pending` and `hotl_resolved`

Two SSE event variants on the chat message stream (`POST /v1/sessions/{id}/messages`, `Accept: text/event-stream`). Both are sprint-12 additions; sprint-11 has neither (the agent loop does not suspend — see [`lld-agent.md`](lld/lld-agent.md) §4.5). Sprint-13 renames the identifier field from `request_id` to `escalation_id` end-to-end (DEC-HLD-016) — the alias is removed.

> **Shipped:** xiaoguai PR #146 (`AgentEvent::HotlPending` / `HotlResolved` rename + SSE field rename), PR #147 (chat-ui `HotlBanner` state keyed by `escalation_id`).

`hotl_pending` — emitted by `xiaoguai-agent` when `HotlGate::check` returns `Suspend`:

```json
{
  "type": "hotl_pending",
  "escalation_id": "uuid",
  "tool": "execute_python",
  "args_redacted": { "password": "***" },    // policy-driven; per-tenant hotl_redaction_policies (DEC-HLD-014)
  "scope": "tool_call.execute_python",
  "expires_at": "2026-05-31T08:12:34Z"       // per-scope-class timeout (DEC-HLD-015); same clock as decisions.recorded_at
}
```

`args_redacted` is computed by `xiaoguai-auth::redaction::RedactionRules::apply(scope, args)`; matching JSONPath selectors are replaced with `"***"`. The field is always present; an empty redaction policy degrades to args verbatim and a boot warning. `expires_at` uses the per-scope class lookup (`tool` / `mcp` / `skill`) with the global `default_expiry` as fallback.

`hotl_resolved` — emitted by `xiaoguai-agent` after `DecisionRegistry` resolves the waiter (verdict from `POST /v1/hotl/decisions`) OR after the server-side timeout fires:

```json
{
  "type": "hotl_resolved",
  "escalation_id": "uuid",
  "verdict": "allow",            // "allow" | "deny" | "timeout"
  "decided_by": "ops@acme.com",  // omitted when verdict = "timeout"
  "recorded_at": "2026-05-30T08:13:01Z"
}
```

Frontend contract (cross-ref [`lld-chat-ui.md`](lld/lld-chat-ui.md) §4.3): on `hotl_pending`, render `<HotlBanner>` in the pending state, keyed by `escalation_id`. On `hotl_resolved`, clear the banner. Sprint-11's chat-ui has neither hook wired — it relies on an optimistic-clear timeout after a successful `POST /v1/hotl/decisions`; sprint-12 (S11-3a.2) removes that timeout and waits for `hotl_resolved` exclusively. Sprint-13 chat-ui must also update its banner state key from `request_id` to `escalation_id` in lockstep — there is no compat path.

### 2.7 Skill packs

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/skills/catalog` | List packs available in the operator-curated catalog | REQ-SKILL-001 |
| `GET` | `/v1/skills/installed` | List packs installed in the caller's tenant | REQ-SKILL-001 |
| `POST` | `/v1/skills/install` | Install a pack (requires valid signature unless `allow_unsigned=true`) | REQ-SKILL-002, REQ-SKILL-003 |
| `DELETE` | `/v1/skills/install/{id}` | Uninstall a pack | REQ-SKILL-002 |

#### 2.7.1 Install request

```json
{
  "pack_id": "skill-pack-github-pr-helper",
  "version": "1.4.0",
  "allow_unsigned": false,
  "knob_overrides": {"max_files_per_pr": 30}
}
```

Response includes `install_id` and `effective_signature` (the signature actually validated, for audit).

### 2.8 Memory

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/memories` | List memories visible to caller | REQ-MEM-001 |
| `POST` | `/v1/memories` | Create a memory (server embeds via configured backend) | REQ-MEM-001 |
| `GET` | `/v1/memories/{id}` | Fetch one | REQ-MEM-001 |
| `PUT` | `/v1/memories/{id}` | Update content / metadata / expires_at | REQ-MEM-001, REQ-MEM-004 |
| `DELETE` | `/v1/memories/{id}` | Delete | REQ-MEM-001 |
| `POST` | `/v1/memories/recall` | Top-k semantic recall by query string | REQ-MEM-001 |
| `GET` | `/v1/memories/similar/{id}` | Top-k similar to an existing memory | REQ-MEM-001 |

#### 2.8.1 Memory shape

```json
{
  "id": "uuid",
  "tenant_id": "uuid",
  "user_id": "uuid",
  "persona_id": "uuid-or-null",
  "content": "User prefers terse responses.",
  "tags": ["preference", "communication"],
  "expires_at": "2027-05-28T00:00:00Z",
  "created_at": "2026-05-28T08:00:00Z"
}
```

`content_embedding` (384-dim) is never returned to clients.

#### 2.8.2 Recall request / response

```json
// POST /v1/memories/recall
{ "query": "How does this user like responses?", "limit": 5, "tags_filter": ["preference"] }

// 200
{
  "items": [
    { "memory": { ... }, "score": 0.87 }
  ],
  "model": "all-minilm",
  "duration_ms": 42
}
```

### 2.9 Workspaces

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/workspaces` | List workspaces | REQ-WS-001 |
| `POST` | `/v1/workspaces` | Create | REQ-WS-001 |
| `PUT` | `/v1/workspaces/{id}` | Update | REQ-WS-001 |
| `DELETE` | `/v1/workspaces/{id}` | Archive (soft delete) | REQ-WS-001 |

### 2.10 Outcomes (ROI telemetry)

| Method | Path | Description | Verifies |
|---|---|---|---|
| `POST` | `/v1/outcomes` | Record an outcome attribution row | REQ-OUT-001 |
| `GET` | `/v1/outcomes/summary` | Aggregated totals (caller's tenant only) | REQ-OUT-002 |
| `GET` | `/v1/outcomes/timeseries` | Daily-bucketed series | REQ-OUT-002 |

#### 2.10.1 Outcome shape

```json
{
  "id": "uuid",
  "tenant_id": "uuid",
  "session_id": "uuid-or-null",
  "category": "deal_closed" | "support_resolved" | "research_hour_saved" | "custom",
  "revenue_usd": "0.00",
  "cost_saved_usd": "12.50",
  "hours_saved": "0.75",
  "deals_closed": 0,
  "attribution_confidence": 0.9,
  "recorded_at": "2026-05-28T09:00:00Z"
}
```

### 2.11 Tenants (client config)

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/tenants/{id}/config` | Tenant client config (e.g., `ai_disclosure_banner`, default persona) | REQ-CHAT-001 |

### 2.12 Usage

| Method | Path | Description | Verifies |
|---|---|---|---|
| `GET` | `/v1/usage` | Token usage aggregation, paginated and filterable by `(provider_id, model, user_id, session_id, from, to)` | REQ-LLM-005 |

---

## 3. MCP tool surface

Xiaoguai is an MCP client that consumes external MCP servers, AND ships one in-tree MCP server (`xiaoguai-mcp-exec`). It can also optionally serve its Toolbox outbound via `POST /v1/mcp/serve` (REQ-MCP-005).

### 3.1 Bundled MCP servers (in-tree)

| Server | Tools | Transport | Crate |
|---|---|---|---|
| `xiaoguai-mcp-exec` | `execute_python` | stdio | `crates/xiaoguai-mcp-exec` |
| `github_pr` (example, demo) | `gh_pr_comment`, `gh_pr_review` | http | `crates/xiaoguai-mcp/src/servers/github_pr.rs` |

### 3.2 `execute_python` contract (verifies REQ-MCP-004, BEH-MCPEXEC-001, BEH-MCPEXEC-002)

**Input:**

```jsonc
{
  "code": "string",                    // required, Python source
  "timeout_secs": 30                   // optional, integer 1..=60, clamped to --timeout-secs
}
```

**Output (text content block, JSON-encoded):**

```jsonc
{
  "exit_code": 0,                      // null on timeout
  "stdout": "...",                     // up to 64 KiB
  "stderr": "...",                     // up to 64 KiB, PII-redacted unless --no-redact-stderr
  "duration_ms": 312,
  "truncated": false,                  // true if any output capped
  "timed_out": false
}
```

**Sandbox guarantees:**

- Fresh tempdir as CWD, dropped on completion regardless of outcome.
- Env scrubbed to allowlist (`PATH`, `LANG`, `LC_*`); secrets never propagate.
- `ulimit -v` (address-space) and tokio `timeout` enforce memory and wall-clock caps.
- Stderr passes through the same PII redactor as the audit chain.
- Output capped at 64 KiB per channel.

**Out-of-scope:** network egress is not blocked at the sandbox layer; rely on k8s NetworkPolicy or `--network none` at the container layer.

### 3.3 Outbound MCP serving

When `POST /v1/mcp/serve` is invoked (or the equivalent CLI flag is set), `xiaoguai` exposes its skill-pack Toolbox via the MCP wire format. Authentication is bearer-JWT-scoped; an unauthenticated caller cannot see the tool list.

OAuth 2.1 PKCE for outbound MCP servers reachable from `xiaoguai` is a Tier-3a roadmap item — see PRD §2.2.

---

## 4. IM gateway endpoints

| Platform | Inbound webhook | Signature scheme | Notes |
|---|---|---|---|
| Feishu | `POST /v1/im/feishu/event` | HMAC-SHA256 over `(timestamp, nonce, body)` | encrypt-key supported |
| DingTalk | `POST /v1/im/dingtalk/event` (fallback) + WebSocket Stream | timestamp + sign HMAC | WebSocket Stream preferred |
| WeCom | `POST /v1/im/wecom/event` | XML + AES-CBC | quick-xml parsing |
| Discord | `POST /v1/im/discord/interactions` | Ed25519 | dalek crate |
| Slack | `POST /v1/im/slack/events` + Socket Mode | HMAC-SHA256 over `v0:<timestamp>:<body>` | Socket Mode for outbound NAT |
| Telegram | `POST /v1/im/telegram/{token}` + long-polling | path secret token | polling fallback |
| Mattermost | `POST /v1/im/mattermost/event` | bearer token (outgoing webhook) | slash-command supported |

All adapters return `200 OK` with platform-required body within ≤ 1.5 s; long-running work is enqueued to the agent loop asynchronously.

---

## 5. Data domain reference

Cross-cutting type aliases used by the API:

| Alias | Concrete type | Notes |
|---|---|---|
| `TenantId` | UUID | foreign key on every tenant-scoped table |
| `SessionId` | UUID | references `sessions(id)` |
| `MessageId` | UUID | references `messages(id)` |
| `UserId` | UUID | references `users(id)` |
| `ProviderId` | UUID | references `llm_providers(id)` |
| `ToolCallId` | string (`call_<base32>`) | client-correlation token in SSE |
| `Money` | string (decimal, USD) | `"12.50"` form to preserve precision |
| `Cursor` | opaque string | not parseable by clients |
| `Timestamp` | RFC 3339 UTC | always with `Z` suffix |

---

## 6. Auth scopes and RBAC

Casbin policy is loaded from `casbin_policies` table at boot and on `POST /v1/admin/casbin/reload` (operator-only).

Reference scopes:

| Scope | Routes |
|---|---|
| `chat` | `/v1/sessions/**`, `/v1/memories/**`, `/v1/skills/installed`, `/v1/skills/install`, `/v1/workspaces/**` (self), `/v1/outcomes` (self) |
| `tenant_admin` | All of `chat`, plus `/v1/hotl/policies/**`, `/v1/skills/catalog`, `/v1/mcp/servers`, `/v1/mcp/marketplace/**`, `/v1/admin/scheduler/**` (own tenant) |
| `operator` | All of `tenant_admin`, plus `/v1/admin/tenants`, `/v1/admin/audit/**`, `/v1/admin/eval/**`, `/v1/admin/today`, `/v1/admin/casbin/**` |
| `scheduler_webhook` | `/v1/scheduler/webhooks/{route_id}` (X-Xiaoguai-Token, not JWT) |

Cross-tenant operator views are gated by Casbin and audited separately.

---

## 7. Streaming contracts

| Channel | Transport | Backpressure | Reconnect |
|---|---|---|---|
| Chat SSE (`POST /v1/sessions/{id}/messages`) | SSE | Server applies `tokio::sync::mpsc` backpressure; if client cannot keep up, server stops the LLM stream | Client can re-invoke same request with `Idempotency-Key` to resume; agent loop is idempotent at tool-call boundary |
| Admin audit tail (`GET /v1/admin/audit?stream=true`) | SSE | Server batches every 250 ms | Stateless; client filters duplicates |
| Today view (`GET /v1/admin/today?stream=true`) | SSE | Same | Same |
| DingTalk Stream | WebSocket | tokio-tungstenite | Auto-reconnect with exponential backoff |
| Slack Socket Mode | WebSocket | Same | Same |

---

## 8. SLO targets (non-binding, tracked in perf-regression CI)

| Endpoint | P50 | P95 | P99 |
|---|---|---|---|
| `GET /healthz` | < 1 ms | < 5 ms | < 10 ms |
| `POST /v1/memories/recall` (100 k rows, HNSW) | 80 ms | 300 ms | 600 ms |
| `POST /v1/sessions/{id}/messages` first-token (Ollama 7B local) | 800 ms | 1.5 s | 3 s |
| `POST /v1/outcomes` | 30 ms | 100 ms | 200 ms |
| `GET /v1/admin/audit/verify` (chain length 1 M) | 5 s | 15 s | 30 s |

---

## 9. Backwards compatibility

| Rule | Statement |
|---|---|
| BC-01 | Adding a new optional request field is non-breaking. |
| BC-02 | Adding a new optional response field is non-breaking. |
| BC-03 | Removing or renaming any request or response field bumps to `/v2/**`. |
| BC-04 | Changing an enum to add a value is non-breaking; clients must tolerate unknown values. |
| BC-05 | Tightening a validation (e.g., max length reduced) requires a deprecation cycle. |
| BC-06 | New SSE event types are additive; clients must skip unknown events. |
| BC-07 | The MCP tool input schema MUST NOT remove or rename input properties within a major MCP server version. |

---

## 10. Deprecation policy

When an endpoint is deprecated:

1. The endpoint continues to return correct responses.
2. A `Deprecation: true` and `Sunset: <RFC 7231 date>` header is added.
3. A `Warning: 299 - "deprecated; see ..."` header is added.
4. The change appears in the next `RELEASE_NOTES.md`.

Sunset is ≥ 6 months from the deprecation announcement for `tenant_admin`-and-below endpoints, ≥ 3 months for `operator` endpoints.

---

## 11. PRD coverage matrix

| Endpoint family | PRD requirements covered |
|---|---|
| Sessions / SSE | REQ-CHAT-001, 002, 003, 004, 005; BEH-CHAT-001 |
| LLM admin | REQ-LLM-001, 002, 003, 005 |
| MCP | REQ-MCP-001, 002, 003, 004, 005; REQ-NFR-003 |
| Memories | REQ-MEM-001, 002, 003, 004; BEH-MEM-001 |
| Workspaces / tasks | REQ-WS-001, REQ-TASK-001, 002 |
| IM | REQ-IM-001, 002, 003, 004; REQ-NFR-004; BEH-IM-001 |
| Auth / tenants | REQ-AUTH-001, 002, 003, 004 |
| Audit | REQ-AUD-001, 002, 003, 004, 005; BEH-AUDIT-001 |
| HotL | REQ-HOTL-001, 002, 003, 004; BEH-HOTL-001 |
| Skills | REQ-SKILL-001, 002, 003, 004 |
| Scheduler | REQ-SCHED-001, 002, 003, 004, 005 |
| Outcomes | REQ-OUT-001, 002, 003 |
| Eval | REQ-EVAL-001, 002 |
| Usage | REQ-LLM-005 |
| Non-functional | REQ-NFR-001, 002, 003, 005, 007, 008 |

Out-of-scope (PRD waivers): `REQ-SKILL-005`, `REQ-AUD-006`, `REQ-EVAL-003` content seeding.

---

## 12. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema:
  name: testany-traceability
  version: "1.0.0"
  profile: prd-profile-v1
artifact:
  id: API-XIAOGUAI-001
  type: API_CONTRACT
  title: Xiaoguai HTTP/REST + SSE + MCP API contract
  status: draft
  owners: [engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-31
  source_documents:
    - PRD-XIAOGUAI-001
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions: []
  flows: []
  test_cases: []
relations:
  - { id: REL-API-001, type: derived_from, from: API-XIAOGUAI-001, to: PRD-XIAOGUAI-001, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
