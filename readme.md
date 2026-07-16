# CLIProxyAPI Pro 需求与开发说明

## 1. 项目定位

CLIProxyAPI Pro 基于两个开源项目开发：

- [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)：负责模型请求、协议转换、账号调度、OAuth、API Key 和统一下游 Key。
- [CPA-Manager-Plus](https://github.com/seakee/CPA-Manager-Plus)：负责管理页面、账号管理、统计、成本、监控和运维。

开发工作区初始使用两个官方仓库作为 Git 子模块；建立自有 fork 后替换子模块地址，并继续保留官方仓库作为上游更新来源。

整体结构：

```text
CLIProxyAPI Pro
= CPA-Manager-Plus 主要改造
+ CLIProxyAPI 少量运行时改造
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

开发原则：

1. 分别维护 CLIProxyAPI fork 和 CPA-Manager-Plus fork。
2. 使用一个 CLIProxyAPI Pro 主项目锁定两个组件版本。
3. 使用 Docker Compose 一键安装，两个组件保持独立容器和数据目录。
4. 不把两个项目合并成一个 Go 工程、二进制或进程。
5. Pro 功能尽量通过新增模块和少量接入点实现，方便继续合并上游更新。

## 3. 功能需求

### 3.1 统一账号管理

#### 目标和范围

提供一个统一账号页面，集中完成账号的查看、添加、编辑、启停、删除、测试、模型配置和用量查看。

统一账号是 CPA-Manager-Plus 对 CLIProxyAPI 配置项和认证文件的归一化管理视图，不改变 CLIProxyAPI 现有底层结构。不引入分组订阅、倍率、余额或商业计费。sub2apiplus 只作为页面布局、交互流程和协议处理的参考；Pro 追求核心体验等效，不承诺复制其商业调度和计费语义。

#### 平台和认证方式

| 平台 | Pro 首期添加方式 | CLIProxyAPI 底层映射 |
|---|---|---|
| Anthropic | OAuth、API | OAuth 认证文件、`claude-api-key` |
| OpenAI | OAuth、API | OAuth 认证文件、`codex-api-key` 或 `openai-compatibility` |
| Gemini | OAuth、API、Vertex | `gemini-cli` 插件、`gemini-api-key`、Vertex Service Account |
| Antigravity | OAuth | Antigravity OAuth 认证文件 |
| Grok / xAI | OAuth | xAI OAuth 认证文件 |

中转站按上游协议归类，不按模型名称归类：

- OpenAI 格式的 Grok、Gemini 或其他模型中转站，从 OpenAI API 入口添加。
- Gemini API 入口只用于 Gemini 原生协议。
- Grok / xAI 和 Antigravity 的普通添加页面不提供 API 类型。
- 已有 `xai-api-key`、`interactions-api-key` 等配置仅作为高级兼容能力保留。

#### 统一添加流程

```text
选择平台
  -> 选择认证方式
  -> 填写或完成授权
  -> 探测账号能力
  -> 同步并选择允许模型
  -> 配置模型别名和映射
  -> 连通性测试
  -> 保存
```

页面不暴露 Codex、Interactions、OpenAI Compatibility 等内部 Provider 名称。OAuth 流程保存认证文件，API 和 Vertex 流程写入对应配置。

允许模型必须在添加账号时即可配置，保存后也可重新编辑。同步失败时允许手工添加模型，但仍需通过连通性测试。

#### 账号管理首页

首页使用单个高密度表格展示所有平台账号，不使用卡片列表。表格至少包含：

| 列 | 内容 |
|---|---|
| 账号 | 名称、邮箱或脱敏凭证摘要 |
| 平台/类型 | 平台、OAuth/API/Vertex、协议或账号等级 |
| 状态 | 启用、健康、错误或需要重新授权 |
| 允许模型 | 模型数量和部分模型标签 |
| 用量窗口 | 官方配额进度和本地用量 |
| 最近活动 | 最近使用、最近测试和最近错误 |
| 操作 | 编辑、测试、启停、删除和更多操作 |

页面顶部提供账号搜索，平台、认证类型、启用状态和健康状态筛选，以及手动刷新、自动刷新、添加账号和批量启停/测试/删除。

不照搬 sub2apiplus 的容量、分组、隐私、倍率和商业计费字段。

#### 底层账号映射

每个持久化认证文件或 API 配置项在统一页面中显示为一个账号。插件生成的运行时虚拟认证不单独形成可编辑账号。

归一化账号至少保留：平台、认证方式、底层来源类型、稳定账号 ID、账号名称或邮箱、启用状态、健康状态、可用模型和可选过期时间。编辑、启停和删除操作根据底层来源分发到现有 Management API。

sub2apiplus 使用数据库 `account.id` 关联配额、用量和操作。Pro 在 CPA-Manager-Plus SQLite 中新增 `pro_accounts`，由 Manager 生成 UUID 作为统一账号的稳定主键；不要求把该 UUID 写入 CLIProxyAPI 的现有配置格式。另建 `pro_account_bindings` 保存一个账号与底层凭证的当前及历史关联：

- `pro_accounts` 保存平台、认证方式、名称、状态、模型规则、时间和用量摘要。
- `pro_account_bindings` 保存来源类型、当前和历史 `auth_index`、来源定位信息、绑定有效期、绑定状态及首次/最近发现时间。
- 首期字段至少包括：`id`、`platform`、`auth_type`、`source_type`、`name`、`email`、`enabled`、`health_status`、`last_error`、`allowed_models`、`model_mapping`、`last_used_at`、`last_tested_at`、`expires_at`、`created_at` 和 `updated_at`；绑定表至少包括 `pro_account_id`、`auth_index`、`source_locator`、`source_fingerprint`、`binding_status`、`is_current`、`valid_from`、`valid_to` 和时间字段。
- 另建 `pro_account_drafts` 保存跨 Management API 操作的持久化状态机，至少包括 `operation_id`、幂等键、操作类型、关联账号、当前状态、状态版本、重试次数、清理截止时间、补偿动作、错误摘要和时间字段；草稿清理不能只依赖前端内存状态。
- `auth_index` 只作为 Gateway 运行时凭证标识，不假设它永久稳定：OAuth 文件通常由认证类型和绝对路径生成，API 凭证会包含 Base URL 和 Key，因此更换 Key 或地址可能改变它。
- 通过统一页面编辑账号时，先更新绑定并保留旧标识，历史用量按该账号的全部绑定标识聚合；账号 UUID 不变。外部直接修改配置导致无法唯一匹配时不得猜测合并，应标记为待确认。
- `source_fingerprint`、`identity_hash`、邮箱、Key 片段或 Hash 只能作为非唯一的匹配提示和人工重绑依据，不能作为账号主键，也不能据此自动合并两个账号。
- 认证目录、容器挂载点、文件名、Key 或 Base URL 变化可能导致绑定漂移。检测到同一部署中大量绑定同时失效时，应进入批量漂移确认和批量重绑流程，不要求管理员逐账号处理；文件路径漂移和 API 凭证漂移必须分别识别。
- 存量同步必须幂等：同一轮快照使用 `source_type + source_locator + auth_index` 形成发现键；精确命中当前绑定时更新快照，首次导入且无候选时创建新 UUID，存在历史定位、指纹或身份候选但无法唯一确认时进入 `pending_confirmation`，不得因重复同步创建多个账号。`ResolveBinding` 返回精确命中、新建、待确认和冲突状态，并保留候选及原因码。
- 历史用量必须按事件时间归属：使用 `usage_events` 的事件时间与绑定的 `[valid_from, valid_to)` 做时间连接，不能把同一 `auth_index` 在所有时期的事件无条件归给当前账号。绑定变更、路径重用或人工重绑后应触发 Pro 账号汇总回填；现有预聚合表不能直接替代该时间归属。原始事件已过保留期时保留归属质量标记，不猜测补齐。
- 新建 `openai-compatibility` 账号时每个 Provider 默认只保存一个 API Key。存量多 Key Provider 在统一页面按 Key 展示为多条账号，但底层仍保留一个共享 Provider，不物理拆分；模型、Header、Base URL 等 Provider 级配置明确标记为共享。

Manager 保存稳定 UUID、期望启用状态、模型规则、绑定历史和操作状态；Gateway 返回实际启用状态、认证状态、冷却状态和运行错误；前端状态由 Manager 统一归一化。Manager 数据库是统一账号 ID 和绑定历史的持久化来源，部署备份必须包含该 SQLite 数据库。SQLite 必须纳入版本化迁移、升级回滚、定期备份、恢复演练和数据目录持久化要求。CLIProxyAPI 仅继续提供凭证执行、调度和 Management API，不承担 Pro 账号表。

#### OpenAI API 协议探测

用户只需填写 Base URL 和 API Key。系统优先探测 `/v1/responses`：

- Responses 可用时，使用 `codex-api-key`。
- Responses 不可用时，测试 `/v1/chat/completions`，成功后使用 `openai-compatibility`。
- 两个协议都支持时只使用 Responses，不创建两个账号或两份并行配置。
- 两个协议都不可用时不保存。

失败时区分认证、地址、网络、协议和上游服务错误。默认自动选择协议；高级选项可强制 Responses 或 Chat Completions。

新 API Key 的处理顺序为：Manager 仅在内存中接收 Base URL 和 Key -> 通过 `ProbeCandidate` 使用临时凭证和候选模型规则获取模型/执行协议探测 -> 确定底层类型 -> 创建停用或不可调度配置 -> 获取运行时 `auth_index` -> 通过 `/api-call` 执行最终连通性测试 -> 启用配置。预探测阶段不依赖 `auth_index`，白名单使用尚未持久化的草稿规则在内存中校验；最终测试才使用已持久化草稿的 `auth_index`。探测失败时不写入可调度配置；用户明确选择保留时，只保存为停用账号。`ProbeCandidate` 可由 Manager 内存探测实现，或使用不持久化、不进入调度且不记录凭证明文的窄 Gateway 探测接口。

探测请求结构参考 sub2apiplus `backend/internal/service/openai_apikey_responses_probe.go`（创建交互见 `frontend/src/components/account/CreateAccountModal.vue`），但错误判定采用以下更严格的 Pro 三态规则，不原样复制参考项目对其他非 `2xx` 响应“保守按支持处理”的行为：

1. 使用带必选 function tool 的非流式 Responses 请求，确认端点和工具调用都可用。
2. 探测模型优先使用已同步模型映射中的具体上游模型；没有列表时使用项目默认测试模型。
3. `404/405` 判定为 Responses 不支持；`2xx` 只有在响应中出现 `function_call` 时才判定可用。
4. 其他 HTTP 错误只能说明端点存在但本次无法完成能力判断，不得据此判定不支持；网络错误保持未知。探测结果必须区分“明确不支持、支持、未知”。
5. 只有 Responses 明确不支持时才继续测试 Chat Completions。两个协议都支持时只选择 Responses；Responses 未知时提示重试或允许保存为停用账号，不自动创建错误类型。

编辑存量 OpenAI API 账号时重新探测。如果底层类型发生变化，先在不修改旧配置的情况下完成新凭证预检，再创建停用/不可调度的新配置并执行最终测试，成功后删除旧配置并启用新配置。任一步失败时执行补偿并保留旧配置；`pro_account_id` 保持不变。若某类底层配置暂不支持停用草稿，必须依赖写入前预检和失败补偿，不能把多次 Management API 调用当作数据库原子事务。

#### 添加事务和取消清理

sub2apiplus 在 OAuth 换取凭证后再创建数据库账号；CLIProxyAPI 的 OAuth/插件流程可能会直接保存认证文件。Pro 按以下规则适配：

- OAuth 授权成功后新认证文件必须在首次保存时就是草稿/停用状态，完成模型配置和连通性测试后再启用；不能先保存为可调度账号再由 Manager 补发停用请求。
- 用户取消、授权超时或保存失败时，删除本次新建的草稿认证文件或配置项。
- 连通性测试失败时默认不启用；用户可明确选择“保存为停用账号”以保留配置。
- 添加和迁移流程使用持久化状态机，状态至少包括：`draft_created`、`credential_saved_disabled`、`probed`、`models_configured`、`tested`、`enabled`、`cancelled`、`compensating` 和 `failed`。每次状态转换使用 `operation_id`、幂等键和状态版本，服务重启后可继续清理或补偿。
- CLIProxyAPI 的 `PostAuthHook` 负责读取明确的草稿意图，写入 `pro_draft=true`，并在首次持久化前设置 `Disabled` 和禁用状态。草稿意图必须与 OAuth Session 关联，不能只依赖授权发起页面的内存或单次 Query 参数；插件 OAuth 在后续状态轮询中完成保存时仍必须恢复同一草稿意图。
- CLIProxyAPI 的 File、Object、PostgreSQL 和 Git 四种 Token Store 都必须允许带 `pro_draft=true` 的 Disabled 凭证首次持久化。普通 Disabled 凭证在目标不存在时仍保持现有跳过行为，防止已删除认证被状态同步意外重建。
- 草稿能力是一个运行时接入边界，但可能涉及 Hook 装配、OAuth Session 草稿意图传递和多个 Store 的共享持久化判断；不能将其理解为单一文件补丁。

状态机以 `enabled`、`cancelled` 和清理完成后的 `failed` 为终态，其他状态均允许恢复扫描。`operation_id` 和幂等键必须唯一，状态更新使用版本比较防止并发覆盖；Manager 启动和定时任务扫描超过截止时间的非终态操作，按最后成功步骤继续执行或进入 `compensating`。补偿失败保留可重试记录和人工处理入口，不能静默删除证据。

#### Gemini 实现规则

- OAuth：通过官方 `gemini-cli` 插件授权，自动识别邮箱、GCP 项目和账号等级。Google One 和 Code Assist 是授权后的识别结果，不是添加前的独立入口。
- API：填写可自定义的 Base URL 和 API Key，底层使用 `gemini-api-key`。
- Vertex：上传 Google Cloud Service Account JSON，并配置项目和地区。

#### 其他平台实现规则

- Anthropic：OAuth 完成授权并保存认证文件；API 填写 Base URL 和 API Key，底层使用 `claude-api-key`。
- Antigravity：只提供 OAuth 添加方式。
- Grok / xAI：只提供 OAuth 添加方式；OpenAI 格式的 Grok 中转站从 OpenAI API 入口添加。

#### 账号模型和用量

所有统一账号都支持：

- 同步项目最新支持模型。
- 同步上游实际支持模型。
- 模型白名单。
- 模型别名和映射。
- 手工添加模型。

sub2apiplus 使用 `model_mapping` 同时表示白名单和映射：`model -> model` 是白名单，`alias -> upstream` 是映射，空映射表示允许全部。Pro 保留该语义，但适配 CLIProxyAPI 的 `allowed_models` 和 `models` 结构：

1. `allowed_models` 存储客户端可请求的模型名；空或缺失表示允许全部。
2. `models` 存储客户端别名到上游模型的映射。有映射时，映射的客户端别名自动加入有效白名单。
3. 请求处理顺序固定为：使用客户端模型名检查白名单，通过后再执行别名映射并请求上游。
4. 模型列表、账号调度、连通性测试和真实请求使用同一套白名单与映射规则。
5. 映射源允许精确名称或仅在末尾出现一个 `*`；映射目标不允许通配符，与 sub2apiplus 保持一致。
6. 插件虚拟认证继承物理账号的白名单和映射，不允许单独编辑。

CLIProxyAPI 已有账号级和 Provider 级 `excluded-models`，可在作用域相同的配置项上复用，但负向黑名单不能替代严格的正向白名单：按已知模型列表计算差集会在同步失败、动态插件模型或上游新增模型时意外放行未知模型；OpenAI Compatibility 的模型、Header 和 Base URL 还可能被同一 Provider 多 Key 共享。因此 Gateway 需要增加最小的账号级运行时白名单接入，至少覆盖 OAuth 认证文件和共享 Provider 下的独立账号。

白名单应接入 Auth/模型 Registry 的统一模型注册和调度路径。构建 Registry 时先生成含别名和前缀的客户端可见模型名，再按 `allowed_models` 过滤；处理请求时先用客户端请求模型检查白名单，通过后再解析为上游模型。模型列表、账号调度和真实请求复用同一判断，不在各 Executor 重复实现。`/v0/management/api-call` 是任意上游 HTTP 工具，不经过模型 Registry；Manager 使用它进行连通性测试或协议探测时，必须自行确认测试模型属于账号有效白名单。现有 `excluded-models` 可作为配置落地优化，但不能改变账号级白名单语义。`allowed_models` 和通用草稿保存能力应尽量以不包含 Pro 业务概念的形式向 CLIProxyAPI 上游提交 PR，fork 补丁作为合并前的过渡实现。

`pro_accounts` 是模型规则的管理端权威来源。保存规则时，Adapter 根据来源类型将规则写入 OAuth 认证文件元数据、独立 API Key 配置或 Gateway 新增的账号级运行时字段；共享 OpenAI Compatibility Provider 的 Base URL、Header 和模型目录仍保持 Provider 级共享，独立 Key 的 `allowed_models` 不得覆盖同 Provider 的其他账号。Gateway 回显当前生效规则和规则版本，Manager 只有在写入、重载和回读一致后才提交本地状态；任一步失败时按草稿状态机补偿并保留旧规则。

模型同步顺序为：上游实际模型列表 -> 项目内置模型目录 -> 用户手工模型。同名模型去重，保留用户已选白名单和自定义映射；上游不支持模型列表时不视为账号错误。

账号只参与允许模型的调度。账号详情用简单窗口显示配额、重置时间、输入/输出/缓存/推理 Token、请求次数、失败次数和官方价格估算成本。

账号总览至少显示：平台和认证方式、名称或邮箱、启用和健康状态、可用模型、配额和 Token 用量、估算成本、最近使用、最近测试和最近错误。

#### 行内用量窗口

用量窗口直接显示在账号表格的每一行，不要求用户先进入详情页。窗口采用紧凑的两层结构：

```text
5h  [======----] 62%  6d 11h 后重置
7d  [=---------]  4%  4d 8h 后重置

663 次  111.2M Token  估算 $98.20  查询  刷新
OpenAI OAuth 支持时：重置次数 2  重置
```

展示规则：

1. 官方配额可用时，显示窗口名称、使用百分比、进度条和重置倒计时。
2. 同一账号有多个窗口时紧凑纵向排列，例如 `5h`、`7d` 和模型专项窗口。
3. 窗口下方显示当前统计周期的请求数、Token 和官方价格估算成本。
4. 支持按需查询官方配额和刷新本地统计。
5. 官方数据优先，其次使用响应头被动采样，最后使用 CLIProxyAPI 本地用量统计。
6. 本地统计不是官方剩余配额，必须明确区分，不得模拟或虚构官方配额。
7. 无数据时显示“暂无配额数据”，不将未知配额显示为零。
8. 加载、无数据和错误状态保持稳定尺寸，不引起表格抖动。
9. 错误仅显示简短摘要，完整信息放在 Tooltip 或详情中。
10. 截图中的 `A/U` 双成本属于 sub2apiplus 的账号成本和用户计费成本，Pro 不复制该商业计费语义，只显示单一官方价格估算成本。
11. OpenAI OAuth 在上游明确支持时可显示剩余 reset credits，并允许管理员主动消耗一次 credit 重置官方窗口；该操作必须进行能力检测和二次确认，防止重复点击，不支持或状态未知时不显示为零。

用量页面布局、字段命名、缓存时长和懒加载方式参考 sub2apiplus `GET /admin/accounts/:id/usage?source=passive|active&force=true`、`AccountUsageCell.vue`、`UsageProgressBar.vue`、`AccountTodayStatsCell.vue` 和 `account_usage_service.go`；数据来源、错误语义以及主动/被动查询行为以本节定义的 Pro 统一契约为准，不假设参考项目已在所有平台实现相同语义：

- `passive`：只读已有响应头快照和本地用量，不主动请求上游。
- `active`：使用账号凭证主动查询官方配额；`force=true` 跳过成功结果缓存。
- 返回结构统一包含 `source`、`updated_at`、配额窗口、本地统计和可机器识别的错误码。
- 官方配额成功结果后端缓存 3 分钟，可恢复错误缓存 1 分钟；前端行内数据缓存 5 分钟。
- 页面自动刷新只刷新账号列表、`passive` 快照和本地统计，不循环主动请求上游配额。
- 桌面端在用量列可见时加载；移动端使用 `IntersectionObserver` 在行接近视口时懒加载，避免一次请求全部账号。

上述 `passive`、`active` 和 `force` 是 Pro BFF 重新定义的统一契约：参考项目当前并非所有平台都支持统一的被动查询，`force=true` 也并非所有分支都会绕过成功缓存。Pro 后端必须按本节规则归一化，不把平台差异直接暴露给前端。

所有账号类型返回同一响应结构，但不强制所有平台具备官方主动查询能力。仅支持本地统计的账号收到 `active` 请求时不调用不存在的上游接口，返回 `source=local`、本地统计、空官方窗口和机器可识别的 `official_usage_unsupported`；官方状态未知时使用 `official_usage_unknown`，不得把不支持或未知显示为零额度。

本地统计按管理端时区的当日 `00:00` 至次日 `00:00` 批量聚合，包含请求数、成功/失败数、输入/输出/缓存/推理 Token 和官方价格估算成本。该富统计需要基于 CPA-Manager-Plus 的 `usage_events`、账号模型汇总和 `pro_account_bindings` 新增 Pro 账号聚合能力，不假设 sub2apiplus 行内统计已经提供全部字段。未知模型成本显示为“-”，不按零成本处理。

OpenAI reset credits 是独立的账号能力和上游副作用操作，不属于普通用量刷新，也不同于 CLIProxyAPI 清理本地冷却状态的 `reset-quota`。首期放在 P5：仅对支持该能力的 OpenAI OAuth 账号提供剩余次数查询和重置操作；失败不影响普通用量窗口和账号列表。P1-P4 页面不渲染该按钮；P5 也只有在能力状态为 `supported` 时显示，`unknown` 和 `unsupported` 均隐藏操作。

首期按账号类型展示：

| 账号类型 | 用量窗口 |
|---|---|
| OpenAI OAuth | 由 `ProAccountUsageService` 通过 Adapter 复用现有 Codex `wham/usage` 查询，显示官方窗口和本地统计 |
| OpenAI API | 本地统计 |
| Anthropic OAuth | 由 `ProAccountUsageService` 通过 Adapter 复用现有 Anthropic OAuth usage 查询，显示官方滚动窗口和本地统计 |
| Anthropic API | 本地统计 |
| Gemini OAuth / API / Vertex | 首期显示本地统计；官方配额只在 `gemini-cli` 插件或上游明确提供稳定契约后增加 |
| Antigravity OAuth | 由 `ProAccountUsageService` 通过 Adapter 复用现有配额查询，按模型显示额度和重置时间 |
| Grok / xAI OAuth | 由 `ProAccountUsageService` 通过 Adapter 复用现有 xAI 配额/账单探测和响应头快照，加本地统计 |

#### 旧入口迁移

侧边栏只显示新的 `/accounts` 账号管理。原 `/ai-providers`、`/auth-files`、`/oauth` 路由和页面暂时保留，用于兼容旧书签、旧配置、插件 Provider 和尚未覆盖的高级字段。

旧页面不再作为一级菜单，可从“账号管理 -> 更多操作 -> 高级管理”进入。统一页完成功能对等后，再单独评估删除旧页面。

#### 实施结构和顺序

CPA-Manager-Plus 新增 `ProAccountService`、`ProAccountSyncService`、`ProAccountUsageService`、`ProAccountProbeService` 和底层账号 Adapter。统一账号 BFF 使用 Manager 私有的 `/v0/pro/accounts` 命名空间，与 Gateway `/v0/management/*` 透传和插件 Management 命名空间分离：

```text
GET/POST       /v0/pro/accounts
GET/PUT/DELETE /v0/pro/accounts/:id
POST           /v0/pro/accounts/:id/test
GET            /v0/pro/accounts/:id/usage
POST           /v0/pro/accounts/sync

P5 按能力提供：
GET            /v0/pro/accounts/:id/openai-reset-credits
POST           /v0/pro/accounts/:id/openai-reset
```

上述 `/v0/pro/*` 路由由 Manager 精确匹配、本地处理并执行与现有管理页面一致的管理员鉴权；未匹配的 `/v0/pro/*` 返回 `404`，不得转发到 Gateway。现有 `/v0/management/accounts` 及其他 Management API 的转发兼容行为保持不变，并通过兼容测试锁定。前端复用 CPA-Manager-Plus 现有 React Router、Zustand、高密度 Provider 表格和健康检查抽屉的交互模式，不移植 sub2apiplus 的 Vue 组件。

BFF 从 P1 起定义可测试的基础契约：列表使用游标分页并支持搜索、平台、认证类型、启用和健康状态筛选；写操作携带 `operation_id`、幂等键和当前资源版本，版本冲突返回机器可识别错误；错误响应统一包含 `code`、`message`、`retryable` 和可选 `details`。`sync` 默认只写 Manager 账号与绑定数据，不改 Gateway 配置；`dry_run=true` 返回新增、更新、待确认和冲突候选，确认同步保持幂等。P5 的批量操作和重绑在进入该阶段时补充独立请求体与逐项结果契约。

底层 Adapter 避免在业务服务中散布平台判断。P1 先定义账号读取能力，例如 `List`、`Get` 和 `ResolveBinding`；P2 增加 `GetUsage` 和用量快照能力；P3 再扩展 `Create`、`Update`、`Enable`、`Delete`、`ProbeCandidate`、`Probe`、`Test` 和 `SyncModels`。Adapter 同时返回 `capabilities`、`supported/unsupported/unknown` 状态和逐项错误，允许单个平台失败而不阻断其他账号。OAuth 认证文件、API 配置、Vertex 和插件 Provider 分别实现适配，不要求 P1 提前实现全部写操作。

凭证操作遵循以下边界：

- 已存在账号的主动配额查询、连通性测试、协议探测优先通过 CLIProxyAPI `/v0/management/api-call`，使用 `auth_index` 和 `$TOKEN$`，Manager 不读取 Gateway 宿主机文件或认证目录。
- 创建或编辑账号仍通过 CLIProxyAPI 现有配置和认证 Management API；`/api-call` 不能替代配置写入，也不能在没有 `auth_index` 时凭空选择新凭证。
- CLIProxyAPI 只增加两个必要的能力边界：OAuth/插件认证首次保存即为草稿/停用，以及 Auth/模型 Registry 级的 `allowed_models` 过滤。前者按本节要求包含 Hook 装配、OAuth Session 草稿意图和四种 Store 的首次持久化语义，不能理解为只有一个代码修改点。首期不新增 Gateway 账号数据库、不在每条 usage 事件中写入 `pro_account_id`、不增加新的模型请求代理路径。
- `/api-call` 不经过模型 Registry。Manager 使用它做测试或探测时必须先校验测试模型属于账号有效白名单，避免测试路径绕过账号模型规则。
- 新 API Key 只在请求内从 Manager 转交 Gateway，不写入 Manager 日志、SQLite、错误消息或前端持久化状态；新 OAuth/配置先作为停用草稿，探测和模型配置完成后再启用。

推荐交付阶段：

1. P0：建立主仓库基线提交、自有 fork、`origin/upstream` 关系、组件版本锁定和顶层 CI；记录 File、Object、PostgreSQL 和 Git Store 的兼容矩阵及测试环境。
2. P1：只读 Adapter、`pro_accounts`、`pro_account_bindings`、存量同步、只读 BFF 和统一账号列表。
3. P2：用量 Adapter、行内官方配额、Pro 账号富统计聚合、缓存和懒加载。
4. P3：写操作与探测 Adapter、持久化草稿状态机、OAuth 草稿首次持久化、四种 Store 兼容测试、账号级白名单传播和回读、协议探测和连通性测试基础能力。某 Store 尚未通过草稿兼容测试时，使用该 Store 的部署必须禁用 Pro OAuth 添加并返回明确能力错误。
5. P4：添加、编辑、启停、删除、OpenAI 底层协议迁移和失败补偿。
6. P5：批量操作、错误分类、批量漂移重绑和 OpenAI reset credits 查询/重置。
7. P6：旧入口隐藏、兼容验证、跨版本迁移、SQLite 备份恢复和上游合并测试。

#### 组件改造范围

- CPA-Manager-Plus：实现 `/v0/pro/accounts` 私有 BFF、统一账号页面、SQLite 账号表和绑定历史、只读及读写 Adapter、持久化草稿状态机和补偿动作、添加向导、底层账号归一化、OpenAI 探测与迁移编排、模型配置、Pro 账号富统计聚合、状态和用量展示，以及 P5 的 OpenAI reset credits 能力。现有 Provider 表格、健康检查和 React 状态管理作为 UI 基础复用。
- CLIProxyAPI：优先复用现有 Management API。首期复用已有 `auth_index`、认证快照和用量事件字段，不新增 `pro_account_id` 到每条用量事件；仅增加 OAuth/插件草稿首次持久化能力和 Auth/模型 Registry 级白名单能力。草稿持久化规则覆盖 File、Object、PostgreSQL 和 Git Store。只有现有事件无法提供账号标识时，才补充最小的 auth 快照字段。

#### 验收标准

1. 所有平台账号可在一个表格中查看，不需切换 AI Provider、认证文件和 OAuth 页面。
2. 每个平台只显示上表允许的添加方式，并在添加阶段配置允许模型。
3. OpenAI API 同时支持两个协议时优先 Responses，不重复创建账号。
4. Gemini API 可自定义 Base URL；OpenAI 格式的 Gemini 中转站从 OpenAI 添加。
5. Grok / xAI 和 Antigravity 不显示 API 添加入口。
6. 账号表格直接显示紧凑的用量区域；官方配额可用时显示窗口、进度和重置时间，否则显示本地统计或“暂无官方配额数据”。
7. 无官方配额时正确降级为本地统计，不显示虚假百分比或剩余额度。
8. 用量查询失败不影响账号列表和其他账号；桌面和移动端不出现重叠、截断或表格抖动。
9. 存量配置能在统一列表中识别，旧路由仍可访问。
10. 添加、编辑、启停、删除和测试能正确作用于底层配置或认证文件。
11. 页面、日志和错误信息不暴露 API Key、OAuth Token 或鉴权 Header。
12. 通过统一页面更换 Key、Base URL 或 OpenAI 底层协议后 `pro_account_id` 不变；绑定历史保留，历史用量仍归属同一账号。
13. 空白名单允许全部模型；非空白名单对模型列表、调度、测试和真实请求同时生效。
14. 自动刷新不会循环请求上游官方配额；只有用户主动查询时才使用 `active&force=true`。
15. `/v0/pro/accounts` 及其子路由只由 Manager 本地处理，未知 `/v0/pro/*` 返回 `404`；原 `/v0/management/accounts` 和其他 Gateway Management API 的转发行为不被静默改变。
16. OAuth 和插件 OAuth 新凭证首次保存时即为不可调度草稿；File、Object、PostgreSQL 和 Git Store 都能持久化带草稿标记的 Disabled 凭证，普通已删除 Disabled 凭证不会被重新创建。
17. 用户取消、授权超时、服务重启或后续步骤失败后，持久化状态机可以继续清理或补偿，不遗留短暂参与调度的凭证。
18. 认证目录、文件名、Key 或 Base URL 变化后可保留账号 UUID；无法唯一匹配时进入待确认，批量漂移可批量重绑，任何指纹都不会触发自动合并。
19. Pro 的 `passive`、`active` 和 `force` 在所有支持账号类型上遵循统一 BFF 契约；富统计按 Pro 账号全部历史绑定聚合，不复用 sub2apiplus 的商业 A/U 双成本语义。
20. 支持 reset credits 的 OpenAI OAuth 账号可以查询剩余次数并在二次确认后执行重置；不支持或状态未知时不显示虚假次数，失败不影响普通用量查询。
21. SQLite 升级迁移、回滚、备份和恢复流程通过发布验收，恢复后账号 UUID、绑定历史、草稿状态和历史用量归属保持正确。
22. 存量同步重复执行不会产生重复账号；精确命中、创建、待确认和冲突结果可区分，无法唯一识别时不自动合并。
23. 历史用量按事件时间与绑定有效期归属；绑定变化后的回填不会把路径重用或相同 `auth_index` 不同时期的事件归错账号。
24. Manager 保存的模型规则与 Gateway 回显的生效规则及版本一致；共享 Provider 下修改一个 Key 的白名单不会影响其他账号，写入或回读失败时旧规则保持可用。
25. 新 API Key 在没有 `auth_index` 时可以使用不持久化的候选凭证完成预探测；最终测试只使用已停用草稿的 `auth_index`，临时凭证不进入日志、SQLite 或调度。
26. 只支持本地统计的账号收到主动官方查询时返回统一结构和 `official_usage_unsupported`，不请求不存在的上游接口，也不显示虚假额度。

### 3.2 API Key 官方客户端兼容

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

### 3.3 协议模型列表配置

管理员可以分别自定义每个协议模型列表中显示的模型，包括增加、删除、同步和排序。下表仅为默认值，不按 Provider 硬编码：

| 协议 | 默认模型范围 |
|---|---|
| Anthropic Messages | Claude 模型 |
| OpenAI Responses | OpenAI / Codex Responses 模型 |
| OpenAI Chat Completions | OpenAI 兼容、Grok 等模型 |
| Gemini 原生协议 | Gemini、Antigravity 等模型 |

要求：

1. 模型列表和实际调用使用同一份规则。
2. 手工填写隐藏模型时必须拒绝调用。
3. 模型映射后再次校验最终模型。
4. 保留现有标准地址，不要求新增协议专用 Base URL。

部分客户端使用静态模型目录，服务端可能无法控制其界面显示，但仍须限制实际调用。

OpenAI Responses 和 Chat Completions 客户端都可能请求同一个 `/v1/models`。当请求没有携带可识别的客户端特征时，服务端无法可靠判断应返回哪一份列表；具体识别和回退规则在技术设计阶段确定。

### 3.4 账号连通性测试

所有账号都可以在统一账号页面执行测试：

```text
选择账号和模型
  -> 应用最终模型映射
  -> 根据账号设置选择普通请求或官方兼容 profile
  -> 发起最小真实请求
  -> 显示并保存结果
```

测试结果至少区分：

- 成功。
- 认证失败。
- 模型不可用或无权限。
- 协议不兼容。
- 配额耗尽或限流。
- 网络、代理或 TLS 错误。
- 上游服务异常。

测试流量应进入 Token 和成本统计，并单独标记。HTTP 200 但正文为降级或临时不可用提示时不能判定为成功。

## 4. 组件分工

| 功能 | CPA-Manager-Plus | CLIProxyAPI |
|---|---|---|
| 统一账号页面和添加向导 | 主要实现 | 提供 Management API |
| OpenAI 协议探测 | 负责探测流程和结果判断 | 优先复用现有 Management API，不默认新增专用接口 |
| 模型白名单和映射 | 配置页面 | 运行时执行 |
| 连通性测试 | 页面和结果展示 | 执行真实请求 |
| 用量、成本和健康状态 | 主要实现 | 产生运行数据 |
| 官方客户端兼容 | 配置和状态展示 | Executor、Transport 和路由执行 |
| 协议模型范围 | 配置页面 | 过滤列表并限制调用 |

CPA-Manager-Plus 不代理普通模型请求，只通过 CLIProxyAPI Management API 管理配置和认证文件，不绕过 API 直接读写 Gateway 宿主机文件、认证目录或运行数据库。

## 5. 第一版验收标准

1. 一个页面可以查看和管理全部平台账号。
2. 添加账号必须先选平台，再选 OAuth 或 API Key。
3. OpenAI API Key 可以自动识别 Responses 和 Chat Completions 能力。
4. OAuth 和 API Key 账号都可以配置允许模型和模型映射。
5. 每个账号都可以测试连通性，并显示明确失败原因。
6. 开启官方客户端兼容后，Claude 和 Codex 请求通过抓包对比测试。
7. 管理员可以分别配置四种协议显示的模型；可识别的模型目录请求按对应配置返回，隐藏模型无法直接调用。
8. 每个账号和 Key 都能查看 Token、请求次数和官方成本估算。
9. Manager 故障不影响 CLIProxyAPI 继续处理模型请求。
10. 两个组件可以独立升级和回滚。

## 6. 建议开发顺序

1. 建立 Pro 主项目、两个 fork 和统一 Compose。
2. 定义 Management API、用量事件和协议模型策略契约。
3. 实现统一账号页面、添加向导、用量窗口和连通性测试。
4. 实现 OpenAI API Key 自动探测和账号模型配置。
5. 在 CLIProxyAPI 增加 Claude/Codex 官方客户端兼容 profile。
6. 在 CLIProxyAPI 实现协议模型策略、模型目录识别和回退规则。
7. 完成集成与升级测试后发布第一版。

## 7. 默认目录与部署

开发工作区：

```text
cliproxyapi-pro/
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

`sub2apiplus` 只作为开发参考，不放入 CLIProxyAPI Pro 项目。

产品名称使用 `CLIProxyAPI Pro`；开发目录、仓库名、安装目录、Compose 项目名和 Docker 镜像名称统一使用小写 `cliproxyapi-pro`。
