# LLD — `xiaoguai-llm`

| | |
|---|---|
| Document ID | `LLD-LLM-001` |
| Refines | (`xiaoguai-llm` has no single HLD decision; refines REQ-LLM-001…005) |
| Verifies | `REQ-LLM-001` … `REQ-LLM-005` |

## 1. Module purpose

`xiaoguai-llm` owns:

- The `LlmProvider` trait, implemented by 8 backends.
- The `LlmRouter` which resolves which provider to call for a given `(tenant_id, model_alias)` and cascades through `fallback_order` on failure.
- The `UsageSink` hook that emits a `token_usage` row per completed stream.
- Streaming response envelope `LlmStream` (tokens, tool calls, finish reason).

## 2. Public interface

```rust
#[async_trait]
pub trait LlmProvider: Send + Sync {
    async fn chat(&self, req: ChatRequest) -> Result<LlmStream, LlmError>;
    fn capabilities(&self) -> Capabilities;
}

pub enum ProviderKind {
    Ollama, OpenAiCompat, Anthropic, Gemini, Bedrock, AzureOpenAi, Mistral, Groq,
}

pub struct LlmRouter {
    storage: Storage,
    providers: dashmap::DashMap<ProviderId, Arc<dyn LlmProvider>>,
    usage_sink: Arc<dyn UsageSink>,
}

impl LlmRouter {
    pub async fn chat(&self, tenant_id: TenantId, model_alias: &str, req: ChatRequest)
        -> Result<LlmStream, LlmError>;
    pub async fn register(&self, row: LlmProviderRow) -> Result<(), LlmError>;
}

pub trait UsageSink: Send + Sync {
    fn record(&self, row: TokenUsageRow);
}
```

## 3. Module structure

```
crates/xiaoguai-llm/
├── src/
│   ├── lib.rs              # LlmRouter, traits
│   ├── stream.rs           # LlmStream envelope + tool-call deserialization
│   ├── usage.rs            # UsageSink trait + StorageUsageSink impl
│   ├── token_count.rs      # NEW v0.5.4.1 — conservative token estimator
│   ├── error.rs
│   └── backends/
│       ├── ollama.rs
│       ├── openai_compat.rs
│       ├── anthropic.rs
│       ├── gemini.rs
│       ├── bedrock.rs       # uses aws-sdk + SigV4
│       ├── azure_openai.rs
│       ├── mistral.rs
│       └── groq.rs
└── tests/
    ├── router_fallback.rs
    └── usage_emission.rs
```

## 4. Key flows

### 4.1 Chat with fallback

```
LlmRouter::chat(tenant, alias, req)
   ├─ resolve provider chain for (tenant, alias)
   │     primary = provider where default_for_models contains alias
   │     fallbacks = primary.fallback_order
   ├─ for provider in [primary, fallbacks...]:
   │     try provider.chat(req)
   │     on stream: forward tokens; on close: usage_sink.record(row); return
   │     on error 5xx / timeout: log + continue
   └─ on all failed: return LlmError::AllProvidersFailed
```

### 4.2 Usage recording

`UsageSink::record` is called on stream close. The implementation `StorageUsageSink` opens a transaction, inserts `token_usage`, audits `llm.usage` only if cost > 0.

### 4.3 Token estimation (`token_count` module, v0.5.4.1)

Pure, dependency-free estimator used by `xiaoguai-agent::history::compact` to decide whether to trigger summarisation. **Conservative on purpose** — rounds up so callers err on the side of compacting earlier rather than later.

Public surface (re-exported from `lib.rs`):

```rust
pub fn estimate_tokens(s: &str) -> usize;
pub fn estimate_message_tokens(messages: &[Message]) -> usize;
```

| Property | Value |
|---|---|
| Heuristic | 4 chars/token (long-standing GPT-3 rule of thumb) |
| Char vs byte | Uses `chars().count()` — CJK is NOT double-counted |
| Empty string | Returns 0 (no overhead) |
| Per-message overhead | 4 tokens (OpenAI-documented for `gpt-3.5-turbo`; conservative upper bound across providers) |
| Tool calls | Summed: `name` + `arguments_json` strings |
| Tool replies | Adds `tool_call_id` length |
| Saturation | `saturating_add` throughout — no overflow panic on giant inputs |

**Why not tiktoken**: heavy native dep, licensing complexity, model-specific tokenisers diverge anyway. The estimator only needs to be accurate-enough to drive a trigger threshold (typically 75 % of context), not exact. Both directions (code denser, whitespace sparser) fit within that safety margin.

Tests: 8 unit tests covering empty string, sub-token rounds-up boundary, CJK width, per-message overhead, tool-call aggregation, tool-role id overhead, multi-message sum, realistic 200-char paragraph.

### 4.4 MiniMax backend with reasoning-content passthrough (v0.5.4.3, DEC-024, PR #79)

`crates/xiaoguai-llm/src/minimax.rs` adds the 9th provider backend. Mirrors `GroqBackend`'s pattern (clean OpenAI-compatible reference) but explicitly **not** a wrapper around `OpenAiCompatBackend` — per DEC-024 there are three reasons that route would silently drop critical functionality:

```rust
pub struct MinimaxBackend {
    api_key: String,
    base_url: String,           // default "https://api.minimax.io/v1"
    http: reqwest::Client,
}

impl MinimaxBackend {
    pub fn new(api_key: impl Into<String>) -> Self;
    pub fn with_base_url(self, url: impl Into<String>) -> Self;
}

impl LlmBackend for MinimaxBackend {
    fn name(&self) -> &'static str { "minimax" }
    async fn chat_stream(&self, req: ChatRequest) -> Result<ChatStream, LlmError>;
}
```

Endpoint: `POST {base_url}/chat/completions` (OpenAI-compatible). Auth: `Authorization: Bearer {api_key}`.

**Reasoning content passthrough** — the M1/M2/M2.5/M2.7 thinking-mode models emit `reasoning_content` in their streaming deltas alongside `content`. We surface this via a new `ChatChunk.reasoning_delta: Option<String>` field added in `crates/xiaoguai-llm/src/types.rs`. The other 8 backends leave it `None` (graceful degrade — no breaking change to their behaviour). Token attribution: `xiaoguai_llm_reasoning_tokens_total{provider, model}` counter increments per chunk so operators can track thinking-mode cost separately from output cost.

Seed migration `0023_minimax_provider_seed.sql` adds one `llm_providers` row with `kind='minimax'`, `enabled=false`, `models=['MiniMax-M1', 'MiniMax-M2', 'MiniMax-M2.5', 'MiniMax-M2.7', 'abab6.5-chat']`. Default-disabled means no surprise outbound traffic.

Tests: 5 mockito-backed integration tests covering content passthrough, reasoning_delta capture, abab6.5-chat null-reasoning path (model doesn't emit thinking), 401 unauthorized handling, and the `xiaoguai_llm_reasoning_tokens_total` counter increment.

## 5. Error handling

```rust
pub enum LlmError {
    Provider(ProviderKind, String),
    Timeout,
    InvalidModel(String),
    AllProvidersFailed(Vec<(ProviderKind, String)>),
    Capability(&'static str),     // e.g., tool calls not supported on this provider
    Cancelled,
}
```

`Cancelled` is mapped to SSE `final { stop_reason: "cancelled" }` upstream.

## 6. Concurrency / transactions

- Each backend uses a tokio `reqwest::Client` with HTTP/2 keep-alive.
- `LlmRouter` itself is `Send + Sync` and cheap to clone (`Arc` internally).
- `UsageSink::record` is fire-and-forget for in-flight stream, but completion is required before the SSE `final` event is sent (so usage is consistent with the user-visible response).

## 7. Test design

| Layer | Cases |
|---|---|
| Unit | Backend request shape per provider; OpenAI-compat tool-call parsing; UsageSink row mapping. `token_count` (v0.5.4.1): empty string, sub-token round-up boundary, CJK width, per-message overhead, tool-call aggregation, tool-role id overhead, many-message sum, realistic-paragraph. |
| Integration | `tests/router_fallback.rs` — wiremock returns 503 from primary, secondary succeeds; usage row appears once; `tests/usage_emission.rs` — exact token count |
| System | Test strategy L2 E2E: `/v1/sessions/{id}/messages` produces matching `/v1/usage` row |

## 8. Traceability metadata

<!-- TRACEABILITY-METADATA:BEGIN -->
```yaml
schema: { name: testany-traceability, version: "1.0.0", profile: lld-profile-v1 }
artifact:
  id: LLD-LLM-001
  type: LLD
  title: xiaoguai-llm LLD
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
    - { id: DEC-LLD-LLM-001, title: "Provider trait with capabilities", statement: "LlmProvider exposes capabilities() so router can skip incompatible models (e.g., tool calls).", status: approved, scope: in, decision: "Trait + capabilities introspection.", rationale: "Mistral and older Ollama models lack tool-call support." }
    - { id: DEC-LLD-LLM-002, title: "Fail-cascade fallback chain", statement: "Router cascades through fallback_order on 5xx/timeout.", status: approved, scope: in, decision: "Linear cascade.", rationale: "Avoids parallel fan-out cost; simple operator mental model." }
  flows:
    - { id: FLOW-LLD-LLM-001, title: "Chat with fallback", statement: "Primary then fallbacks; first successful stream wins.", status: approved, scope: in, kind: module_interaction }
  test_cases: []
relations:
  - { id: REL-LLD-LLM-001, type: refines, from: DEC-LLD-LLM-002, to: REQ-LLM-003, status: active }
waivers: []
```
<!-- TRACEABILITY-METADATA:END -->
