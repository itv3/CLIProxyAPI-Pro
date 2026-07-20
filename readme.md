# CLIProxyAPI Pro 需求与开发说明

## 1. 项目定位

CLIProxyAPI Pro 基于两个开源项目开发：

- [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)：负责模型请求、协议转换、账号调度、OAuth、API Key 和统一下游 Key。
- [CPA-Manager-Plus](https://github.com/seakee/CPA-Manager-Plus)：负责管理页面、账号管理、统计、成本、监控和运维。

开发工作区初始使用两个官方仓库作为 Git 子模块；建立自有 fork 后替换子模块地址，并继续保留官方仓库作为上游更新来源。组件版本以主仓库记录的子模块 SHA 为唯一权威，CI 从对应 SHA 构建 Docker 镜像，镜像标签只标识发布产物，不构成第二套版本来源。

```text
CLIProxyAPI Pro
= CPA-Manager-Plus 主要改造
+ CLIProxyAPI 有界运行时改造
+ 统一安装、测试和发布
```

[sub2apiplus](https://github.com/itv3/sub2apiplus) 仅作为账号交互、协议探测、连通性测试和官方客户端兼容的参考，不是运行依赖。

### 组件显示版本

运行中的组件使用 `上游版本-修改序号` 作为显示版本。例如，`v7.2.92-1` 表示基于上游 `v7.2.92` 的第一个 Pro 修改版；同一上游基线继续发布时依次使用 `-2`、`-3`，合并新的上游版本后从 `-1` 重新开始。

更新检查只使用连字符前的上游基线进行比较，因此 `v7.2.92-1` 与上游 `v7.2.92` 视为相同版本，上游发布 `v7.2.93` 后才提示更新。CLIProxyAPI Pro 主仓库的 `v1.1` 等标签继续表示整体产品发布，Docker 镜像标签表示对应发布产物；组件源码身份仍以主仓库记录的子模块 SHA 为最终依据。

## 2. 架构与组件边界

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

普通模型请求不经过 CPA-Manager-Plus；Manager 停止服务时，反向代理和 CLIProxyAPI 仍应独立处理模型请求。

| 功能 | CPA-Manager-Plus | CLIProxyAPI |
|---|---|---|
| 统一账号和添加向导 | 主要实现 | 提供 Management API |
| OpenAI 协议探测 | 负责流程和结果判断 | 优先复用现有 Management API |
| 模型白名单和映射 | 配置页面 | 运行时执行 |
| 连通性测试 | 页面、编排和结果 | 固定账号执行真实请求 |
| 用量、成本和健康状态 | 聚合和展示 | 产生运行数据 |
| 官方客户端兼容 | 配置和状态展示 | Executor、Transport 和路由执行 |
| 按协议显示模型列表 | 配置页面 | 账号归组和列表过滤 |

开发原则：

1. 分别维护 CLIProxyAPI fork 和 CPA-Manager-Plus fork，由 Pro 主项目锁定两个组件版本。
2. 使用 Docker Compose 一键安装，两个组件保持独立容器和数据目录，不合并为一个 Go 工程、二进制或进程。
3. Pro 功能尽量通过新增模块和少量接入点实现，方便继续合并上游更新。
4. Manager 只通过 Management API 管理 Gateway，不直接读写其宿主机配置文件、认证目录或运行数据库。
5. Manager SQLite 与 Gateway 配置及认证文件之间不存在跨组件事务；跨组件写操作使用持久化草稿状态机、幂等键和补偿动作实现最终一致。

### 本地参考源码

`.references/sub2apiplus` 保存 [itv3/sub2apiplus](https://github.com/itv3/sub2apiplus) 的只读浅克隆，并将 [Wei-Shaw/sub2api](https://github.com/Wei-Shaw/sub2api) 配置为上游参考地址，供开发人员和 AI 阅读、检索及对比页面布局、交互流程、协议探测和兼容逻辑。

`.references/` 是本机开发辅助目录，不是正式组件、Git 子模块、构建依赖、运行依赖或发布内容。主仓库通过 `.git/info/exclude` 的 `/.references/` 规则在本地忽略该目录，其中的文件不进入提交，也不参与 CI、构建、打包或部署。

参考源码只能读取，禁止在其中实现 Pro 功能、安装依赖、执行构建或产生提交。更新时只能在参考仓库中执行 `git pull --ff-only`；正式实现必须写入 `components/cpa-manager-plus` 或 `components/cliproxyapi`。

## 3. 功能状态

| 功能 | 状态 | 当前边界 |
|---|---|---|
| 统一账号管理 | 已实现 | 覆盖 OAuth、API Key、Vertex 和插件账号 |
| 账号连通性测试 | 已实现 | 固定指定账号，包含定时测试和历史结果 |
| 按协议显示模型列表 | 已实现 | 默认关闭，只影响模型列表，不限制实际请求 |
| API Key 官方客户端兼容 | 已实现 | 当前支持 Anthropic Claude Desktop 和 OpenAI Codex Desktop Profile |
| 统一 IP 库代理 | 待开发 | 分为账号引用和 OAuth 会话级代理两个阶段 |

## 4. 功能需求与实现边界

### 4.1 统一账号管理

#### 目标与范围

CLIProxyAPI 原生按 API 配置、OAuth 认证文件和插件认证分别管理凭证。Pro 在不替换 Gateway 凭证与调度机制的前提下提供稳定、统一的账号视图，重点解决入口分散、数据结构不一、`auth_index` 变化导致身份断裂、多步操作产生半完成配置、模型与用量信息来源不同，以及重启后无法恢复未完成操作等问题。

| 平台 | 添加方式 | Gateway 底层来源 |
|---|---|---|
| Anthropic | OAuth、API Key | Claude OAuth 认证文件、`claude-api-key` |
| OpenAI | OAuth、API Key | Codex OAuth 认证文件、`codex-api-key`、`openai-compatibility` |
| Gemini | OAuth、API Key、Vertex | Gemini 插件认证、`gemini-api-key`、Vertex Service Account |
| Antigravity | OAuth | Antigravity OAuth 认证文件 |
| Grok / xAI | OAuth | xAI OAuth 认证文件 |

OpenAI 格式的中转站统一从 OpenAI API Key 入口添加，由协议探测决定使用 Responses 或 Chat Completions；Gemini API Key 入口只处理 Gemini 原生协议。

统一账号管理由两个 fork 共同实现：Manager 保存统一账号 ID、绑定历史、操作状态和页面数据，Gateway 继续负责凭证持久化、账号调度和真实模型请求。原 `/ai-providers`、`/auth-files` 和 `/oauth` 路由保留为高级入口，统一账号页使用 `/accounts`。

```text
浏览器 /accounts
  -> CPA-Manager-Plus /v0/pro/accounts/*
     -> ProAccountService / Lifecycle / Usage / Probe
        -> ProAccountGateway Client
           -> CLIProxyAPI /v0/management/*
              -> 配置、认证文件、Auth Registry 和生产 Executor
```

#### 核心流程

1. `ProAccountService` 从 Gateway 的认证文件和 API 配置快照生成归一化账号，再通过 `ResolveBinding` 同步到 Manager SQLite。
2. Manager 为每个账号生成稳定 UUID，并在绑定表保存当前及历史 `auth_index` 和来源定位；Key、Base URL、文件路径或认证类型变化时不更换 UUID。
3. 同步过程使用发现键保证幂等，不根据邮箱、指纹或 Key 摘要自动合并账号；无法唯一匹配的绑定漂移进入待确认记录，由管理员重绑。
4. 列表使用游标分页，并由 Manager 统一执行搜索、平台、认证方式、启用状态和健康状态筛选。
5. API Key 创建时先探测协议和模型，再创建停用草稿、写入模型规则、执行连通性测试，成功后启用。OpenAI 优先选择 Responses，明确不支持时才回退 Chat Completions。
6. OAuth 和 Vertex 先在 Gateway 保存不可调度草稿，再由向导补充模型规则并测试；取消、超时或失败时由生命周期服务清理或补偿。
7. 编辑、启停、删除、协议迁移、重新授权和 Token 刷新统一由 `ProAccountLifecycleService` 编排。写操作携带资源版本、`operation_id` 和幂等键；`pro_account_drafts` 持久化跨 API 操作状态，恢复 Worker 在 Manager 启动后继续执行或补偿。
8. 账号管理模型目录合并项目内置目录、上游模型列表和用户输入，但不等同于 4.3 的客户端模型列表。`allowed_models` 先校验客户端模型名，`model_mapping` 再解析上游模型，写入后必须从 Gateway 回读确认。
9. 配置合成产生的 `alias == name` 条目只登记目录，不是真实映射，也不得隐式加入白名单；只有 `alias != name` 的有效别名可以随白名单模型注册。
10. `ProAccountUsageService` 按全部历史绑定聚合 `usage_events`，并通过平台 Adapter 补充官方配额。官方配额只在用户主动刷新时请求；列表自动刷新使用被动快照和本地统计。
11. 批量启停、测试、删除、绑定重审、OpenAI reset credits 和数据库备份均由 Manager 实现，不改变 Gateway 的模型请求入口。

#### SQLite 数据

| 表 | 用途 |
|---|---|
| `pro_accounts` | 稳定账号 UUID、平台、认证方式、状态、模型规则和资源版本 |
| `pro_account_bindings` | 账号与 `auth_index`、来源定位之间的当前及历史有效期 |
| `pro_account_binding_reviews` | 无法自动确认的绑定漂移及候选账号 |
| `pro_account_drafts` | 创建、编辑、测试、迁移和补偿状态，不保存凭证明文 |
| `usage_events` | 复用原始用量事件，按事件时间与绑定有效期归属到统一账号 |

这些表由 `internal/repository/sqlite/migrate.go` 统一创建和升级。SQLite 数据目录必须持久化；自动备份、恢复和命令行备份位于 `internal/repository/sqlite/backup.go`、`internal/worker/database_backup.go` 和 `internal/command/databasebackup/`。

#### BFF 接口

`/v0/pro/*` 由 Manager 本地鉴权和处理，未知路径返回 `404`，不会转发给 Gateway。

| 接口组 | 用途 |
|---|---|
| `GET/POST /v0/pro/accounts`、`GET/PUT/DELETE /v0/pro/accounts/:id` | 列表、详情和账号生命周期操作 |
| `POST /v0/pro/accounts/sync`、`GET /binding-reviews`、`POST /rebind` | 存量同步和绑定漂移处理 |
| `GET /capabilities`、`POST /probe`、`GET /model-catalog` | 能力检测、候选凭证探测和账号管理模型目录 |
| `/oauth/*`、`POST /vertex`、`POST /:id/complete` | OAuth、Vertex 和停用草稿完成流程；OAuth 状态使用 GET 查询 |
| `GET/PUT /:id/models` | 模型目录、白名单和映射规则 |
| `GET /:id/usage`、`GET /:id/openai-reset-credits`、`POST /:id/openai-reset` | 本地/官方用量及 OpenAI 配额重置 |
| `POST /batch`、`GET /operations/*` | 批量操作和持久化操作状态 |
| `/:id/reauthorize/*`、`POST /:id/refresh-token` | OAuth 重新授权和主动刷新凭证；重新授权状态使用 GET 查询 |

前端请求类型和序列化集中在 `apps/web/src/services/api/proAccounts.ts`，后端入口集中在 `internal/http/controller/proaccount/`，避免账号业务继续散布到原 Management API 代理代码中。

#### Gateway 接入

Gateway fork 增加两类通用能力，但不保存 Pro 账号 UUID：

1. `allowed_models`、模型别名和规则版本进入认证快照、配置合成、Auth Registry 注册及调度路径。账号的注册和调度范围由白名单模型及其非同名别名组成；白名单为空且存在非同名映射时只注册映射后的客户端模型名，白名单和映射均为空时保留允许全部模型的兼容行为。同名映射不得绕过白名单。客户端模型列表还按 4.3 的开关过滤协议组，但该过滤不进入实际请求校验与调度路径。
2. OAuth、插件 OAuth 和 Vertex 认证文件支持首次以 `credential_draft`（兼容旧 `pro_draft`）和 Disabled 状态持久化；API 配置项可通过 Management API 直接创建为 Disabled。File、Object、PostgreSQL 和 Git Store 共用草稿判断，普通已删除的 Disabled 凭证不会被意外重建。

Manager 依赖 Gateway 返回稳定的 `auth_index`、来源定位、实际启用状态、模型规则版本和能力标记；新增 Management API 字段必须向后兼容。

### 4.2 账号连通性测试

#### 目标与执行链

连通性测试必须证明指定账号在正式执行链中可用，而不是由调度器切换到其他账号；同时允许测试停用草稿，不改变其持久化状态。测试固定当前绑定的 `auth_index`，并复用生产 Translator、Executor 和 Transport。

```text
AccountTestModal
  -> POST /v0/pro/accounts/:id/test
     -> ProAccountTestService
        -> 校验账号版本、当前绑定和 allowed_models
        -> 应用 model_mapping
        -> POST CLIProxyAPI /v0/management/account-test
           -> 按 auth_index 固定 Auth
           -> 构造协议对应的最小测试请求
           -> ExecuteProtocolWithAuthManager
           -> 生产 Translator / Executor / Transport
```

Manager 使用 `operation_id` 和幂等键记录操作，完成后更新 `last_tested_at`、健康状态和最近错误。网络调用失败与上游业务失败分开处理，避免把 Manager 到 Gateway 的连接问题误判为账号问题。

#### 模型、协议和状态规则

1. 测试模型来自账号管理模型目录，也允许手工输入；Manager 先用 `allowed_models` 校验客户端模型名，再应用 `model_mapping`。
2. OpenAI Responses 使用 `responses`，OpenAI Compatibility 和 xAI 使用 `chat_completions`，Anthropic 使用 `messages`，Gemini 和 Antigravity 使用 `generate_content`。
3. `default` 发送普通最小请求；`compact` 只允许 OpenAI Responses 账号，并走 `responses/compact`。
4. Gateway 根据 `auth_index` 固定运行时 Auth，禁止调度器切换账号。
5. 测试停用账号时只克隆并临时清除 Disabled、冷却和错误状态，不修改数据库及 Auth Registry 中的原状态。
6. 测试携带 `account_connectivity_test` 和 `execution_source` 元数据，失败不会写入正常调度冷却状态。
7. 4.3 的协议组显示过滤只影响模型列表，不限制连通性测试调用。

Gateway 返回成功状态、HTTP 状态、协议、模式、模型、上游模型、耗时、响应摘要、错误码和是否可重试。错误统一归类为认证失败、模型不可用、协议不支持、配额耗尽、限流、账号或 Executor 不可用及上游异常。响应摘要和错误文本限制长度并脱敏常见 API Key，Manager 再转换为管理员可读提示，同时保留脱敏后的上游摘要。

#### 定时测试

定时测试复用同一个 `ProAccountTestService`，不维护第二套执行逻辑：

- `pro_account_scheduled_tests` 保存账号、模型、Cron、启用状态、结果保留数和自动恢复选项。
- `pro_account_scheduled_test_results` 保存每次运行的状态、错误、响应摘要和延迟。
- `ProAccountScheduledTestRunner` 在 Manager 启动后执行到期计划，并按 `max_results` 清理旧结果。
- `/v0/pro/accounts/:id/scheduled-test-plans` 提供计划的查询、创建、更新和删除；子路由提供立即运行和结果查询。
- `AccountScheduledTestsModal.tsx` 负责计划编辑和历史结果展示。

### 4.3 按协议显示模型列表

#### 目标与边界

CLIProxyAPI 默认在不同协议的模型列表接口返回同一份可用模型。Kilo、Cline、Cursor 和 Roo Code 等客户端会据此自动读取模型目录，可能展示当前上游协议无法直接提供的模型。Pro 以账号实际 Provider 和协议配置归组，在继续遵守账号启用状态、`allowed_models`、配额冷却和暂停状态的基础上过滤模型列表。

配置字段为 `protocol-model-list-enabled`，默认 `false`。Manager 通过现有 Management API 保存配置，Gateway 配置监听器将其热更新到 `BaseAPIHandler` 的原子快照，无需重启。该开关只在模型列表 Handler 中读取，不进入请求校验、Translator、Executor 或 Auth 调度路径；关闭时保持原有全量目录。

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

OpenAI Responses 和 OpenAI Compatible 的推理入口不同，但共用 OpenAI 模型列表协议组。

| Provider API | 推理端点 | 模型列表识别方式 | Gateway 协议组 |
|---|---|---|---|
| Anthropic Messages | `/v1/messages` | `GET /v1/models`，携带 `Anthropic-Version` | `claude` |
| OpenAI Responses | `/v1/responses` | `GET /v1/models`，OpenAI 格式 | `openai` |
| OpenAI Compatible | `/v1/chat/completions` | `GET /v1/models`，OpenAI 格式 | `openai` |
| Gemini Native | `/v1beta/models/{model}:generateContent` | `GET /v1beta/models` | `gemini` |

账号归组规则：

- Claude 归入 `claude`。
- OpenAI、Codex、Kimi、xAI 和 `openai-compatibility` 归入 `openai`。
- Gemini、Gemini Interactions、Vertex、AI Studio 和 Antigravity 归入 `gemini`。
- 插件虚拟认证和未知 Provider 不设置协议组，其模型对所有协议目录可见。

#### 核心规则

1. `Service` 解析账号模型后先应用 `allowed_models`；白名单为空但存在非同名映射时仅保留映射后的客户端模型名，随后将账号 ID、Provider、协议组和模型注册到全局 Model Registry。
2. `ModelRegistry` 在原有客户端模型关系之外保存 `clientProtocolGroups` 和模型级 `clientUnscopedModels`；插件附加模型可单独标记为不限协议组。
3. `GetAvailableModelsForProtocol` 只统计目标协议组和未归组客户端提供的模型，同时复用账号计数、配额恢复时间、冷却、暂停及模型元数据选择逻辑。
4. 缓存键包含输出格式和协议组；账号模型、归属或可用状态变化时沿用 Model Registry 的失效机制，配额恢复后重新计算。
5. Claude、OpenAI Chat、OpenAI Responses 和 Gemini Handler 根据热更新开关选择全量或按协议目录。
6. `/v1/models` 根据 `Anthropic-Version` 等请求特征选择 Claude 或 OpenAI 输出格式；`/v1beta/models` 使用 Gemini 格式。
7. Home 模式优先使用控制中心目录，不进入本地过滤；未知来源和插件模型通过未归组或模型级全局标记保持可见。
8. 实际推理请求不读取该开关。即使某模型未显示，只要符合账号白名单且存在可用账号，Gateway 仍可通过现有 Translator 和 Executor 完成跨协议调用。

Manager 的 `VisualConfigEditor.tsx` 展示开关，`visualConfig.ts` 定义字段和默认值，`useVisualConfig.ts` 负责与 YAML 字段双向转换，并只在字段修改时写回以保留未知配置项。Manager 不读取 Model Registry，也不自行过滤模型，实际行为以 Gateway 热更新结果为准。

### 4.4 API Key 官方客户端兼容

#### 目标与范围

部分 API Key 中转账号会根据身份 Header、Body 默认值、Beta、Session、缓存键或端点判断客户端。Pro 按账号保存官方客户端 Profile，在不改变账号调度和模型映射的前提下，让普通请求和账号测试共用同一构造链，并支持独立启停和热更新。

| 平台 | 账号类型 | 官方客户端 Profile | Gateway 行为 |
|---|---|---|---|
| Anthropic | API Key | `claude-desktop-2.1.215-v1` | Claude Code/Claude Desktop Messages、Count Tokens 和工具请求兼容 |
| OpenAI | Responses/Codex API Key | `codex-desktop-0.145.0-alpha.18-v1` | Codex Desktop Responses、SSE 和 Compact 兼容 |

配置缺失或关闭时保持原 API Key 行为；OAuth、xAI 和 OpenAI Chat Completions 兼容账号不进入该逻辑。明确识别为官方 Claude/Codex 客户端时保留其当前链路；其他客户端及账号测试使用兼容 Profile。Profile ID、Provider 和 TLS 策略由 Gateway 注册表校验，未知或不匹配配置 fail-closed，不静默回退。

#### 配置和执行

兼容配置保存在 Auth Attributes 的 `official_client_compatibility` JSON 中，配置合成、Management API 和 watcher 热更新共享同一 Profile 注册表。切换开关或 Profile 不改变 Auth ID；请求先完成账号模型映射，再进入 Profile Body、Header、Transport 和路由构造。

账号测试通过 Management API 固定 `auth_index`，复用 4.2 的生产执行链和 Profile；停用账号只在本次请求中临时执行。Manager 的“配置面板 → Pro 配置”提供兼容区域：

- 读取 Anthropic/OpenAI API Key 账号实时详情，并按全部、已启用、已关闭筛选。
- 展示账号、平台、开关、Profile 和状态；Anthropic 显示 Claude Code，OpenAI 显示 Codex Desktop。
- 根据 Gateway 能力决定是否允许操作；旧 Gateway 不支持时隐藏或禁用开关。
- 使用版本号、操作 ID 和幂等键定点更新，写入后回读确认；后续失败时恢复原配置。
- 配置只保存在 Gateway 配置和 Auth Attributes，不新增 Manager SQLite 字段，不参与账号列表投影，也不影响旧 Provider 页面保存未知字段。

#### Anthropic API Key

- `/v1/messages` 保留用户 System、Messages、工具和缓存控制，Profile 负责公开身份 System、JSON `metadata.user_id`、Session 和固定 Header。
- Messages 固定携带 `context-1m-2025-08-07`；普通请求使用六项基础 beta，结构化输出用 `structured-outputs-2025-12-15` 替换末尾 fallback beta；Count Tokens 在基础 beta 后追加 `token-counting-2024-11-01`。
- Messages 同时发送 `x-api-key` 和 `Authorization: Bearer`；Profile 身份 Header 不能被账号自定义 Header 覆盖，Count Tokens 保持端点专属鉴权和 beta 规则。
- Metadata 是包含 `device_id`、`account_uuid`、`session_id` 的 JSON 字符串；计费 System 块使用 Claude Desktop 版本和入口标识，不写入旧 `cch` 字段。
- 普通请求只归一实际携带的已知工具，通过请求级双向映射还原 JSON 和 SSE 工具调用，不注入完整工具基线。
- 账号测试使用 Claude Code 形态：文本数组内容、`system`、`max_tokens: 512`、`stream: true` 和完整 27 个 Claude Code 工具，汇总流式 SSE 后展示摘要。
- 命中 Profile 时不叠加旧 `applyCloaking`；首版使用现有 Transport，不提供非空 TLS Profile。

#### OpenAI API Key

- Profile 生成 `originator`、`x-codex-*`、Installation/Session/Thread/Window/Turn、Client Metadata、缓存键和必要 Body 默认值。
- 官方客户端或 WebSocket 链路保持原行为；其他客户端和账号测试强制使用 HTTP/SSE Codex Responses Profile。
- 根 Base URL 归一到 `/v1/responses`；已有 `/v1` 时追加 `/responses`，其他路径前缀继续保留。兼容关闭时维持原 `/responses` 逻辑。
- `/v1/responses/compact` 使用独立的非流式 JSON allowlist 和 `Accept: application/json`，不复用普通 Responses 的 stream、缓存和 Client Metadata；官方 Codex Desktop 手工压缩仍按普通 Responses 的 `request_kind=compaction` 信号处理。
- 账号测试复用 Codex mimic Responses 请求；Compact 测试固定账号并使用独立构造，不能根据普通 Responses 成功推断 Compact 可用。

#### 安全边界

- 不修改 Translator、账号调度、模型映射、全局 Transport 或普通 API Key 的 Header 优先级。
- 不复制隐藏 Prompt、产品 Memory、账号状态或完整 TLS/HTTP2 帧级指纹；`tls-profile` 首版只接受空值。
- 页面、日志、错误、测试夹具和脱敏抓包不得包含 API Key、OAuth Token、Cookie 或完整鉴权 Header。
- `.references/sub2apiplus` 只用于交叉参考；Profile 契约和 Golden Fixture 保存在 `docs/official-client-compatibility/`。

详细架构、配置契约、实施顺序和测试矩阵见 [API Key 官方客户端兼容实施方案](docs/api-key-official-client-compatibility-implementation-plan.md)；冻结的客户端版本、抓包结论和脱敏 Golden Fixture 见 [Profile 契约](docs/official-client-compatibility/profile-contracts.md)。

### 4.5 统一 IP 库代理功能

该功能待开发，分两个阶段实施。CLIProxyAPI 已支持全局代理和账号级代理，Pro 需要补齐统一 IP 库、账号引用和 OAuth 授权阶段的代理选择。

#### 第一阶段：统一 IP 库与账号引用（方案 A）

第一阶段尽量不修改 CLIProxyAPI 核心，主要由 CPA-Manager-Plus 实现：

1. 新增“IP 管理”页面，统一管理 HTTP、HTTPS、SOCKS5 和 SOCKS5H 代理。
2. 支持标准添加、批量导入、编辑、删除、单条/批量连通性测试和账号引用数统计。
3. 代理在 Manager SQLite 中使用稳定 `proxy_id`；API、OAuth 和 Vertex 账号在添加、编辑时从 IP 库选择代理。
4. Manager 将 `proxy_id` 解析为代理 URL：API 账号写入 `proxy-url`，OAuth 和 Vertex 写入认证文件的 `proxy_url`，Gateway 继续按原方式执行代理。
5. OAuth 向导保存所选 `proxy_id`，授权成功后将代理 URL 绑定到认证文件，后续模型请求和 Token 刷新使用该代理。
6. 修改代理时级联更新全部引用账号；代理被引用时禁止删除；代理密码加密保存并脱敏回显。
7. 账号代理选项分为“继承全局代理”“强制直连”和“选择 IP 库代理”，账号设置优先于 Gateway 全局 `proxy-url`。

第一阶段限制：OAuth 授权码首次换取令牌、设备码申请和轮询仍继承 Gateway 全局代理；用户登录和授权确认始终由浏览器或操作系统的代理设置决定出口。

#### 第二阶段：OAuth 会话级代理（方案 B）

1. Manager 启动 OAuth 时将账号选择的代理传给 Gateway。
2. Gateway 在对应 OAuth Session 保存代理 URL，仅对该次授权会话生效。
3. 授权码换令牌、设备码申请和轮询使用会话代理；授权成功后继续将同一代理写入账号 `proxy_url`。
4. 不临时修改 Gateway 全局代理，避免并发 OAuth 授权串用代理。

第二阶段需要修改 Gateway 的 OAuth 启动参数、Session 和各 Provider 后端 OAuth Client，应作为独立变更集实施和回归；浏览器授权页仍不受 Manager 所选代理控制。

| 对比项 | sub2api | CPA Pro 第一阶段 | CPA Pro 第二阶段 |
|---|---|---|---|
| 代理库位置 | 代理库与账号、执行在同一后端 | 代理库在 Manager，请求由 Gateway 执行 | 与第一阶段相同 |
| 账号引用 | 账号直接保存 `proxy_id` | Manager 保存 `proxy_id`，再同步代理 URL | 与第一阶段相同 |
| 代理修改 | 运行时读取 `proxy_id`，修改一处生效 | Manager 级联同步全部引用账号 | 与第一阶段相同 |
| OAuth 后端阶段 | 选中代理用于换令牌或设备码轮询 | 继承 Gateway 全局代理 | 使用 OAuth Session 独立代理 |
| 浏览器授权 | 由浏览器代理决定 | 与 sub2api 相同 | 与 sub2api 相同 |
| 上游合并影响 | 无双组件合并问题 | Gateway 基本不改，侵入较小 | 修改 Gateway OAuth 链路，合并成本较高 |

sub2api 的代理库是运行时唯一数据源；CPA Pro 为保持 Manager 与 Gateway 分层，由 Manager 管理 `proxy_id`，Gateway 继续只识别代理 URL。第一阶段完成统一管理和账号引用，第二阶段补齐 OAuth 后端授权阶段的独立代理。

## 5. 实现文件与上游合并关注点

| 功能 | CPA-Manager-Plus | CLIProxyAPI |
|---|---|---|
| 统一账号后端 | `internal/http/controller/proaccount/`、`internal/model/pro_account*.go`、`internal/repository/proaccount*/`、`internal/service/proaccount*/`、`internal/worker/pro_account_*` | `sdk/cliproxy/allowed_models.go`、`sdk/cliproxy/auth/allowed_models.go`、`sdk/cliproxy/auth/credential_draft.go` |
| 统一账号前端 | `apps/web/src/features/accounts/`、`apps/web/src/pages/AccountsPage.tsx`、`apps/web/src/services/api/proAccounts.ts` | — |
| 连通性与定时测试 | `internal/service/proaccounttest/`、`internal/service/proaccountscheduledtest/`、`internal/http/controller/proaccountscheduledtest/`、`internal/repository/proaccountscheduledtest/`、`AccountTestModal.tsx`、`AccountScheduledTestsModal.tsx` | `internal/api/handlers/management/account_connectivity.go`、`sdk/api/handlers/handlers.go`、`sdk/cliproxy/executor/types.go`、`sdk/cliproxy/auth/conductor.go` |
| 按协议显示模型 | `VisualConfigEditor.tsx`、`useVisualConfig.ts`、`visualConfig.ts` | `internal/config/sdk_config.go`、`internal/watcher/diff/config_diff.go`、`internal/registry/model_registry.go`、`internal/registry/protocol_models.go`、各协议模型 Handler |
| 官方客户端兼容 | `OfficialClientCompatibilityPanel.tsx`、`VisualConfigEditor.tsx`、`ConfigPage.tsx`、`internal/service/proaccountgateway/`、`internal/service/proaccountlifecycle/` | `internal/officialclient/`、`internal/config/`、`internal/watcher/synthesizer/`、`internal/runtime/executor/*official_client*`、Claude/Codex Executor |

统一账号相关的高频上游接入文件包括 Manager 的 `internal/app/context.go`、`internal/http/router/router.go`、`internal/repository/sqlite/migrate.go`、`internal/store/store.go`、`cmd/cpa-manager-plus/main.go`，以及 Gateway 的 Management Handler、`oauth_sessions.go`、`vertex_import.go`、`internal/watcher/synthesizer/*`、四种 Token Store、`sdk/cliproxy/service.go` 和 `internal/api/server.go`。

合并上游时重点确认：

1. `/v0/pro/*` 仍在 Management API 代理之前本地匹配，旧账号路由仍可访问。
2. SQLite 迁移保持增量兼容，恢复 Worker 和数据库备份仍可工作。
3. Gateway 继续回显 `auth_index`、来源定位、草稿状态、模型规则和能力标记。
4. 四种 Token Store 的 Disabled 草稿语义一致，普通已删除凭证不会被重建。
5. 连通性测试仍固定指定账号，且不改变正常调度、冷却或停用状态。
6. 协议归组、执行元数据、Profile 和 Auth 选择逻辑仍进入正确的生产链路。

## 6. 验证与验收

### 统一验证命令

- Manager 后端：在 `components/cpa-manager-plus/apps/manager-server` 执行 `go test ./...`。
- Manager 前端：在 `components/cpa-manager-plus/apps/web` 执行 `npm test`、`npm run type-check`、`npm run lint` 和 `npm run build`。
- Gateway：在 `components/cliproxyapi` 执行 `go test ./...` 并完成构建。

对应 Go 测试集中在实现目录的 `*_test.go`，前端测试集中在相关组件的 `*.test.tsx` 或 `*.test.ts`。

### 专项回归重点

- 统一账号：存量同步、API/OAuth/Vertex 创建、编辑迁移、启停删除、模型规则、用量聚合、恢复 Worker、数据库备份和旧路由兼容。
- 模型规则：`alias == name` 不得扩大模型列表、注册集合或调度范围；`alias != name` 的有效别名仍可随白名单模型使用。
- 连通性测试：指定 Auth 固定执行、停用账号临时执行、协议映射、错误脱敏、定时计划和历史清理。
- 协议模型列表：配置解析与热更新、协议归组、缓存与可用性、各协议目录、跨协议调用、插件/未知来源和 Home 边界。
- 官方客户端兼容：Profile 校验、热更新、Header 保护、Claude Messages/Count Tokens/工具 SSE、Codex Responses/SSE/Compact/WebSocket。
- 官方客户端实测：根据上游实际收到的 Header、Body、beta、Session、工具和路由验收，不能只参考本地 Usage 页面。
- 兼容场景：开启后 Anthropic `anyrouter.top` 的普通消息和测试应成功；OpenAI `anyrouter.top`、`free.lyclaude.site` 的普通消息和测试应成功。同时回归关闭开关、官方客户端绕过、xAI/OAuth 隔离、自定义 Header 保护、敏感信息脱敏及 Manager 停止后的 Gateway 独立转发。

### 当前版本验收标准

1. 一个页面可以查看和管理全部平台账号。
2. 添加账号先选平台，再选 OAuth 或 API Key。
3. OpenAI API Key 可以自动识别 Responses 和 Chat Completions 能力。
4. OAuth 和 API Key 账号都可以配置允许模型和模型映射。
5. 每个账号都可以测试连通性，并显示明确失败原因。
6. 开启官方客户端兼容后，Claude 和 Codex 请求通过抓包对比测试。
7. 开启按协议显示模型列表后，各目录只显示对应账号协议组；未知、插件及 Home 模型保持全量，列表隐藏不影响手工调用。
8. 每个账号和 Key 都能查看 Token、请求次数和官方成本估算。
9. Manager 故障不影响 Gateway 继续处理模型请求。
10. 两个组件可以独立升级和回滚。

## 7. 实施顺序

既有功能按以下依赖推进：建立 Pro 主项目、两个 fork 和统一 Compose → 定义 Management API、用量事件和协议模型契约 → 实现统一账号、添加向导、用量与连通性测试 → 实现 OpenAI 自动探测和账号模型配置 → 增加 Claude/Codex Profile → 实现协议归组和模型列表过滤 → 完成集成及升级测试后发布。

后续代理功能先实施 Manager 统一 IP 库和账号引用，再以独立变更集增加 Gateway OAuth 会话级代理，最后完成跨组件、并发授权、升级与回滚测试。

## 8. 默认目录与部署

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

产品名称使用 `CLIProxyAPI Pro`；开发目录、仓库名、安装目录、Compose 项目名和 Docker 镜像名称统一使用小写 `cliproxyapi-pro`。
