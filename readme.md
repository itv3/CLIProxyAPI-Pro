# CLIProxyAPI Pro 需求与开发说明

## 1. 项目定位

CLIProxyAPI Pro 基于两个开源项目开发：

- [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)：负责模型请求、协议转换、账号调度、OAuth、API Key 和统一下游 Key。
- [CPA-Manager-Plus](https://github.com/seakee/CPA-Manager-Plus)：负责管理页面、账号管理、统计、成本、监控和运维。

开发工作区初始使用两个官方仓库作为 Git 子模块；建立自有 fork 后替换子模块地址，并继续保留官方仓库作为上游更新来源。组件版本以主仓库记录的子模块 SHA 为唯一权威：Docker 镜像标签由 CI 从对应 SHA 构建生成，只作为发布产物标识，不构成第二套版本来源。

整体结构：

```text
CLIProxyAPI Pro
= CPA-Manager-Plus 主要改造
+ CLIProxyAPI 有界运行时改造
+ 统一安装、测试和发布
```

[sub2apiplus](https://github.com/itv3/sub2apiplus) 仅作为账号交互、协议探测、连通性测试和官方客户端兼容的参考，不作为运行依赖。

## 2. 项目结构

CLIProxyAPI Pro 对外是一个产品，内部保留两个独立组件：

```text
客户端和管理员
       |
       v
统一域名和反向代理
       |
       +-- 模型请求 --> CLIProxyAPI
       |
       +-- 管理页面 --> CPA-Manager-Plus
                          |
                          +-- Management API --> CLIProxyAPI
```

反向代理是统一入口。代理选型、TLS 终止和完整配置由部署设计确定，本节只锁定路由与透传契约：

| 路径 | 目标组件 | 说明 |
|---|---|---|
| 各协议标准模型地址（`/v1/*`、`/v1beta/*` 等） | CLIProxyAPI | 支持 SSE 流式透传和 WebSocket 长连接，不缓冲流式响应体 |
| `/v0/management/*` | CPA-Manager-Plus | 按现有行为由 Manager 透传到 CLIProxyAPI |
| `/v0/pro/*` | CPA-Manager-Plus | Manager 本地处理；未匹配路由返回 `404`，不转发到 CLIProxyAPI |
| 管理页面及其静态资源 | CPA-Manager-Plus | 由 Manager 提供页面和管理接口 |

普通模型请求流量不经过 CPA-Manager-Plus；Manager 停止服务时，反向代理和 CLIProxyAPI 仍应能够独立处理模型请求。

开发原则：

1. 分别维护 CLIProxyAPI fork 和 CPA-Manager-Plus fork。
2. 使用一个 CLIProxyAPI Pro 主项目锁定两个组件版本。
3. 使用 Docker Compose 一键安装，两个组件保持独立容器和数据目录。
4. 不把两个项目合并成一个 Go 工程、二进制或进程。
5. Pro 功能尽量通过新增模块和少量接入点实现，方便继续合并上游更新。
6. Manager 的 SQLite 与 Gateway 的配置和认证文件之间不存在跨组件事务；跨组件写操作按最终一致设计，一致性由 3.1 定义的持久化草稿状态机、幂等键和补偿动作保证。

### 本地参考源码

当前开发工作区在 `.references/sub2apiplus` 保存 [itv3/sub2apiplus](https://github.com/itv3/sub2apiplus) 的只读浅克隆，并将 [Wei-Shaw/sub2api](https://github.com/Wei-Shaw/sub2api) 配置为其上游参考地址。该目录用于页面布局、交互流程、协议探测和兼容逻辑的阅读、检索与对比，方便开发人员和 AI 在同一工作区内分析参考实现。

`.references/` 是本机开发辅助目录，不是 CLIProxyAPI Pro 的正式组件、Git 子模块、构建依赖、运行依赖或发布内容。主仓库通过 `.git/info/exclude` 的 `/.references/` 规则在本地忽略整个目录，其中的说明文件、独立 Git 仓库和源码均不进入 CLIProxyAPI Pro 提交，也不参与 CI、构建、打包或部署。

参考源码只能读取，禁止在其中实现 Pro 功能、安装依赖、执行构建或产生提交。需要更新时只能在参考仓库中执行 `git pull --ff-only`。所有正式实现必须写入 `components/cpa-manager-plus` 或 `components/cliproxyapi`。

## 3. 功能需求

### 3.1 统一账号管理

#### 主要解决的问题

CLIProxyAPI 原生按 API 配置、OAuth 认证文件和插件认证分别管理凭证，CPA-Manager-Plus 原有页面也按 Provider、认证文件和 OAuth 流程拆分。Pro 统一账号管理在不替换 Gateway 凭证与调度机制的前提下，为这些底层来源增加统一、稳定的管理视图，主要解决：

1. 同一部署的账号分散在多个页面，管理员需要先理解 Gateway 内部 Provider 和配置结构才能完成日常操作。
2. OAuth、API Key、Vertex 和插件账号的数据结构、状态字段及操作接口不同，前端难以统一展示和批量管理。
3. `auth_index` 会随 Key、Base URL、文件路径或协议变化，直接使用它作为账号身份会导致编辑迁移后账号、历史用量和操作记录断裂。
4. 添加、协议探测、模型配置、测试和启用涉及多次 Management API 调用，中途失败可能产生半完成配置或短暂进入正常调度。
5. 模型白名单、映射、健康状态、官方配额和本地用量来自不同接口，缺少以账号为中心的一致结果。
6. 存量配置、绑定漂移、批量操作和失败恢复缺少持久化状态，服务重启后难以继续处理。

当前统一入口覆盖以下账号类型：

| 平台 | 添加方式 | Gateway 底层来源 |
|---|---|---|
| Anthropic | OAuth、API Key | Claude OAuth 认证文件、`claude-api-key` |
| OpenAI | OAuth、API Key | Codex OAuth 认证文件、`codex-api-key`、`openai-compatibility` |
| Gemini | OAuth、API Key、Vertex | Gemini 插件认证、`gemini-api-key`、Vertex Service Account |
| Antigravity | OAuth | Antigravity OAuth 认证文件 |
| Grok / xAI | OAuth | xAI OAuth 认证文件 |

OpenAI 格式的中转站统一从 OpenAI API Key 入口添加，由协议探测决定使用 Responses 或 Chat Completions；Gemini API Key 入口只处理 Gemini 原生协议。

#### 实现状态与边界

统一账号管理已由 CPA-Manager-Plus 和 CLIProxyAPI 两个 fork 共同实现。CPA-Manager-Plus 保存统一账号 ID、绑定历史、操作状态和页面数据，CLIProxyAPI 继续负责凭证持久化、账号调度和真实模型请求。

Manager 不直接读取或修改 Gateway 宿主机的配置文件、认证目录或运行数据库，所有底层操作均通过 CLIProxyAPI Management API 完成。原 `/ai-providers`、`/auth-files` 和 `/oauth` 路由继续保留为高级管理入口，统一账号页使用 `/accounts`。

```text
浏览器 /accounts
  -> CPA-Manager-Plus /v0/pro/accounts/*
     -> ProAccountService / Lifecycle / Usage / Probe
        -> ProAccountGateway Client
           -> CLIProxyAPI /v0/management/*
              -> 配置、认证文件、Auth Registry 和生产 Executor
```

#### Manager 端实施逻辑

1. `ProAccountService` 从 Gateway 的认证文件和 API 配置快照中生成归一化账号，再通过 `ResolveBinding` 同步到 Manager SQLite。
2. 每个账号由 Manager 生成稳定 UUID。当前 `auth_index`、来源定位和历史变更记录在绑定表中，Key、Base URL、文件路径或认证类型变化时不更换账号 UUID。
3. 无法唯一匹配的绑定漂移写入待确认记录，由管理员重绑；同步过程使用发现键保证幂等，不根据邮箱、指纹或 Key 摘要自动合并账号。
4. 列表接口使用游标分页，并在 Manager 侧统一完成搜索、平台、认证方式、启用状态和健康状态筛选。
5. API Key 创建先由 `ProAccountProbeService` 使用候选凭证探测协议和模型，再创建停用草稿、写入模型规则、执行连通性测试，成功后启用。OpenAI 优先选择 Responses，明确不支持时才回退 Chat Completions。
6. OAuth 和 Vertex 创建先在 Gateway 保存不可调度草稿，再由添加向导补充模型规则并完成测试。取消、超时或失败时由生命周期服务执行清理或补偿。
7. 编辑、启停、删除、协议迁移、重新授权和 Token 刷新统一由 `ProAccountLifecycleService` 编排；写操作携带资源版本、`operation_id` 和幂等键，避免并发覆盖及重复执行。
8. `pro_account_drafts` 保存跨多个 Management API 调用的状态。Manager 启动后的恢复 Worker 会扫描未完成操作，继续执行或进入补偿流程。
9. 账号管理模型目录由项目内置目录、上游模型列表和用户手工输入合并，用于账号配置与连通性测试，不等同于 3.3 定义的客户端模型列表。`allowed_models` 在映射前校验客户端模型名，`model_mapping` 再解析为上游模型；规则写入后必须从 Gateway 回读确认。
10. `ProAccountUsageService` 按账号全部历史绑定聚合 `usage_events`，并通过平台 Adapter 补充官方配额。前端只在用户主动刷新时请求官方配额，列表自动刷新使用被动快照和本地统计。
11. 批量启停、测试、删除、绑定重审、OpenAI reset credits 和数据库备份均在 Manager 侧实现，不改变 Gateway 的普通模型请求入口。

#### SQLite 数据

| 表 | 用途 |
|---|---|
| `pro_accounts` | 保存稳定账号 UUID、平台、认证方式、状态、模型规则和资源版本 |
| `pro_account_bindings` | 保存账号与 `auth_index`、来源定位之间的当前及历史有效期 |
| `pro_account_binding_reviews` | 保存无法自动确认的绑定漂移及候选账号 |
| `pro_account_drafts` | 保存创建、编辑、测试、迁移和补偿操作状态，不保存凭证明文 |
| `usage_events` | 复用现有原始用量事件，按事件时间与绑定有效期归属到统一账号 |

上述表由 `internal/repository/sqlite/migrate.go` 统一创建和升级。SQLite 数据目录需要持久化；自动备份、恢复和命令行备份逻辑位于 `internal/repository/sqlite/backup.go`、`internal/worker/database_backup.go` 和 `internal/command/databasebackup/`。

#### BFF 接口分组

`/v0/pro/*` 由 Manager 本地鉴权和处理，未知路径直接返回 `404`，不会转发给 Gateway。

| 接口组 | 用途 |
|---|---|
| `GET/POST /v0/pro/accounts`、`GET/PUT/DELETE /v0/pro/accounts/:id` | 列表、详情和账号生命周期操作 |
| `POST /v0/pro/accounts/sync`、`GET /binding-reviews`、`POST /rebind` | 存量同步和绑定漂移处理 |
| `GET /capabilities`、`POST /probe`、`GET /model-catalog` | Gateway 能力检测、候选凭证探测和账号管理静态模型目录 |
| `/oauth/*`、`POST /vertex`、`POST /:id/complete` | OAuth、Vertex 和停用草稿完成流程；OAuth 状态查询使用 GET |
| `GET/PUT /:id/models` | 账号管理模型目录、白名单和映射规则 |
| `GET /:id/usage`、`GET /:id/openai-reset-credits`、`POST /:id/openai-reset` | 本地/官方用量及 OpenAI 配额重置 |
| `POST /batch`、`GET /operations/*` | 批量操作和持久化操作状态查询 |
| `/:id/reauthorize/*`、`POST /:id/refresh-token` | OAuth 重新授权和主动刷新凭证；重新授权状态查询使用 GET |

前端请求类型和序列化集中在 `apps/web/src/services/api/proAccounts.ts`，后端入口集中在 `internal/http/controller/proaccount/`，避免账号业务继续散布到原有 Management API 代理代码中。

#### 前端实现

- `AccountsPage.tsx`：统一账号表格、筛选、选择、批量操作、行内用量和自动刷新。
- `AccountWizardModal.tsx`：API、OAuth 和 Vertex 添加流程，以及草稿完成和取消清理。
- `AccountEditModal.tsx`、`AccountModelRulesEditor.tsx`：账号编辑、账号管理模型目录、白名单和映射。
- `AccountStatsModal.tsx`、`AccountActionsModal.module.scss`：统计详情和账号操作布局。
- `AccountReauthorizeModal.tsx`：OAuth 账号重新授权。
- `AccountBindingReviewModal.tsx`：绑定漂移确认和人工重绑。
- `accountRefresh.ts`、`accountUsageCache.ts`、`accountTablePresentation.ts`：刷新合并、行内缓存和表格展示逻辑。

#### CLIProxyAPI 接入

Gateway fork 增加两类通用能力，不保存 Pro 账号 UUID：

1. `allowed_models`、模型别名和规则版本进入认证快照、配置合成、Auth Registry 注册及调度路径，使客户端模型列表和真实请求共同遵守账号状态与 `allowed_models` 等账号级规则。客户端模型列表还须按 3.3 的开关行为进行协议组过滤；该过滤不进入实际请求的模型校验与调度路径。
2. OAuth、插件 OAuth 和 Vertex 认证文件支持首次以 `credential_draft`（兼容旧 `pro_draft`）和 Disabled 状态持久化；API 配置项通过 Management API 直接创建为 Disabled。File、Object、PostgreSQL 和 Git Store 共用认证草稿判断，普通已删除的 Disabled 凭证仍不会被意外重建。

Manager 依赖 Gateway 返回稳定的 `auth_index`、来源定位、实际启用状态、模型规则版本和能力标记；新增 Gateway Management API 字段必须保持向后兼容。

#### 文件范围与上游合并关注点

| 组件 | Pro 新增模块 | 需要与上游反复合并的接入文件 |
|---|---|---|
| CPA-Manager-Plus 后端 | `internal/http/controller/proaccount/`、`internal/model/pro_account*.go`、`internal/repository/proaccount*/`、`internal/service/proaccount*/`、`internal/worker/pro_account_*` | `internal/app/context.go`、`internal/http/router/router.go`、`internal/repository/sqlite/migrate.go`、`internal/store/store.go`、`cmd/cpa-manager-plus/main.go` |
| CPA-Manager-Plus 前端 | `apps/web/src/features/accounts/`、`apps/web/src/pages/AccountsPage.tsx`、`apps/web/src/services/api/proAccounts.ts` | `MainLayout.tsx`、`MainRoutes.tsx`、`services/api/index.ts`、旧账号页面入口 |
| CLIProxyAPI | `sdk/cliproxy/allowed_models.go`、`sdk/cliproxy/auth/allowed_models.go`、`sdk/cliproxy/auth/credential_draft.go` | `internal/api/handlers/management/auth_files.go`、`config_*.go`、`oauth_sessions.go`、`vertex_import.go`、`internal/watcher/synthesizer/*`、四种 Token Store、`sdk/cliproxy/service.go`、`internal/api/server.go` |

合并上游时先检查上述接入文件，再检查新增目录。重点确认：`/v0/pro/*` 仍在 Management API 代理之前本地匹配；SQLite 迁移保持增量兼容；Gateway 仍回显 `auth_index`、草稿状态和模型规则；四种 Token Store 的 Disabled 草稿语义一致；旧账号路由仍可访问。

#### 验证入口

- Manager 后端：在 `components/cpa-manager-plus/apps/manager-server` 执行 `go test ./...`。
- Manager 前端：在 `components/cpa-manager-plus/apps/web` 执行 `npm test`、`npm run type-check` 和 `npm run build`。
- Gateway：在 `components/cliproxyapi` 执行 `go test ./...`。
- 上游合并后至少回归存量同步、API/OAuth/Vertex 创建、编辑迁移、启停删除、模型规则、用量聚合、恢复 Worker、数据库备份和旧路由兼容。

### 3.2 账号连通性测试

#### 主要解决的问题

账号测试不能只验证某个模型端点是否整体可用，还必须证明指定账号在正式执行链中能够完成请求。Pro 连通性测试主要解决：

1. 使用普通模型请求测试时，调度器可能自动切换到其他可用账号，成功结果不能证明被检查账号本身可用。
2. 新建草稿或已停用账号不参与正常调度，如果为测试而临时启用，可能在验证完成前接收真实流量。
3. 不同平台分别使用 Responses、Chat Completions、Messages 和 Generate Content；独立拼接探测请求容易偏离正式协议转换和 Executor 行为。
4. 测试模型必须经过账号白名单和别名映射，否则测试结果与账号模型规则、调度及真实请求不一致；3.3 的协议组显示过滤不作为连通性测试的调用限制。
5. 上游错误格式差异较大，原始信息可能过长或包含凭证，需要统一分类、脱敏并保存最近测试状态。
6. 单次人工测试无法持续发现凭证过期、模型权限或上游连通性变化，需要可复用同一执行逻辑的定时测试和历史结果。

实现后的测试会固定到账号当前绑定的 `auth_index`，允许在不改变持久化启用状态的情况下检查停用账号，并复用生产 Translator、Executor 和 Transport，因此结果反映的是该账号实际进入生产链路时的可用性。

#### 调用链

账号连通性测试已复用 Gateway 的生产翻译和 Executor 链路，不通过 Manager 直接拼接上游 HTTP 请求。

```text
AccountTestModal
  -> POST /v0/pro/accounts/:id/test
     -> ProAccountTestService
        -> 校验账号版本、当前绑定和 allowed_models
        -> 应用 model_mapping
        -> POST CLIProxyAPI /v0/management/account-test
           -> 按 auth_index 定位并固定 Auth
           -> 构造最小非流式请求
           -> ExecuteProtocolWithAuthManager
           -> 生产 Translator / Executor / Transport
```

Manager 使用 `operation_id` 和幂等键记录测试操作，并在完成后更新账号的 `last_tested_at`、健康状态和最近错误。网络调用失败与上游返回的业务失败分别处理，避免把 Manager 到 Gateway 的连接问题误判成账号问题。

#### 模型和协议处理

1. 前端从账号管理模型目录选择测试模型，也允许输入手工模型。
2. Manager 先用账号当前 `allowed_models` 校验客户端模型名，再应用 `model_mapping` 得到上游模型。
3. OpenAI Responses 使用 `responses`，OpenAI Compatibility 和 xAI 使用 `chat_completions`，Anthropic 使用 `messages`，Gemini 和 Antigravity 使用 `generate_content`。
4. `default` 模式发送普通最小请求；`compact` 模式只允许 OpenAI Responses 账号，并走 `responses/compact`。
5. Gateway 根据 `auth_index` 找到运行时 Auth，再通过固定认证元数据禁止调度器切换到其他账号。
6. 测试停用账号时只克隆并临时清除 Disabled、冷却和错误状态，数据库及 Auth Registry 中的原账号状态不变。
7. 连通性测试带有 `account_connectivity_test` 和 `execution_source` 元数据，失败不会写入正常调度冷却状态。

#### 结果处理

Gateway 返回成功状态、HTTP 状态、协议、模式、模型、上游模型、耗时、响应摘要、错误码和是否可重试。错误统一归类为认证失败、模型不可用、协议不支持、配额耗尽、限流、账号/Executor 不可用及上游异常。

响应摘要和错误文本会限制长度，并对常见 API Key 格式进行脱敏。Manager 再将错误码转换为管理员可读提示，同时保留脱敏后的上游摘要用于排查。

#### 定时测试

定时测试复用同一个 `ProAccountTestService`，不会维护第二套测试实现：

- `pro_account_scheduled_tests` 保存账号、模型、Cron、启用状态、结果保留数和自动恢复选项。
- `pro_account_scheduled_test_results` 保存每次运行的状态、错误、响应摘要和延迟。
- `ProAccountScheduledTestRunner` 在 Manager 启动后执行到期计划，并按 `max_results` 清理旧结果。
- `/v0/pro/accounts/:id/scheduled-test-plans` 提供计划的查询、创建、更新和删除；子路由提供立即运行和结果查询。
- `AccountScheduledTestsModal.tsx` 负责计划编辑和历史结果展示。

#### 修改文件

| 组件 | 主要文件 |
|---|---|
| Manager 前端 | `apps/web/src/features/accounts/AccountTestModal.tsx`、`AccountScheduledTestsModal.tsx`、`apps/web/src/services/api/proAccounts.ts` |
| Manager BFF | `internal/http/controller/proaccount/handler.go`、`internal/http/controller/proaccountscheduledtest/` |
| Manager 服务 | `internal/service/proaccounttest/service.go`、`internal/service/proaccountgateway/connectivity.go`、`internal/service/proaccountscheduledtest/` |
| Manager 持久化与任务 | `internal/model/pro_account_schedule.go`、`internal/repository/proaccountscheduledtest/`、`internal/repository/sqlite/pro_account_scheduled_test_migration.go`、`internal/worker/pro_account_scheduled_runner.go` |
| Gateway Management API | `internal/api/handlers/management/account_connectivity.go`、`internal/api/server.go` |
| Gateway 生产执行接入 | `sdk/api/handlers/handlers.go`、`sdk/cliproxy/executor/types.go`、`sdk/cliproxy/auth/conductor.go` |

对应测试集中在同目录的 `*_test.go`、`AccountTestModal.test.tsx` 和 `AccountScheduledTestsModal.test.tsx`。上游合并时重点检查 Gateway 的 Management 路由注册、执行元数据传递和 Auth 选择逻辑，确保测试仍固定到指定账号且不会改变正常调度状态。

### 3.3 按协议显示模型列表

#### 主要解决的问题

CLIProxyAPI 默认在不同协议的模型列表接口中返回同一份当前可用模型。对于会根据 `Provider API` 自动读取模型目录的 Kilo、Cline、Cursor 和 Roo Code，这会把无法由当前上游协议直接提供的模型一起展示。按协议显示模型列表主要解决：

1. Anthropic、OpenAI 和 Gemini 客户端读取模型列表时缺少与上游账号协议一致的展示范围。
2. 模型名称不能可靠表示协议归属；通过 OpenAI 兼容协议接入的智谱、Grok 或其他中转站模型仍应属于 OpenAI 协议组。
3. 同名模型可能同时由多个协议组的账号提供，过滤后仍需使用当前协议组账号对应的模型元数据和可用状态。
4. 模型列表必须继续遵守账号启用状态、`allowed_models`、配额冷却和暂停状态，不能维护一套脱离 Model Registry 的静态列表。
5. 插件模型、未知来源模型和 Home 控制中心模型需要保持原有可见性，避免新协议或扩展模型被错误隐藏。
6. 列表过滤只能改善客户端选型体验，不能破坏 CLIProxyAPI 已有的跨协议翻译、模型映射和实际请求调度。

#### 实现状态与边界

按协议显示模型列表已由 CPA-Manager-Plus 和 CLIProxyAPI 两个 fork 共同实现。Manager 在可视化配置的 Pro 分区提供开关，Gateway 负责账号协议归组、模型注册和列表过滤。

配置字段为 `protocol-model-list-enabled`，默认值是 `false`。配置修改通过现有 Management API 保存到 CLIProxyAPI 配置，并由配置监听器热更新到 `BaseAPIHandler` 的原子快照，不需要重启 Gateway。

该字段只在模型列表 Handler 中读取，不进入模型请求校验、Translator、Executor 或 Auth 调度路径。关闭时保持 CLIProxyAPI 原有全量模型目录；开启时才按客户端模型列表协议选择协议组。

```text
CPA-Manager-Plus VisualConfigEditor
  -> useVisualConfig 读写 protocol-model-list-enabled
     -> CLIProxyAPI Management API 保存配置
        -> Config Watcher 热更新 SDKConfig
           -> BaseAPIHandler.UpdateClients
              -> Models Handler
                 -> ModelRegistry.GetAvailableModelsForProtocol
```

#### 协议组与模型列表入口

Kilo 根据用户选择的 `Provider API` 使用对应推理端点和模型列表请求。OpenAI Responses 和 OpenAI Compatible 的推理入口不同，但共用 OpenAI 模型列表协议组。

| Provider API | 推理端点 | 模型列表识别方式 | Gateway 协议组 |
|---|---|---|---|
| Anthropic Messages | `/v1/messages` | `GET /v1/models`，携带 `Anthropic-Version` | `claude` |
| OpenAI Responses | `/v1/responses` | `GET /v1/models`，使用 OpenAI 格式 | `openai` |
| OpenAI Compatible | `/v1/chat/completions` | `GET /v1/models`，使用 OpenAI 格式 | `openai` |
| Gemini Native | `/v1beta/models/{model}:generateContent` | `GET /v1beta/models` | `gemini` |

`protocolGroupForAuth` 根据账号实际 Provider 和协议配置确定归属：

- Claude 账号归入 `claude`。
- OpenAI、Codex、Kimi、xAI 和 `openai-compatibility` 账号归入 `openai`。
- Gemini、Gemini Interactions、Vertex、AI Studio 和 Antigravity 账号归入 `gemini`。
- 插件虚拟认证和无法识别的 Provider 不设置协议组，其模型对所有协议目录可见。

#### Gateway 实施逻辑

1. `Service` 解析账号模型后先应用 `allowed_models`，再通过 `RegisterClientWithProtocolGroupAndUnscopedModels` 把账号 ID、Provider、协议组和模型注册到全局 Model Registry。
2. `ModelRegistry` 在原有客户端模型关系之外保存 `clientProtocolGroups` 和模型级 `clientUnscopedModels`。插件附加模型可标记为不限定协议组，而不必把整个宿主账号改成全局可见。
3. `GetAvailableModelsForProtocol` 只统计目标协议组和未归组客户端提供的模型，同时复用原有账号计数、配额恢复时间、冷却、暂停和模型元数据选择逻辑。
4. 协议模型缓存使用输出格式和协议组组成独立键；账号模型、协议归属或可用状态变化时沿用 Model Registry 的缓存失效机制，配额恢复后也会重新计算。
5. Claude、OpenAI Chat、OpenAI Responses 和 Gemini 模型 Handler 从 `BaseAPIHandler` 读取热更新开关；关闭时调用 `GetAvailableModels`，开启时调用 `GetAvailableModelsForProtocol`。
6. `/v1/models` 由统一路由根据 `Anthropic-Version` 等请求特征选择 Claude 或 OpenAI 输出格式；`/v1beta/models` 使用 Gemini 输出格式。
7. Home 模式在 `internal/api/server.go` 中优先走控制中心模型目录，不进入本地协议过滤。未知来源和插件模型通过未归组或模型级全局可见标记保留在各协议目录中。
8. 实际推理请求不读取 `protocol-model-list-enabled`。用户手工填写其他协议组的模型 ID 时，只要符合账号白名单且存在可用账号，Gateway 仍通过现有 Translator 和 Executor 完成跨协议调用。

例如，客户端通过 `/v1/messages` 手工调用 OpenAI 协议账号提供的模型时，该模型可以不出现在 Claude 模型目录中，但 Gateway 仍会将 Anthropic Messages 请求转换为对应的 OpenAI 上游协议，并把响应转换回 Anthropic Messages 格式。

#### Manager 配置接入

- `VisualConfigEditor.tsx` 在 Pro 配置分区展示“按协议显示模型列表”开关。
- `visualConfig.ts` 保存前端字段定义和默认值，默认关闭。
- `useVisualConfig.ts` 将 `protocolModelListEnabled` 与 YAML 字段 `protocol-model-list-enabled` 双向转换，只在字段被修改时写回配置，保留未知配置项。
- Manager 不读取 Model Registry，也不自行过滤模型；保存成功后的实际行为以 Gateway 热更新结果为准。

#### 修改文件

| 组件 | 主要文件 |
|---|---|
| Manager 可视化配置 | `apps/web/src/components/config/VisualConfigEditor.tsx`、`apps/web/src/hooks/useVisualConfig.ts`、`apps/web/src/types/visualConfig.ts` |
| Gateway 配置与热更新 | `config.example.yaml`、`internal/config/sdk_config.go`、`internal/watcher/diff/config_diff.go`、`sdk/api/handlers/handlers.go` |
| Gateway 协议归组与注册 | `sdk/cliproxy/service.go`、`internal/registry/model_registry.go`、`internal/registry/protocol_models.go` |
| Gateway 模型列表入口 | `internal/api/server.go`、`sdk/api/handlers/claude/code_handlers.go`、`sdk/api/handlers/openai/openai_handlers.go`、`sdk/api/handlers/openai/openai_responses_handlers.go`、`sdk/api/handlers/gemini/gemini_handlers.go` |

#### 验证入口

- 配置解析与热更新：`internal/config/protocol_model_list_test.go`、`internal/watcher/diff/config_diff_test.go`、`sdk/api/handlers/handlers_protocol_model_list_test.go`。
- 协议归组、缓存和可用性：`internal/registry/protocol_models_test.go`、`sdk/cliproxy/service_protocol_models_test.go`。
- 各协议模型目录：`sdk/api/handlers/claude/protocol_models_test.go`、`sdk/api/handlers/openai/protocol_models_test.go`、`sdk/api/handlers/gemini/protocol_models_test.go`。
- 跨协议调用与 Home 边界：`sdk/api/handlers/model_execution_test.go`、`internal/api/server_protocol_model_list_test.go`。
- Manager 配置读写：`apps/web/src/components/config/VisualConfigEditor.test.tsx`、`apps/web/src/hooks/useVisualConfig.test.ts`。
- Gateway 在 `components/cliproxyapi` 执行 `go test ./...`；Manager 前端在 `components/cpa-manager-plus` 执行 `npm test`、`npm run type-check` 和 `npm run build`。

### 3.4 API Key 官方客户端兼容

Anthropic 和 OpenAI API Key 账号提供“官方客户端兼容”开关。

开启后，让 Kilo、Cline、Cursor、Roo Code 等非官方客户端的上游请求尽量接近 Claude Desktop 和 Codex Desktop 直接使用该 API Key 时的请求形态。

该功能基于 CLIProxyAPI 现有 cloak 框架扩展。不开启时保持 CLIProxyAPI 原有处理；识别为官方 Claude 或 Codex 客户端时不重复改写。

#### 通用要求

- 使用可版本化的 Claude 和 Codex profile，并按真实官方客户端抓包维护。
- 先执行账号模型映射，再根据最终模型构造兼容请求。
- 测试连接必须复用正式请求的 profile 和构造链。
- 受保护的身份 Header 不允许被账号自定义 Header 覆盖。
- 页面、日志和抓包不得暴露 API Key、OAuth Token 和鉴权 Header。

#### Anthropic API Key

目标请求形态为 Claude Desktop。主要要求：

- `/v1/messages` 和 `/v1/messages/count_tokens` 使用各自独立的构造规则。
- 按 profile 对齐客户端身份 Header、Anthropic Beta 顺序、System、Metadata、Session 和缓存控制。
- 对工具名称和工具结构进行必要归一，保持请求与响应中的工具调用可以正确对应。
- 区分固定身份 Beta 和根据请求内容动态增加的功能 Beta。
- TLS 默认保持 CLIProxyAPI 原有 Transport；只有管理员明确选择 TLS profile 时才启用对应指纹。

#### OpenAI API Key

目标请求形态为 Codex Desktop。主要要求：

- 按 profile 对齐客户端身份 Header、`originator`、`x-codex-*` 和 Turn Metadata。
- 按官方请求形态补齐 Responses 所需的 Body 默认值、缓存键和客户端 Metadata。
- `/v1/responses`、Responses Compact、SSE 和 WebSocket 分别处理，不共用一套模板。
- Compact 保持独立的非流式 JSON 请求形态，不盲目补入普通 Responses 字段。
- HTTP、SSE、WebSocket 和 TLS 的选择以对应 profile 和真实抓包为准。

#### 验证要求

兼容效果必须通过以下方式验证：

- 对比官方客户端和兼容请求的 Header、Body、TLS 和路由。
- 分别验证正式请求和账号连通性测试。
- 同时回归官方客户端和 Kilo、Cline、Cursor、Roo Code 等非官方客户端。
- 以上游实际收到的请求为准，不能只根据本地 Usage 页面判断。

具体请求基线参考 [sub2apiplus README.md 第 1.1 节“API Key 官方客户端兼容”](https://github.com/itv3/sub2apiplus/blob/main/README.md#11-api-key-官方客户端兼容)，开发时不得只复制单个 Header 或常量。

该功能只要求尽量接近已验证的官方请求形态，不复制隐藏 Prompt、产品 Memory 或账号状态，也不承诺 TLS 和 HTTP/2 帧级指纹完全一致。

## 4. 组件分工

| 功能 | CPA-Manager-Plus | CLIProxyAPI |
|---|---|---|
| 统一账号页面和添加向导 | 主要实现 | 提供 Management API |
| OpenAI 协议探测 | 负责探测流程和结果判断 | 优先复用现有 Management API，不默认新增专用接口 |
| 模型白名单和映射 | 配置页面 | 运行时执行 |
| 连通性测试 | 页面和结果展示 | 执行真实请求 |
| 用量、成本和健康状态 | 主要实现 | 产生运行数据 |
| 官方客户端兼容 | 配置和状态展示 | Executor、Transport 和路由执行 |
| 按协议显示模型列表 | 配置页面 | 识别客户端协议并按上游账号协议过滤模型列表 |

CPA-Manager-Plus 不代理普通模型请求，只通过 CLIProxyAPI Management API 管理配置和认证文件，不绕过 API 直接读写 Gateway 宿主机文件、认证目录或运行数据库。

## 5. 第一版验收标准

1. 一个页面可以查看和管理全部平台账号。
2. 添加账号必须先选平台，再选 OAuth 或 API Key。
3. OpenAI API Key 可以自动识别 Responses 和 Chat Completions 能力。
4. OAuth 和 API Key 账号都可以配置允许模型和模型映射。
5. 每个账号都可以测试连通性，并显示明确失败原因。
6. 开启官方客户端兼容后，Claude 和 Codex 请求通过抓包对比测试。
7. 开启按协议显示模型列表后，各模型目录仅显示对应账号协议组；未知、插件及 Home 模型保持全量；列表隐藏不影响手工调用。
8. 每个账号和 Key 都能查看 Token、请求次数和官方成本估算。
9. Manager 故障不影响 CLIProxyAPI 继续处理模型请求。
10. 两个组件可以独立升级和回滚。

## 6. 建议开发顺序

1. 建立 Pro 主项目、两个 fork 和统一 Compose。
2. 定义 Management API、用量事件和协议模型策略契约。
3. 实现统一账号页面、添加向导、用量窗口和连通性测试。
4. 实现 OpenAI API Key 自动探测和账号模型配置。
5. 在 CLIProxyAPI 增加 Claude/Codex 官方客户端兼容 profile。
6. 在 CLIProxyAPI 实现协议模型策略、协议组归属、开关行为和模型列表过滤规则。
7. 完成集成与升级测试后发布第一版。

## 7. 默认目录与部署

开发工作区：

```text
cliproxyapi-pro/
├── .references/                 # 本地只读参考源码，主仓库忽略
│   └── sub2apiplus/
├── components/
│   ├── cliproxyapi/
│   └── cpa-manager-plus/
├── deploy/
├── integration/
└── docs/
```

生成两个 Docker 镜像：

```text
cliproxyapi-pro-gateway
cliproxyapi-pro-manager
```

默认安装目录：

```text
/opt/cliproxyapi-pro/
├── compose.yaml
├── .env
├── gateway/
│   ├── config.yaml
│   ├── auths/
│   └── logs/
├── manager/
│   └── data/
└── proxy/
```

`gateway` 负责模型请求，`manager` 负责管理页面和统计。两个镜像通过一个 Compose 安装，对外使用同一个域名。

`sub2apiplus` 只作为本地开发参考，可放在工作区的 `.references/sub2apiplus`；该目录被主仓库本地忽略，不属于 CLIProxyAPI Pro Git 仓库、产品架构或部署内容。

产品名称使用 `CLIProxyAPI Pro`；开发目录、仓库名、安装目录、Compose 项目名和 Docker 镜像名称统一使用小写 `cliproxyapi-pro`。
