# Xiaoguai 设计文档评审与交付前缺口清单

- 评审日期：2026-05-28
- 评审对象：`/Users/zw/testany/myskills/xiaoguai-agent-design/docs/`
- 对照实现：`/Users/zw/testany/myskills/xiaoguai/`
- 评审范围：只做 review，不改代码，不改现有设计文档

## 1. 总结结论

这套设计文档已经具备比较完整的“产品 + 架构 + API + 测试 + 运维”骨架，明显不是草稿级材料，优点是：

- 有较完整的主文档链路：`PRD`、`HLD`、`API Contract`、`Guardrails`、`Runbook`、`Test Strategy`、`Test Spec`、多份 `LLD`。
- 采用 retrofit 写法，明确说明文档是在实现之后回补，这一点是诚实且有助于后续追踪的。
- 文档里大量引用实现仓库和 ADR，具备追溯意识。

但当前最大问题不是“有没有文档”，而是“文档声明的 v1.4 状态过于乐观，和当前代码存在明显漂移”。尤其在 memory、tasks、workspaces、toolchain、默认端口、skill pack 签名、LLM/provider 管理、测试覆盖状态这些主题上，存在：

- 已在文档中写成 shipped，但实现仍未完全对外暴露。
- 已在实现中前进，但前端/设计文档还保留旧说明。
- 文档声称是 single source of truth，但与当前 router / CLI / config 默认值不一致。

结论：

1. 文档框架基本齐全，但还不算“准确可交付”。
2. 现在最需要的是一次“状态校准”，把文档改成真实 reflecting current code，而不是继续增加文档数量。
3. 交付前必须补一轮“实现闭环清单”，否则会出现销售/交付/研发/测试各说各话。

## 2. 设计文档是否齐全

### 2.1 已具备的文档类别

当前 `docs/` 下已具备：

- PRD：`prd-xiaoguai.md`
- HLD：`hld.md`
- API 合同：`api-contract.md`
- 项目 guardrails：`guardrails.md`
- 测试策略：`test-strategy.md`
- 测试规格：`test-spec.md`
- 运维 runbook：`runbook.md`
- 一组 LLD：`lld-storage`、`lld-audit`、`lld-llm`、`lld-mcp`、`lld-mcp-exec`、`lld-agent`、`lld-runtime`、`lld-orchestrator`、`lld-memory`、`lld-rag`、`lld-im-gateway`、`lld-personas`

### 2.2 仍然缺少或不够完整的设计文档

以下主题在代码或 PRD/HLD 中已经是重要能力，但没有对应的独立设计文档，或者粒度明显不够：

1. `xiaoguai-api` 没有独立 LLD。
说明：当前 API 是对外主入口，承载 sessions、scheduler、skills、memory、workspaces、admin 等多个域，只有 API Contract 不够，缺少 handler/state/auth/rate-limit/error mapping 的实现级设计。

2. `xiaoguai-cli` 没有独立 LLD。
说明：CLI 已经是统一入口之一，且承担大量 operator 操作，但没有命令组织、远端/本地模式、兼容策略的设计文档。

3. `xiaoguai-scheduler` / `xiaoguai-watch` / `xiaoguai-anomaly` 没有独立 LLD。
说明：这些已经是主产品能力，不应只停留在 PRD/HLD 描述层。

4. `xiaoguai-tasks` 没有独立 LLD。
说明：PRD/HLD 把 Kanban / tasks 作为 v1.4 范围内 shipped 能力，但没有与其重要性匹配的设计文档。

5. workspaces 没有独立设计文档。
说明：代码里已有 `/v1/workspaces` 路由和 bridge，但设计文档只在 PRD/HLD 顺带提到。

6. Personas 与 sessions/workspaces/tasks 的交互边界没有完整设计。
说明：有 `lld-personas.md`，但它更像 repository 视角，没有覆盖 persona 如何在 chat、orchestrator、memory、tasks 中贯穿。

7. 前端设计缺位。
说明：chat-ui/admin-ui 已经承担真实产品职责，但没有单独的 UI/UX contract、状态管理约束、fallback 策略、兼容策略文档。

8. 交付/发布设计缺位。
说明：有 runbook，但缺“发布状态定义”：什么叫 alpha / beta / GA，什么能力能写 shipped，什么只能写 experimental。

## 3. 文档不准确或与实现不一致的问题

以下问题优先级较高，因为它们会直接误导研发、测试、交付或用户预期。

### 3.1 PRD/HLD 把 tasks 写成 shipped，但 API 还未对外完整暴露

证据：

- PRD 把 `xiaoguai-tasks` 写成 v1.4 shipped：`docs/prd-xiaoguai.md:43-45`
- HLD module map 也把 `xiaoguai-tasks` 放进正式模块：`docs/hld.md:84-86`
- 但当前 API router 里没有 `/v1/tasks*` 路由：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-api/src/routes/mod.rs:37-146`
- 前端 Kanban 页自己也写了 “404 fallback: if /v1/tasks/* returns 404 (kanban not yet in backend)”：`/Users/zw/testany/myskills/xiaoguai/frontend/admin-ui/src/panes/Kanban.tsx:12-14`

判断：

- `xiaoguai-tasks` crate 本身已经存在，但“对用户可用的 product surface”还没有闭环。
- 因此当前文档不应写成“shipped”，最多应写成“crate landed, API surface incomplete”或“backend partially landed”。

### 3.2 Memory 已经实现，但前端和部分文档仍保留“未 shipped”说法

证据：

- PRD 把 memory 写成 shipped：`docs/prd-xiaoguai.md:41`
- HLD 也把 `xiaoguai-memory` 列为正式 crate：`docs/hld.md:73`
- API router 已挂 `/v1/memories`：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-api/src/routes/mod.rs:119-137`
- `AppState` 已有 `memory_store`：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-api/src/state.rs:206-213`
- `memory_bridge` 已经真正接入 `PgMemoryStore` + `Ollama/InMemory` embedder 选择：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-core/src/memory_bridge.rs:1-55`
- 但前端 shared 仍写 “The xiaoguai-memory Rust crate is not yet shipped”：`/Users/zw/testany/myskills/xiaoguai/frontend/shared/src/memory.ts:2-7`

判断：

- 这里是反向漂移：实现已经前进，配套文档/前端注释落后。
- 如果不清理，会导致团队内部对 feature readiness 认知混乱。

### 3.3 API Contract 自称 single source of truth，但与实现不一致

证据：

- API Contract 明确写自己是 single source of truth：`docs/api-contract.md:12`
- 但它写默认监听地址 `0.0.0.0:7600`：`docs/api-contract.md:20`
- 同时 CLI 绝大多数默认 base URL 仍是 `http://localhost:8080`：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-cli/src/main.rs:76,155,175,194,214,233,246`
- quickstart 也仍然以 `8080` 为主：`/Users/zw/testany/myskills/xiaoguai/docs/user-guide/quickstart.md:16-79`
- config example 又使用 `7600`：`/Users/zw/testany/myskills/xiaoguai/deploy/config.example.yaml:13-16`

判断：

- 当前默认端口存在双轨现实：配置默认 7600，用户文档和 CLI 默认 8080。
- API Contract 不能继续声称 single source of truth，除非先完成统一。

### 3.4 PRD 中多个 API/能力写成存在，但实现里没有对应对外接口或仍为局部能力

典型例子：

1. `REQ-LLM-001` 写 `POST /v1/admin/llm/providers`：`docs/prd-xiaoguai.md:141-145`
   - 当前 router 中没有这组路由：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-api/src/routes/mod.rs:37-146`

2. `REQ-MCP-005` 写 `POST /v1/mcp/serve`：`docs/prd-xiaoguai.md:151-155`
   - 实现里是“当 enabled 时挂一个 MCP router”，并非标准 admin-style REST mutation 语义：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-api/src/routes/mod.rs:219-238`

3. IM webhook 路径在 API Contract 中写 `/v1/im/<platform>/event`：`docs/api-contract.md:29`
   - 当前 `xiaoguai-api` router 没体现这组 routes，IM 更偏独立 gateway/其他 wiring。

判断：

- PRD 和 API Contract 部分内容仍带“目标态”语气，不完全是“现状”。
- retrofit 文档需要显式区分 `implemented / partially implemented / planned but not exposed`。

### 3.5 Guardrails 中“必须通过”的 CI/规则与当前实现状态不一定一致

证据：

- Guardrails 规定 `cargo fmt --check` 和 `cargo clippy --all-targets --all-features -- -D warnings` 必须 clean：`docs/guardrails.md:33-39`
- 但仓库历史 handoff 多次提到现实中会用 `--ignore-rust-version`、有 advisory/gated job、以及一部分流程依赖 label/paid runner。
- Guardrails 还写 `cargo test --workspace` MUST pass：`docs/guardrails.md:37`
- 测试策略和 runbook 又大量描述 ignored/gated/nightly/real-dependency 测试：`docs/test-strategy.md:52-60, 192-198`

判断：

- Guardrails 是“理想规范”，但没有明确区分哪些已强制执行、哪些还是 aspirational。
- 这会让评审者误以为现在所有 gate 都是完全硬门槛。

### 3.6 HLD/PRD 对 skill-pack signed install 的描述，落地证据不足

证据：

- PRD 把 signed pack install 写成 P0：`docs/prd-xiaoguai.md:210-213`
- HLD DEC-007 也明确声明 pack local-first and signed：`docs/hld.md:152-158`
- 当前 router 有 `GET/POST/DELETE /v1/skills/*`：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-api/src/routes/mod.rs:111-118`
- 但现有 skills 路由只从代码层面能确认 catalog/install/uninstall surface，未从这次 review 中看到和签名校验强绑定的实现证据。

判断：

- 需要补一次“签名校验已落地”的实现追溯；若没有，应把文档降级为 planned / partial。

### 3.7 Personas 文档粒度不足，且与实际实现形态不完全一致

证据：

- HLD 里 personas 被定义为 `system_prompt + default_tools + default_skills`：`docs/hld.md:188-192`
- LLD 里人格还涉及 session attachment、tool allowlist、memory filtering：`docs/lld/lld-personas.md:12-54`
- 实现里 `xiaoguai-personas` 还包含 session-persona 绑定与查询：`/Users/zw/testany/myskills/xiaoguai/crates/xiaoguai-personas/src/traits.rs`、`src/pg.rs`

判断：

- 现有 HLD 对 personas 的描述过轻，无法支撑它作为 cross-cutting capability 的设计理解。

### 3.8 Test Strategy / Test Spec 的覆盖率说法偏乐观

证据：

- Test Strategy 要求 P0 coverage 100%：`docs/test-strategy.md:25-26, 97-100`
- Test Spec 直接给出“P0 authored 33/33, coverage 100%”：`docs/test-spec.md:23-29`
- 但从代码现状看，仍存在 tasks surface 未暴露、frontend fallback 保留、多个文档和接口状态不一致。

判断：

- 文档里的“coverage 100%”更像目标表，不像经过真实实现核对后的可信声明。
- 建议改成“documented planned coverage”或“validated coverage as of commit <sha>”。

## 4. 项目当前存在的问题列表

### P0 / 高优先级问题

1. 文档状态与实现状态漂移，容易误导交付和测试。
2. `tasks` 已有 crate/前端/CLI 预埋，但对外 API 仍未闭环。
3. API Contract、CLI 默认值、quickstart、config example 在默认端口上不统一。
4. `AppState` 仍然持续膨胀，新增能力每次都扩到 router/core/tests，多分支并行时冲突风险高。
5. 文档里多处“single source of truth”或“shipped”表述过强，但项目仍有明显 partial 状态。

### P1 / 中优先级问题

6. 缺少 `api/cli/scheduler/watch/anomaly/tasks/workspaces` 这些主题的独立设计文档。
7. Personas、orchestrator、memory 之间的跨模块约束没有被一份文档说清楚。
8. 测试策略与真实 CI 执行状态之间缺少“硬门槛 vs 目标态”标识。
9. 前端注释、shared client、runbook、设计文档的版本同步机制不足。
10. Skill pack 的签名/激活/热加载状态需要更明确的真值定义。

### P2 / 结构性问题

11. 文档中“retrofit 现状”和“下一步 roadmap”仍有混写，阅读者不容易分辨。
12. 设计文档大量引用 PR/ADR/任务号，但缺少固定到 commit/tag 的状态基线。
13. 运维 runbook 假定的命令、端口、配置字段，与实际 settings 字段命名仍存在错位风险。

## 5. 修改意见

### 5.1 文档组织层面的修改意见

1. 在 `docs/` 根目录增加一份 `status-matrix.md`。
内容建议：逐项列出 capability、crate landed、API exposed、CLI exposed、UI exposed、tests passing、doc synced、release ready。

2. 为每份 retrofit 文档增加统一的状态标签：
- `implemented`
- `partially_implemented`
- `planned`
- `doc_outdated`

3. PRD/HLD 中所有 `✅ shipped` 项都应有一个“验收锚点”：
- 路由存在
- CLI/前端存在
- smoke/e2e/test 证据存在
- 对应 commit/tag 可追踪

4. 单独增加以下设计文档：
- `lld-api.md`
- `lld-cli.md`
- `lld-scheduler.md`
- `lld-watch.md`
- `lld-anomaly.md`
- `lld-tasks.md`
- `lld-workspaces.md`
- `frontend-architecture.md`

5. 增加一份 `release-readiness-definition.md`。
明确什么条件下文档可以写 `shipped`，什么只能写 `preview/experimental/partial`。

### 5.2 准确性修订意见

1. 修正 PRD/HLD 中关于 `tasks` 的 shipped 结论，改成 partial 或 backend-incomplete，直到 `/v1/tasks*` 真正上线。
2. 修正前端 shared/admin 对 memory 的旧注释，避免继续写“未 shipped”。
3. 统一默认端口：至少选一个作为 canonical default，并同步 API Contract、CLI 默认值、quickstart、runbook、config example。
4. 对 `LLM provider admin API`、`IM webhook route`、`mcp serve` 这类目标态描述，标注真实状态而不是直接写成当前可用。
5. 在 Guardrails / Test Strategy 中区分：
- current enforced
- current gated/advisory
- target policy

### 5.3 研发结构层面的修改意见

1. 给 `tasks` 做完整 API 暴露与端到端连通，避免“有 crate 没入口”的半成品状态长期存在。
2. 给 `AppState` 做分层拆分，降低后续功能并行开发冲突。
3. 建立“文档同步检查表”，每次新增 crate/route/CLI command 时必须同步 PRD/HLD/API/Runbook/Frontend 注释。
4. 为 capability 状态引入统一真值源，可由代码生成文档片段，减少手工漂移。

## 6. 交付之前还要开发的内容列表

以下列表按“影响交付可信度”的优先级排列。

### 6.1 必须完成（建议作为交付前 blocking items）

1. 完成 `/v1/tasks*` 后端 API 暴露。
当前已有 `xiaoguai-tasks` crate、CLI、前端 mock/fallback，但没有形成真正的对外可用产品面。

2. 统一默认端口与默认接入方式。
至少明确 7600 和 8080 哪个是 canonical，并同步 CLI、quickstart、config、runbook、API Contract。

3. 完成 shipped/partial 状态重审。
把 PRD/HLD/Test Spec 里所有 `shipped` 声明逐项过一遍，按真实实现修正。

4. 完成 memory / workspaces / tasks / personas 的联调闭环验证。
不只是 crate 在，还要验证 API、CLI、UI、文档是否一致。

5. 补 `lld-tasks.md` 与 `lld-api.md`。
这是目前最明显的设计缺口。

### 6.2 强烈建议完成

6. 增加 capability status matrix 文档，作为交付真值表。
7. 明确 skill pack 签名校验、激活、热加载当前实际状态，并补实现或修正文档。
8. 给 orchestrator + personas + memory 增加一份跨模块交互设计。
9. 区分 test coverage 的“已实测”和“计划覆盖”，避免对外过度承诺。
10. 为 runbook 的所有关键命令做一次实机校验，尤其是端口、二进制名、路径、配置字段名。

### 6.3 交付后尽快补齐

11. scheduler/watch/anomaly 独立 LLD。
12. frontend architecture / state model 文档。
13. 统一发布/交付状态命名体系（preview/beta/ga/experimental）。
14. 文档对齐自动化检查或生成机制。

## 7. 建议的下一步执行顺序

1. 先做 capability 状态盘点。
2. 修正文档中的状态、默认值、路由真值。
3. 把 `tasks` API 真正补齐。
4. 补缺失 LLD。
5. 再做一次交付前 review，确认文档与实现已经收敛。

## 8. 最终判断

- 文档整体框架：合格，甚至偏强。
- 文档准确性：当前不够，存在明显状态漂移。
- 是否适合直接作为交付设计包：暂时不建议直接交付。
- 主要原因：不是缺文档，而是“文档写得比代码成熟度更像最终态”。

如果目标是对外交付，这套文档最需要的不是继续扩写，而是一次严格的“与当前代码逐项对账”。
