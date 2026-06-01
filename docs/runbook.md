# Runbook — Xiaoguai v1.4

| | |
|---|---|
| Document ID | `RUNBOOK-XIAOGUAI-001` |
| Type | Operator runbook |
| Status | `draft` |
| Source | [`HLD-XIAOGUAI-001`](hld.md), LLDs under [`lld/`](lld/), implementation repo `docs/runbooks/` |
| Updated | 2026-05-28 |

Consolidates operational guidance for single-tenant air-gap and multi-tenant SaaS deployments. Specific procedure refs back to the underlying ADR or LLD when needed.

---

## 1. Pre-flight

### 1.1 System requirements

| Component | Requirement |
|---|---|
| OS | Linux x86_64 or aarch64 (production). macOS supported for development only. |
| Kernel | ≥ 5.10 |
| Postgres | 17 with `pgvector` extension installable |
| Ollama | ≥ 0.3 (air-gap default); the model `all-minilm` (memory) and at least one chat model (e.g., `qwen2.5-coder:7b`) pulled |
| Resources (single-tenant) | 4 vCPU / 8 GiB RAM / 30 GiB disk minimum; 16 GiB recommended if hosting Ollama on the same node |
| Outbound network | Optional. Air-gap default does NOT require internet. |

### 1.2 Secrets and config

Place under `~/.xiaoguai/` (chmod 600):

```yaml
# config.yaml
database:
  url: postgres://xg:****@localhost:5432/xiaoguai
  max_connections: 16
cache:
  url: ""               # empty -> in-process backend
  key_prefix: "xiaoguai:"
auth:
  oidc_issuer: https://idp.example.com
audit:
  hmac_key_env: XIAOGUAI_AUDIT_SIGNING_KEY
mcp_exec:
  timeout_secs: 30
  memory_mb: 512
scheduler:
  enabled: true
  tick_interval_secs: 30
```

Environment variables:

| Variable | Purpose |
|---|---|
| `XIAOGUAI_AUDIT_SIGNING_KEY` | HMAC key (hex, ≥ 32 bytes). REQUIRED. |
| `XIAOGUAI_AUDIT_REDACT_PII` | `true` (default) or `false`. |
| `XIAOGUAI_CACHE__URL` | Override `cache.url`. |
| `OLLAMA_HOST` | If set, selects `OllamaEmbedder`; if unset, `InMemoryEmbedder` (dev only). |
| `XIAOGUAI_OTEL__EXPORT_URL` | Optional OTLP exporter; absence disables export. |
| `XIAOGUAI_IM__USE_IN_PROCESS_HISTORY` | Dev-only opt-in. Production must be `false`. |
| `XIAOGUAI_IM__MAX_MESSAGES_PER_CONVERSATION` | Default 50. |

---

## 2. Deployment — Helm (recommended)

### 2.1 Install

```bash
kubectl create namespace xiaoguai

# Create four mandatory Secrets first:
kubectl -n xiaoguai create secret generic xiaoguai-postgres \
  --from-literal=DATABASE_URL='postgres://xg:****@xiaoguai-postgres:5432/xiaoguai'
kubectl -n xiaoguai create secret generic xiaoguai-audit \
  --from-literal=hmac_key="$(openssl rand -hex 32)"
kubectl -n xiaoguai create secret generic xiaoguai-oidc \
  --from-literal=client_id='xiaoguai' --from-literal=client_secret='****'
kubectl -n xiaoguai create secret generic xiaoguai-im \
  --from-literal=feishu_app_secret='****'

helm install xiaoguai deploy/helm/xiaoguai \
  --namespace xiaoguai \
  --set image.tag=v1.4.0 \
  --values your-values.yaml
```

### 2.2 Verify deploy

```bash
kubectl -n xiaoguai rollout status deploy/xiaoguai --timeout=180s
kubectl -n xiaoguai exec deploy/xiaoguai -- /usr/local/bin/xiaoguai-core smoke
curl -sS http://xiaoguai.xiaoguai.svc:8080/healthz | jq
```

`smoke` connects to every declared dependency and returns non-zero on any failure.

### 2.3 Boot logs to confirm

```
cache: in-process backend (no Redis/Valkey URL configured)        # air-gap mode
cache: connected to Redis/Valkey                                  # HA mode
memory: selected embedding backend choice=Ollama("http://...")
memory: selected embedding backend choice=InMemory                # dev mode
serve: audit PII redaction ENABLED  (XIAOGUAI_AUDIT_REDACT_PII)
serve: audit PII redaction DISABLED via XIAOGUAI_AUDIT_REDACT_PII
```

---

## 3. Deployment — single-tenant air-gap

```bash
# 1. Pull tarball release
curl -LO https://github.com/xiaoguai-agent/xiaoguai/releases/download/v1.4.0/xiaoguai-v1.4.0-linux-amd64.tar.gz
tar xzf xiaoguai-v1.4.0-linux-amd64.tar.gz -C /opt/

# 2. systemd unit (uses deploy/systemd/xiaoguai-core.service)
sudo cp /opt/xiaoguai/deploy/systemd/xiaoguai-core.service /etc/systemd/system/

# 3. Environment file (chmod 600)
cat <<EOF | sudo tee /etc/xiaoguai/env
XIAOGUAI_AUDIT_SIGNING_KEY=$(openssl rand -hex 32)
DATABASE_URL=postgres://xg:****@localhost:5432/xiaoguai
OLLAMA_HOST=http://localhost:11434
EOF
sudo chmod 600 /etc/xiaoguai/env

# 4. Boot
sudo systemctl daemon-reload
sudo systemctl enable --now xiaoguai-core
sudo systemctl status xiaoguai-core
```

Air-gap profile (default `values.yaml`) does NOT require Valkey, S3, or cloud LLM endpoints.

---

## 4. Upgrade

```bash
# 1. Snapshot
pg_dump --format=custom -d xiaoguai > xiaoguai-pre-upgrade.dump

# 2. Helm upgrade
helm -n xiaoguai upgrade xiaoguai deploy/helm/xiaoguai --set image.tag=v1.4.1 --reuse-values

# 3. Verify migrations applied
kubectl -n xiaoguai exec deploy/xiaoguai -- xiaoguai admin migrate status
# Should show every migration as Applied.

# 4. Verify audit chain
kubectl -n xiaoguai exec deploy/xiaoguai -- \
  curl -sS -H "Authorization: Bearer $ADMIN_JWT" \
  http://localhost:8080/v1/admin/audit/verify?tenant_id=<bootstrap> | jq
# Expect: {"valid":true, ...}
```

If the new release adds migrations, they run automatically on first boot. A failed migration exits the process; the rollout-status will report Unavailable.

---

## 5. Rollback

| Scenario | Procedure |
|---|---|
| Bad release | `helm -n xiaoguai rollback xiaoguai N` where N = previous revision. |
| Bad migration | Roll image back; restore DB from `xiaoguai-pre-upgrade.dump` (last-known-good); audit chain remains valid because migrations are forward-only and never edit `audit_log`. |
| Bad config | `helm -n xiaoguai upgrade xiaoguai ... --reuse-values --set <key>=<good-value>` then restart pods. |

Rollback NEVER edits `audit_log` rows directly. If `verify` reports `valid=false` after rollback, investigate before doing anything else (chain integrity must be defended).

---

## 6. HMAC key rotation

The audit chain HMAC key MUST be rotated periodically. Procedure (30-day dual-key window):

```bash
# 1. Snapshot the current chain tail
kubectl -n xiaoguai exec deploy/xiaoguai -- xiaoguai admin audit head > pre-rotation.json

# 2. Create the new key secret
kubectl -n xiaoguai create secret generic xiaoguai-audit-next \
  --from-literal=hmac_key="$(openssl rand -hex 32)"

# 3. Helm upgrade: switch primary to new secret, keep old as the verifier key
helm -n xiaoguai upgrade xiaoguai deploy/helm/xiaoguai \
  --reuse-values \
  --set secrets.audit_primary=xiaoguai-audit-next \
  --set secrets.audit_verifier=xiaoguai-audit

# 4. Verify chain still valid (verifier accepts both keys)
kubectl -n xiaoguai exec deploy/xiaoguai -- \
  curl -sS -H "Authorization: Bearer $ADMIN_JWT" \
  http://localhost:8080/v1/admin/audit/verify?tenant_id=<tenant> | jq

# 5. After 30 days (calendar reminder), drop the old secret
kubectl -n xiaoguai delete secret xiaoguai-audit
helm -n xiaoguai upgrade xiaoguai ... --set secrets.audit_verifier=
```

Failure to keep the 30-day verifier window means historical entries signed with the old key will fail verification; the chain APPEARS broken even though no data was tampered (RISK-OPS-002).

---

## 7. Disaster recovery

| Scenario | Procedure |
|---|---|
| Postgres lost | Restore most recent `pg_dump` → run `xiaoguai-core smoke` → restart pods → `audit/verify` to confirm chain. |
| Cache backend lost (Valkey mode) | Restart pods (state is ephemeral); rate-limit windows reset. |
| Whole node lost (single-tenant tarball) | Reinstall binary, restore Postgres, re-mount config + secrets, systemctl start. |
| Suspected tenant data leak | `xiaoguai admin audit verify --tenant <id>`; `GET /v1/admin/audit?actor=<suspected>&from=…&to=…`; export evidence via `/v1/admin/audit?stream=true`. |
| Image registry compromise | `cosign verify` before redeploy; revoke and re-sign latest tag. |

---

## 8. Observability and alerts

| Channel | Endpoint / source | Use |
|---|---|---|
| Liveness | `GET /healthz` | k8s probe |
| Readiness | `GET /healthz?ready=true` | k8s probe |
| Prometheus metrics | `GET /metrics` | Local pull only |
| OTLP traces | `XIAOGUAI_OTEL__EXPORT_URL` | Off by default; redacted by `RedactingSpanExporter` |
| Logs | stdout / `journalctl -u xiaoguai-core` | JSON via `tracing-subscriber` |
| Audit | `/v1/admin/audit`, `audit_log` table | Always on |

### 8.1 Suggested alerts

| Alert | Condition |
|---|---|
| `xiaoguai_db_unreachable` | Probe failure for 60 s |
| `xiaoguai_audit_chain_broken` | `verify` reports `valid=false` |
| `xiaoguai_hotl_escalation_spike` | > 3 escalations / minute for any tenant |
| `xiaoguai_token_usage_anomaly` | Z-score > 3 over 1 h baseline |
| `xiaoguai_image_size_regress` | Daily build > 200 MB |
| `xiaoguai_ollama_unreachable` | `LLM router` cascades for 5 min |

---

## 9. Operational procedures

### 9.1 Kill a runaway session

```bash
curl -X POST http://xiaoguai.xiaoguai.svc:8080/v1/sessions/<sess-id>/cancel \
  -H "Authorization: Bearer $OPERATOR_JWT"
```

Bounded by slowest in-flight tool call.

### 9.2 Disable a tenant immediately

```sql
UPDATE tenants SET enabled = false WHERE id = '<tenant-id>';
```

Effective on the next request (auth middleware re-checks).

### 9.3 Disable an MCP server

```sql
UPDATE mcp_servers SET enabled = false WHERE id = '<id>';
```

Or via API:

```bash
curl -X PATCH http://xiaoguai/v1/mcp/servers/<id> \
  -H "Authorization: Bearer $ADMIN_JWT" \
  -d '{"enabled": false}'
```

### 9.4 Pause an LLM provider

```sql
UPDATE llm_providers SET enabled = false WHERE id = '<id>';
```

If the disabled provider is the only one for a model alias, requests to that alias return `503`.

### 9.5 Seed a HotL policy

```bash
xiaoguai hotl policy create \
  --tenant-id <tenant> \
  --scope tool_call.execute_python \
  --window-secs 3600 \
  --max-count 50 \
  --escalate-to ops@acme.com
```

### 9.6 Switch memory backend

```bash
export OLLAMA_HOST=http://localhost:11434
ollama pull all-minilm
kubectl -n xiaoguai rollout restart deploy/xiaoguai
kubectl -n xiaoguai logs deploy/xiaoguai | grep "memory: selected embedding backend"
# choice=Ollama("http://localhost:11434")
```

To switch back to InMemory (smoke only):

```bash
kubectl -n xiaoguai set env deploy/xiaoguai OLLAMA_HOST-
```

### 9.7 Switch cache backend

```bash
# in-process -> Valkey
kubectl -n xiaoguai set env deploy/xiaoguai XIAOGUAI_CACHE__URL=redis://xiaoguai-valkey:6379/0
kubectl -n xiaoguai rollout restart deploy/xiaoguai
# log: cache: connected to Redis/Valkey
```

Switching modes loses in-process state (rate limit windows, session locks). This is by design — ephemeral state is not persisted.

### 9.8 Provision OIDC `scopes` claim for HotL operators (sprint-14 — DEC-HLD-019, GR-SEC-17)

`xiaoguai-api` enforces `hotl:decide` (and sprint-14's `hotl:policy:{read,write}`) by reading the JWT `scopes` claim. Most OIDC issuers do **not** emit a `scopes` claim by default — they emit `groups` or `roles`. This step makes the production issuer emit `scopes` so the extractor (DEC-HLD-018) sees the values it expects. Skipping this step manifests as `403 scope_required` on every `POST /v1/hotl/decisions`.

**Verification.** First confirm what your tokens currently carry:

```bash
TOKEN=$(curl -s -X POST "$OIDC_ISSUER/token" \
  -d grant_type=password -d username="$OPERATOR_USER" -d password="$OPERATOR_PASS" \
  -d client_id="$XIAOGUAI_CLIENT_ID" | jq -r .access_token)

# Decode the payload:
echo "$TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq '.scopes // .scope // "ABSENT"'

# Or hit /v1/me and look at the parsed scope list:
curl -sH "Authorization: Bearer $TOKEN" "$XIAOGUAI_URL/v1/me" | jq '.scopes'
```

If the field reads `"ABSENT"` or `[]`, apply one of the per-issuer recipes below.

#### Keycloak (recommended for self-hosted)

```bash
# Create a client scope named `xiaoguai-hotl` with a hard-coded scope mapper.
kcadm.sh create client-scopes -r xiaoguai -s name=xiaoguai-hotl -s protocol=openid-connect
SCOPE_ID=$(kcadm.sh get client-scopes -r xiaoguai -q name=xiaoguai-hotl --fields id -F json | jq -r '.[0].id')

# Add a `hardcoded-claim` mapper producing the scopes claim as a JSON array.
kcadm.sh create client-scopes/$SCOPE_ID/protocol-mappers/models -r xiaoguai \
  -s name=hotl-scopes -s protocol=openid-connect \
  -s protocolMapper=oidc-hardcoded-claim-mapper \
  -s 'config."claim.name"=scopes' \
  -s 'config."claim.value"=["hotl:decide","hotl:policy:read","hotl:policy:write"]' \
  -s 'config."jsonType.label"=JSON' \
  -s 'config."id.token.claim"=false' \
  -s 'config."access.token.claim"=true'

# Attach the scope to the xiaoguai client as default.
kcadm.sh update clients/$XIAOGUAI_CLIENT_DBID/default-client-scopes/$SCOPE_ID -r xiaoguai
```

Restrict the scope assignment to the `operator` and `tenant_admin` realm-roles via a role-conditional client-scope (Keycloak 22+ feature) or use a script mapper that reads `realm_access.roles` and emits only the matching scope subset.

#### Auth0 (Actions API)

In the Auth0 dashboard, add a Post-Login Action:

```javascript
exports.onExecutePostLogin = async (event, api) => {
  const roles = (event.authorization?.roles ?? []);
  const scopes = [];
  if (roles.includes('operator') || roles.includes('tenant_admin'))
    scopes.push('hotl:decide');
  if (roles.includes('tenant_admin') || roles.includes('operator'))
    scopes.push('hotl:policy:read', 'hotl:policy:write');
  if (scopes.length)
    api.accessToken.setCustomClaim('scopes', scopes);
};
```

Deploy → assign Action to the Login flow → next operator login carries the claim.

#### Okta (Custom Authorization Server)

Create a custom claim in the authorization server (Security → API → Authorization Servers → Default → Claims):

| Field | Value |
|---|---|
| Name | `scopes` |
| Include in token type | `Access Token` |
| Value type | `Expression` |
| Value | `Arrays.flatten(Arrays.filter(Groups.startsWith("xiaoguai:hotl:", 0, 100), {String g: g.substring(15)}))` |

This maps Okta groups named `xiaoguai:hotl:<scope>` (e.g. `xiaoguai:hotl:decide`) to the `scopes` claim. Create the groups, assign operator users to them, and the next token carries the values.

#### Azure AD / Entra ID

Azure does not natively emit `scopes` from groups. Use a Conditional Access app role mapping:

1. App registrations → xiaoguai → App roles → add `hotl-decide`, `hotl-policy-read`, `hotl-policy-write`.
2. Enterprise applications → xiaoguai → Users and groups → assign operators to the `hotl-decide` role.
3. Add a [Token Configuration](https://learn.microsoft.com/en-us/azure/active-directory/develop/optional-claims) optional claim of type `Optional Claim` named `scopes`, source `Application`, expression `roles` (the `roles` claim that Azure emits is then read by xiaoguai-api's claim parser as the `scopes` source — see `xiaoguai-auth::claims::parse_scopes` which accepts either `scopes` or `roles`).

#### Authelia (self-hosted lightweight)

In `configuration.yml`:

```yaml
identity_providers:
  oidc:
    clients:
      - id: xiaoguai
        scopes: [openid, profile, email, hotl:decide, hotl:policy:read, hotl:policy:write]
        grant_types: [authorization_code, refresh_token]
```

Per-user scope assignment lives in `users_database.yml` under `groups:`; xiaoguai-api accepts `groups` as a fallback source when `scopes` is absent.

#### Confirm the gauge

After the issuer rollout:

```bash
# Should rise to ≥0.95 within 10 minutes of operator logins:
curl -s "$XIAOGUAI_URL/metrics" | grep xiaoguai_oidc_scopes_claim_present
# xiaoguai_oidc_scopes_claim_present{issuer="https://idp.example.com"} 1
```

If it stays at 0, the issuer is emitting tokens but the parser is not finding the claim — check `parse_scopes` log lines at boot for the field-name it looked for, and add the field-name to the issuer's claim configuration.

---

## 10. Scheduler operations

### 10.1 Inspect jobs

```bash
psql "$DATABASE_URL" -c "SELECT id, trigger->>'type' AS trig, enabled FROM scheduled_jobs;"
```

### 10.2 Fire a job now (for testing)

```bash
curl -X POST -H "Authorization: Bearer $ADMIN_JWT" \
  http://xiaoguai/v1/admin/scheduler/jobs/<id>/fire-now
```

### 10.3 Stuck job

```bash
psql "$DATABASE_URL" -c \
  "SELECT id, job_id, started_at FROM scheduled_job_runs \
   WHERE status='running' AND started_at < now() - interval '30 minutes';"
# Cancel the session it's tied to:
curl -X POST http://xiaoguai/v1/sessions/<sess>/cancel -H "Authorization: Bearer $OPERATOR_JWT"
```

### 10.4 Webhook 404 / 401

- `404`: `route_id` mismatch or job disabled. Verify with the SQL in 10.1.
- `401`: missing or wrong `X-Xiaoguai-Token`. Rotate via `DELETE /v1/admin/scheduler/tokens/<id>` + `POST /v1/admin/scheduler/tokens`.

### 10.5 Runaway proactive

In-memory budget ledger; restart pods to clear, then disable the job:

```bash
kubectl -n xiaoguai rollout restart deploy/xiaoguai
psql "$DATABASE_URL" -c "UPDATE scheduled_jobs SET enabled=false WHERE id='<id>';"
```

---

## 11. Sandbox operations (`xiaoguai-mcp-exec`)

### 11.1 Register

```bash
xiaoguai mcp register \
  --id exec-py-sandbox \
  --transport stdio \
  --command xiaoguai-mcp-exec \
  --args '--timeout-secs 30 --memory-mb 512'
```

### 11.2 Verify isolation

```bash
# Time-bound
xiaoguai chat --prompt "Run python: while True: pass"
# Expect timed_out=true within deadline

# Env scrub
XIAOGUAI_AUDIT_SIGNING_KEY=should-not-leak \
  xiaoguai chat --prompt "Run python: import os; print(os.environ.get('XIAOGUAI_AUDIT_SIGNING_KEY','MISSING'))"
# Expect: MISSING
```

### 11.3 Tune

| Symptom | Action |
|---|---|
| Numpy/pandas snippets get killed by ulimit | Raise `--memory-mb` to 1024 or 2048 |
| Snippets need to run longer for legitimate jobs | Raise `--timeout-secs` (clamp ≤ 300) |
| Stderr scrubbing too aggressive for debugging | Set `--no-redact-stderr` temporarily; revert after |
| Tempdir cleanup slow on networked fs | Point `--workdir-parent` at tmpfs |

### 11.4 Network egress

NOT enforced at the sandbox. Use k8s NetworkPolicy or `--network none` at the container layer.

---

## 12. IM gateway operations

| Task | Procedure |
|---|---|
| Add a new Feishu app | Register app, copy app_id + app_secret, update `xiaoguai-im` Secret, `helm upgrade` |
| Rotate a webhook secret | Update Secret; restart pods (no in-flight verifications break because secrets are loaded per request) |
| Bind a user's IM identity | User triggers a bind challenge from chat; they paste the bind code into the IM platform |
| Inspect failed deliveries | `SELECT * FROM audit_log WHERE action='im.reply_failed' ORDER BY ts DESC LIMIT 50` |

---

## 13. Troubleshooting

| Symptom | Diagnosis | Resolution |
|---|---|---|
| `/v1/memories` returns 503 | Memory store failed at boot | Check boot logs for `memory: selected embedding backend`; if absent, build did not complete. Verify Ollama reachable. |
| Ollama embedding calls 4xx | Model not pulled | `ollama pull all-minilm` |
| Recall quality poor with `OLLAMA_HOST` unset | InMemoryEmbedder is a hash, not semantic | Set `OLLAMA_HOST`. |
| Postgres rejects vector insert | Dimension mismatch | Use `all-minilm` (384) or migrate the column. |
| PII still appears in audit_log | Redaction off OR pattern out of scope | Check `XIAOGUAI_AUDIT_REDACT_PII`; supported patterns: emails, IPv4, `Bearer …`, AWS keys. |
| `cache.url` empty but the server crashes on boot | Build cut to require Valkey; should not happen on `≥ v1.4.0`. | File an issue. |
| Audit `verify` returns `valid=false` | Tampering OR rotation window expired | Diff `pg_dump` vs last snapshot; check rotation status; investigate before resolving. |
| File-watch on macOS misses events | Symlink path | Use canonical path: `python3 -c "import os; print(os.path.realpath('/var/notes'))"` |
| IM webhook returning 401 | Signature mismatch | Verify clock skew (NTP), confirm secret rotation propagated, check `Authorization` header. |
| First-token latency degraded | Provider cascade hitting slow fallback | Inspect `token_usage` rows; remove the slow provider from `fallback_order`. |

---

## 14. On-call escalation

| Severity | Definition | Response | Escalation |
|---|---|---|---|
| Sev-1 | Service down for any tenant | Page on-call within 5 min | Engineering lead + product within 30 min |
| Sev-2 | Audit chain broken; auth bypass; data leak | Page within 5 min | Security + engineering lead immediately |
| Sev-3 | Degraded performance; partial feature outage | Acknowledge within 30 min | Engineering on-call rotation |
| Sev-4 | Minor or cosmetic | Next-business-day | None |

---

## 15. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema:
  name: testany-traceability
  version: "1.0.0"
  profile: prd-profile-v1
artifact:
  id: RUNBOOK-XIAOGUAI-001
  type: RUNBOOK
  title: Xiaoguai operator runbook
  status: draft
  owners: [ops.xiaoguai, engineering.xiaoguai]
  created_at: 2026-05-28
  updated_at: 2026-05-28
  source_documents:
    - HLD-XIAOGUAI-001
    - API-XIAOGUAI-001
    - GUARDRAILS-XIAOGUAI-001
entities:
  requirements: []
  risks: []
  must_not_regress: []
  external_behaviors: []
  decisions: []
  flows: []
  test_cases: []
relations:
  - { id: REL-RUN-001, type: refines, from: RUNBOOK-XIAOGUAI-001, to: HLD-XIAOGUAI-001, status: active }
  - { id: REL-RUN-002, type: refines, from: RUNBOOK-XIAOGUAI-001, to: GUARDRAILS-XIAOGUAI-001, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
