# API Key 官方客户端兼容实施方案

- 状态：待实施
- 对应需求：[readme.md 3.4 API Key 官方客户端兼容](../readme.md#34-api-key-官方客户端兼容)

## 1. 目标与首版范围

本功能通过账号级开关，让第三方客户端使用 Anthropic/OpenAI API Key 账号时，上游请求尽量接近 Claude Desktop 或 Codex Desktop 直接使用该 API Key 的请求形态。

首版仅支持：

- Anthropic API Key 对应 Claude Desktop Profile。
- OpenAI Responses/Codex API Key 对应 Codex Desktop Profile。
- 开关开启且请求来自非官方客户端时应用 Profile。
- 账号连通性测试强制复用正式 Profile。
- 官方 Claude/Codex Desktop 或 CLI 跳过 Profile，执行现有普通 API Key 链路。

首版不支持：

- Codex CLI 可选 Profile。
- 兼容请求的上游 WebSocket Profile。
- 非空 TLS 指纹 Profile。
- OAuth、xAI 和 `openai-compatibility` Chat Completions 账号。
- Manager 数据库投影、列表徽标和旧 Provider 页面配置入口。
- 隐藏 Prompt、产品 Memory、账号状态或 TLS/HTTP2 帧级完全复制。

## 2. 最小侵入原则

- 配置缺失或关闭时，现有请求行为保持不变。
- Gateway 配置和 Profile 注册表是唯一事实源，Manager 不参与普通模型请求。
- 复用现有模型映射、Translator、连通性测试和 Auth 热更新链路。
- 不修改调度器、模型映射、`internal/translator` 和全局 Transport。
- 共享 Profile 契约与注册表放在不依赖配置层和 Executor 的中立包；请求构造支持逻辑放在 Executor `helps`，Executor 本身只增加少量挂接点。
- 自定义 Header 的全局优先级不变，只有命中 Profile 的请求保护身份字段。
- `.references/sub2apiplus` 保持只读，不成为运行或构建依赖。

## 3. 第 0 步：冻结 Profile 契约

编码前取得 Claude Desktop、Codex Desktop 直连 API Key 的真实脱敏抓包。

### 3.1 抓包范围

- Claude `/v1/messages`：非流式、流式、带工具、无工具。
- Claude `/v1/messages/count_tokens`。
- Codex `/v1/responses`：现有普通执行与 SSE。
- Codex `/v1/responses/compact`。
- 官方 Codex WebSocket 路由，仅用于确认跳过兼容时的当前行为。

### 3.2 冻结内容

- URL、方法、HTTP 版本和路由选择。
- Header 名称、值、必要顺序及动态字段。
- Body 默认值、System、Metadata、缓存和工具结构。
- Installation、Session、Thread、Window、Turn 等字段的稳定性和生命周期。
- 固定 Beta 与动态 Beta 的组合和顺序。
- TLS ClientHello、ALPN 及与当前 Transport 的差异。

### 3.3 Profile 规则

- 使用精确且可版本化的 ID，例如 `claude-desktop-<客户端版本>-v1`、`codex-desktop-<客户端版本>-v1`。
- 已发布 Profile 不原地修改；契约变化时新增 Profile ID。
- 未知、废弃或 Provider 不匹配的 Profile 直接拒绝，不静默回退。
- 原始抓包不进入仓库；仓库只保存脱敏契约和 Golden Fixture。
- `sub2apiplus` 只作规则参考和交叉验证，不能替代本项目真实抓包。

## 4. 配置契约

ClaudeKey、CodexKey 复用同一可选嵌套结构：

```yaml
official-client-compatibility:
  enabled: true
  profile: claude-desktop-<精确版本>-v1
  tls-profile: ""
```

字段规则：

- 配置块缺失：关闭功能。
- `enabled=false`：关闭功能，但允许保留已选择的 Profile。
- `enabled=true`：必须最终持久化精确 Profile ID。
- Management 开启但未提交 Profile 时，由 Gateway 注册表选择当前精确版本并在持久化前写回。
- 直接编辑配置文件时，启用状态必须显式填写精确 Profile。
- 首版只接受空 `tls-profile`，任何非空值直接拒绝。
- xAI 中出现整个配置块即拒绝，禁止保存后静默无效。

Management PATCH 使用整个兼容结构的指针：

| PATCH 状态 | 语义 |
|---|---|
| 字段缺失 | 不修改兼容配置 |
| `{ "enabled": false, ... }` | 明确关闭 |
| `{ "enabled": true, ... }` | 开启并校验 Profile |

## 5. Profile 注册表

在 `components/cliproxyapi/internal/officialclient` 新建不依赖 `internal/config`、Management 或 Executor 的中立包，集中实现：

- Profile ID、Provider、客户端版本和有效状态。
- 端点类型和可声明的 Header/Body 契约。
- 官方客户端识别规则。
- 受保护 Header 集合。
- HTTP/SSE/WS 和 TLS 策略。
- Profile 校验、当前版本解析和脱敏诊断。

配置结构可以引用该中立包定义的兼容配置类型，或将基础字段传给其校验函数；中立包不得反向导入 `internal/config`。这样配置加载、Management、Synthesizer 和 Executor 可以共享同一注册表，避免 `internal/config` 导入 `executor/helps` 形成依赖倒置或循环。

端点专属的请求改写、JSON/SSE 工具回写和 Header 最终处理继续放在 `internal/runtime/executor/helps`，并只依赖中立注册表。Gateway 注册表是合法 Profile 及当前版本的唯一事实源，Manager 不复制默认 Profile 常量。

身份字段规则：

- Installation、Session 等稳定值基于 `auth.ID` 和 Profile 命名空间生成。
- Turn、请求 ID 等请求级值按请求生成。
- 禁止使用原始 API Key 生成缓存键、Metadata 或诊断字段。

## 6. Auth Attributes 与热更新

真实热更新链路：

```text
配置重载
→ ConfigSynthesizer 重新生成 Auth
→ Auth.Attributes 发生变化
→ watcher 派发 Modify
→ AuthManager 使用新 Auth
→ 后续请求读取新 Profile
```

实现要求：

- 不复用只处理 `header:<name>` 的 `addConfigHeadersToAttrs`。
- 新增 `addOfficialClientCompatibilityToAttrs` 和 `officialClientCompatibilityFromAttrs` 等专用方法。
- 使用单个稳定键 `official_client_compatibility`。
- 属性值保存规范化后的完整 JSON，原子传递 `enabled/profile/tls-profile`。
- 显式保留 `enabled=false`，避免部分字段状态不一致。
- `config_diff.go` 只增加脱敏差异描述和测试，不参与热更新生效。

运行时读取规则：

- Attributes 中存在兼容键时，该值绝对权威，包括 `enabled=false`。
- JSON 损坏、Profile 未知或 Provider 不匹配时终止当前请求并返回脱敏配置错误；不得执行兼容链路，也不得回退原始配置或普通 API Key 链路。
- Attributes 中不存在兼容键时视为关闭。
- 不按 API Key/BaseURL 回退扫描配置项，避免重复配置命中歧义。

### 6.1 Profile 退役与升级预检

- 已发布 Profile 不得在同一个版本中直接移除或标记为废弃；必须先经历至少一个兼容迁移版本，或经过事先明确公布的迁移窗口。
- 迁移期内旧 Profile ID 仍保持可执行，但不再作为当前默认版本；Gateway 在启动和配置重载日志中输出脱敏迁移告警，至少包含账号定位信息、Provider、旧 Profile ID、推荐的新 Profile ID 和计划退役版本或日期，不得包含凭据。
- Gateway 启动和配置重载必须预检全部已启用账号引用的 Profile，不得等到真实请求到达后才发现未知、Provider 不匹配或已废弃 Profile。
- 配置热重载发现非法 Profile 时拒绝本次重载并保留上一份有效运行态；冷启动发现非法 Profile 时拒绝加载配置并返回明确的脱敏配置错误。
- 迁移窗口结束后，已废弃 Profile 继续按照 fail-closed 规则拒绝；不得静默回退、自动替换 Profile ID 或执行普通 API Key 链路。
- 运行时 fail-closed 只作为手工 Auth、损坏 Attributes、插件伪造状态或其他绕过配置预检场景的最后防线，不能替代启动和重载预检。

## 7. xAI 三层隔离

由于 `XAIKey = CodexKey`，首版保留类型别名以减少改动，并增加三层隔离：

1. 配置和 Management：
   - 配置加载时拒绝 xAI 兼容块。
   - xAI PUT 显式校验并拒绝。
   - xAI PATCH 增加检测字段，防止未知 JSON 字段被静默忽略。
2. Synthesizer：
   - `synthesizeCodexStyleKeys` 仅在 `provider == "codex"` 时写兼容 Attributes。
   - xAI 永不写入兼容 Attributes。
3. 运行时：
   - 决策器第一步校验 `auth.Provider == "codex"` 和 API Key 类型。
   - 手工 Auth、测试 Auth 或插件伪造 Attributes 也不能触发 Codex Profile。

## 8. 请求级决策与执行顺序

统一命中条件：

```text
账号为 API Key
且 Provider 为 claude 或 codex
且兼容开关开启
且（账号连通性测试或未识别为官方客户端）
```

三态行为：

| 状态 | 行为 |
|---|---|
| 配置缺失或关闭 | 完整执行现有链路 |
| 开启且识别为官方客户端 | 跳过兼容 Profile，执行现有普通 API Key 链路 |
| 开启且为非官方客户端或连通性测试 | 执行端点专属 Profile |

官方客户端采用经抓包验证的 UA、`originator` 等信号正向识别，比较不区分大小写；未明确识别即视为非官方。该识别只决定是否改写请求，不是鉴权边界。

账号连通性测试直接读取现有 `Options.Metadata` 中的 `coreexecutor.ConnectivityTestMetadataKey`，不得另造下游 Header、查询参数或新的 Context 标记。

固定执行顺序：

```text
现有账号选择和模型映射
→ 提前计算 Profile 决策
→ Translator / Thinking / Payload 配置
→ 端点专属 Body Profile
→ 现有基础 Header
→ 账号自定义 Header
→ Profile 最终重写受保护 Header
→ Transport/路由决策
→ 上游
→ 请求级响应逆向映射
```

## 9. 受保护 Header

保持现有 `ApplyCustomHeadersFromAttrs` 不变，不全局改变 Header 优先级。

兼容路径使用专用 Profile Header 最终器：

1. 构造现有基础 Header。
2. 应用账号自定义 Header。
3. 删除所有大小写变体的 Profile 受保护 Header。
4. 按 Profile 规定的名称和值重新写入。

保护范围包括上游鉴权、User-Agent、`originator`、Anthropic Version/Beta/Stainless/Session、Codex Session/Thread/Window/Turn 和其他由抓包确认的 Profile 身份字段。非兼容账号继续保持现有 Header 行为。

## 10. Anthropic 接入

修改现有 Claude Execute、ExecuteStream 和 CountTokens；Profile 支持代码放在 `helps`，Executor 只保留挂接逻辑。

### 10.1 Messages 与 SSE

1. 提前计算决策；命中 Profile 时跳过旧 `applyCloaking`。
2. 执行现有翻译、Thinking 和 Payload 配置。
3. 在 Payload 配置和通用归一化之后应用 Claude Desktop Body Profile。
4. 按 Profile 生成固定和动态 Beta。
5. 应用请求级工具名称映射。
6. 构造基础 Header、自定义 Header 和受保护 Header。

Profile 负责抓包确认的身份 Header、Beta 顺序、System、Metadata、Session 和缓存控制。只允许加入抓包确认的公开身份内容，保留用户实际 System 语义，不复制隐藏 Prompt、Memory 或账号状态。

### 10.2 Count Tokens

Count Tokens 使用独立构造规则：

- 只加入该端点需要的 Beta。
- 不盲目加入 Messages 的 System、Session、压缩或缓存模板。
- 工具结构使用 Count Tokens 专属规则。
- 最终字段和 Header 以真实抓包为准。

### 10.3 工具归一

- 只处理当前请求实际携带的已知工具。
- 保留工具 Schema、描述和业务含义。
- 每个请求维护独立 reverse map。
- JSON 和 SSE 响应只改写结构化工具名称字段。
- 不使用全局 reverse map。
- 不向普通请求注入参考实现的完整工具基线。
- 完整工具基线如有需要，仅用于账号测试或 Keeper 专项场景。

## 11. Codex 接入

修改 Codex Execute、ExecuteStream、Compact 和 `CodexAutoExecutor` 路由决策。

### 11.1 普通 Responses 与 SSE

- 保留当前上游流式实现，不重构现有响应聚合策略。
- Profile Body 在现有 Payload 配置后应用。
- Profile 命中时，现有 `cacheHelper`、`identity-confuse` 不再写入冲突字段。
- Profile 独立拥有缓存键、Session、Client Metadata 和 Turn Metadata。
- Body 默认值必须来自真实抓包，不注入隐藏 Codex Instructions。

### 11.2 Compact

- 保持现有独立 unary JSON 分支。
- 最终阶段删除抓包未包含的普通 Responses 字段，重点检查 `stream`、`store`、`prompt_cache_key`、`client_metadata`。
- 使用 Compact 专属 Header 和 Accept。
- 不复用普通 Responses Body 模板。

### 11.3 WebSocket

- 非官方客户端命中 Codex Desktop Profile 时，上游固定使用 HTTP/SSE。
- 下游 WebSocket 继续使用现有协议桥接。
- 官方 Codex 客户端跳过 Profile，保留当前上游 WebSocket 选择。
- 不修改 scheduler；真实上游 WS Profile 待后续抓包验证后单独实现。

### 11.4 Chat Completions 边界

- 仅 `codex-api-key`/Responses 类型账号可以启用。
- 下游 Chat Completions 若通过现有路径路由到 Codex 账号，则在现有翻译后应用 Profile。
- 不新增强制 Chat Completions 到 Responses 的转换。
- `openai-compatibility` Chat Completions 账号不支持本功能。

## 12. Management API

- Claude/Codex GET、PUT 通过嵌入配置结构自然支持新字段。
- PATCH 使用整个兼容结构的指针，区分“未提交”和“明确关闭”。
- PUT/PATCH 在修改内存配置前完成 Profile、TLS 和 Provider 校验。
- Management 开启但未传 Profile 时，由 Gateway 注册表写入当前精确 ID。
- xAI PUT/PATCH 明确拒绝兼容块。
- 不修改 `config_auth_index.go`，不新增专用状态接口。
- 在现有 Management 响应能力 Header 中增加 `X-CPA-SUPPORT-OFFICIAL-CLIENT-COMPATIBILITY: true`；Manager 必须先探测该能力，不能通过试写配置判断支持状态。
- 错误、日志和诊断不得返回 API Key、Token、Cookie 或完整鉴权 Header。

## 13. Manager

### 13.1 后端

扩展现有统一账号 Create/Update 输入、Gateway 实时详情和生命周期：

- 创建 API Key 账号时写入兼容配置。
- 编辑兼容配置时执行 Gateway 热更新，不触发凭据迁移。
- 凭据迁移时继承原兼容配置，除非用户同时明确修改。
- 详情从 Gateway 实时读取，不写入 `pro_accounts` 或 SQLite。
- Gateway 未声明官方客户端兼容能力时，Manager 隐藏或禁用开关，并返回明确的“不支持”错误，禁止向旧 Gateway 发送新字段。

写入补偿流程：

```text
读取旧兼容块
→ 定点 PATCH 新值
→ GET 回读
→ 与期望值比较
→ 不一致或结果不确定时恢复旧值
→ 再次 GET 验证
```

混合更新的后续步骤失败时，也必须恢复原兼容块。

### 13.2 前端

统一账号只修改新增向导和编辑弹窗：

- 仅 Anthropic API Key、OpenAI Responses/Codex API Key 显示开关。
- OAuth、xAI 和 Chat Completions 隐藏或显示不支持。
- 首版不提供 Profile 下拉选择，由 Gateway 写入当前精确 Profile。
- 编辑弹窗同时展示实际 Profile ID 和 TLS 策略。
- 单独计算 `compatibilityChanged`，不得并入 `credentialsChanged`，切换开关不要求重新输入 API Key。

第一版不修改账号列表、数据库和旧 Provider 页面，但必须增加旧 Provider 整表 PUT 回归测试，确保保存其他字段时不会丢失未知兼容配置块。

## 14. TLS 策略

- Claude、Codex 首版继续使用当前 `NewUtlsHTTPClient` 选择逻辑。
- 抓包用于验证当前 Transport 与官方请求的差异。
- 不实现新 TLS Transport，非空 `tls-profile` 全部拒绝。
- UI 只展示当前默认 Transport，不显示不可用的 TLS 选择器。

后续增加 TLS Profile 时必须作为独立变更集，并满足：

- 使用独立版本化 ID。
- 通过真实 ClientHello 和 ALPN 验证。
- Transport 缓存键至少包含 Profile、代理和目标主机。
- 不全局替换其他账号 Transport。

## 15. 文件级改动范围

### 15.1 CLIProxyAPI

- `internal/config/config.go`、`internal/config/parse.go`：配置结构和 Provider-aware 校验。
- `components/cliproxyapi/config.example.yaml`：补充默认关闭的嵌套配置示例和字段说明。
- `internal/officialclient/`：共享配置类型、Profile 注册表、Provider/端点策略和无环校验逻辑。
- `internal/watcher/synthesizer/config.go`、`helpers.go`：兼容结构写入 Auth Attributes。
- `internal/watcher/diff/config_diff.go`：脱敏差异诊断。
- `internal/api/handlers/management/config_lists.go`：Claude/Codex PATCH 和 xAI 拒绝逻辑。
- `internal/api/handlers/management/handler.go`、`internal/api/server.go`：能力 Header 和跨域暴露配置。
- `internal/runtime/executor/helps/`：请求级决策适配、Header/Body 构造和 JSON/SSE 响应支持逻辑。
- `internal/runtime/executor/claude_executor.go`：Messages、SSE、Count Tokens 挂接。
- `internal/runtime/executor/codex_executor.go`：Responses、SSE、Compact 挂接。
- `internal/runtime/executor/codex_websockets_executor.go`：Codex HTTP/SSE/WS 路由决策。
- 对应配置、Management、Synthesizer、Executor 和路由测试。

### 15.2 CPA-Manager-Plus

- `apps/manager-server/internal/service/proaccountgateway/`：能力探测、兼容配置 DTO、实时读取、PATCH、回读和恢复。
- `apps/manager-server/internal/service/proaccountlifecycle/`：创建、更新、凭据迁移继承和补偿。
- `apps/manager-server/internal/http/controller/proaccount/`：Create/Update 请求字段。
- `apps/web/src/services/api/proAccounts.ts`：前端 DTO 与序列化。
- `apps/web/src/features/accounts/AccountWizardModal.tsx`：新增向导开关。
- `apps/web/src/features/accounts/AccountEditModal.tsx`：编辑、详情展示和独立变更判断。
- 对应后端、前端和旧 Provider 未知字段保留测试。

## 16. 实施变更集

严格一次处理一个变更集，完成并对照本方案复核后再进入下一项。

### 变更集 0：抓包与契约

- 取得真实脱敏抓包。
- 输出 Claude/Codex Profile 契约。
- 冻结精确 Profile ID。
- 建立脱敏 Golden Fixture。
- 不修改运行代码。

### 变更集 1：配置骨架

- 嵌套配置结构和统一校验。
- Profile 注册表。
- JSON Attributes 合成和读取。
- xAI 三层隔离。
- 三态决策、官方客户端识别和受保护 Header 最终器。
- Management GET/PUT/PATCH 和热更新测试。
- 热更新必须覆盖：只修改兼容开关或 Profile、保持 API Key/BaseURL 不变时，Auth ID 保持稳定，watcher 必须派发 `Modify`，AuthManager 必须替换为包含新 JSON Attributes 的 Auth。
- 上述测试必须在不重启 Gateway 的条件下验证，且后续一次 Profile 决策读取到新值；本变更集暂不以上游请求形态变化作为判定条件。
- 暂不改变实际请求形态。

### 变更集 2：Anthropic

- Messages 非流式与 SSE。
- Count Tokens 独立规则。
- Beta、System、Metadata、Session 和缓存。
- 实际工具映射及 JSON/SSE 响应回写。
- 正式请求与连通性测试抓包对比。
- 仅修改兼容开关、不重启 Gateway，验证下一次 Claude 上游请求立即切换为对应的 Profile 或普通链路形态。

### 变更集 3：Codex

- 普通 Responses 与 SSE。
- Compact 独立规则。
- 缓存和 Client Metadata。
- 绕过冲突的 `cacheHelper`、`identity-confuse`。
- HTTP/SSE 和官方 WebSocket 路由回归。
- 仅修改兼容开关、不重启 Gateway，验证下一次 Codex 上游请求立即切换为对应的 Profile 或普通链路形态。

### 变更集 4：Manager

- 后端 DTO、Gateway 生命周期和失败补偿。
- 新增向导和编辑/详情弹窗。
- `compatibilityChanged` 独立保存。
- 旧 Provider 未知字段保留测试。

### 变更集 5：集成验收

- 正式请求与账号连通性测试。
- 官方/非官方客户端回归。
- 抓包契约对比。
- 全量测试、构建、重启和日志检查。

## 17. 测试与验收矩阵

必须覆盖：

- 配置缺失、关闭、开启、未知 Profile 和损坏 Attributes。
- Profile 迁移期旧 ID 可执行并产生启动/重载告警，以及迁移期结束后已废弃 ID 在启动、热重载和运行时的 fail-closed 行为。
- 仅修改兼容开关或 Profile 且 Auth ID 不变时，watcher 派发 `Modify`、AuthManager 读取新 Attributes，并在 Anthropic/Codex 变更集完成后验证不重启时下一请求形态立即改变。
- API Key、OAuth、xAI。
- 官方客户端、非官方客户端、账号连通性测试。
- 模型映射后的最终上游模型。
- Claude Messages 流式/非流式、Count Tokens。
- Claude 工具请求、JSON 响应和 SSE 响应回写。
- Codex 普通 Responses、SSE、Compact、下游 WebSocket。
- 自定义 Header 不同大小写覆盖尝试。
- 关闭开关和官方客户端跳过时的现有行为回归。
- xAI 配置、Management、Synthesizer、伪造 Attributes 四类隔离。
- Manager 创建、编辑、凭据迁移、写后回读和失败恢复。
- 页面、日志、错误和测试夹具的敏感字段扫描。
- Manager 停止后 Gateway 继续正常转发。
- `.references/sub2apiplus` 保持只读且不进入构建依赖。

实现完成后执行：

- 格式化全部改动的 Go 文件。
- Gateway 相关单测、全量测试和构建。
- Manager 后端单测。
- 前端单测、类型检查和生产构建。
- 按现有部署方式重建、重启受影响组件并检查日志。
- 以上游实际收到的请求为最终判定依据。

## 18. 回滚策略

- 账号级关闭开关即可回到现有路径。
- 未知 Profile 不允许静默回退。
- Profile 更新使用新版本 ID，不覆盖旧版本。
- Manager 写入失败自动恢复旧兼容块。
- Gateway 配置始终是唯一事实源，不依赖 Manager 数据库。
- 新 Gateway 与旧 Manager 可以维持普通模型请求；已有兼容配置在未被旧 Manager 重建或整表保存前继续生效。
- 混合版本期间，旧 Manager 不得修改已启用兼容配置的账号；如无法避免，必须先备份 Gateway 配置并在操作后核对兼容块。
- 新 Manager 只有在 Gateway 声明能力后才展示和写入本功能，因此可以与旧 Gateway 共存但不能配置该功能。
- 推荐升级顺序为 Gateway 后 Manager；推荐回滚顺序为先 Manager，再关闭已启用的兼容配置，导出配置备份后回滚 Gateway。
- 回滚到不识别新字段的旧 Gateway 前必须保留完整配置备份，防止旧版本后续整表保存时丢弃未知配置块。
- “独立升级和回滚”保证普通模型请求和既有账号管理继续工作，不保证旧组件能够配置或解释新功能。
- 实施时保留工作树已有修改，不覆盖无关文件。
