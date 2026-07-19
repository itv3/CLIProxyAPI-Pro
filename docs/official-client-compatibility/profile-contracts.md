# API Key 官方客户端兼容 Profile 契约

- 状态：已冻结，供变更集 1～5 使用
- 冻结日期：2026-07-19
- Claude Profile：`claude-desktop-2.1.215-v1`
- Codex Profile：`codex-desktop-0.145.0-alpha.18-v1`
- 参考实现：`sub2apiplus v0.1.156-1`（提交 `ca26467a39697fa1fa6118b47e4d7cd80c041cbc`）

## 1. 契约用途与证据优先级

本文冻结首版 Profile 的路由、Header、Body、身份生命周期和实现边界。机器可读 Golden Fixture 位于 [`fixtures`](fixtures/) 目录，后续实现和回归测试必须读取同一组精确 Profile ID，不得另建未版本化常量。

发生冲突时按以下顺序处理：

1. `readme.md` 和实施方案中的安全、范围及兼容边界。
2. 当前版本官方客户端的真实脱敏抓包。
3. `.references/sub2apiplus` 中的完整算法和测试。
4. 无法验证的历史样本或经验推断。

每条规则使用三种证据级别：

| 级别 | 含义 |
|---|---|
| `capture_verified` | 当前官方客户端抓包已确认。 |
| `reference_derived` | 当前客户端未发出该形态，规则来自 sub2apiplus 完整实现。 |
| `implementation_policy` | 为保护调用语义、安全边界或现有架构而明确确定的首版行为。 |

## 2. 抓包证据

原始 PCAP 含凭据，只保存在仓库外且权限为 `0600`。仓库仅记录 SHA-256、统计结论和脱敏结构，不记录本机路径、请求正文、API Key、Token、Cookie、完整 Metadata 或会话标识。

| 客户端 | SHA-256 | 严格解析结果 | 用途 |
|---|---|---|---|
| Claude Desktop 2.1.215 工具闭环 | `b73d31cad1063d2fba1dead4f6647f5bf42d8449a10ee98286b84673c5824eb3` | 37 个 Messages、18 个 Count Tokens；Messages 中 19 个 `stream=true`、18 个省略 `stream` | 主契约、工具调用和 Count Tokens |
| Claude Desktop 2.1.215 基线 | `6dee74cf4e53540b34ff7d893f1e0532cf57ff8408440930ccea34bc7931d510` | 41 个 Messages、17 个 Count Tokens | 交叉验证 |
| Codex Desktop 0.145.0-alpha.18 基线 | `caeadd94ffc4d838d32d1bb0a848120e9d04073b9bec5e1f7db084601e746e0e` | 6 个 HTTP/SSE Responses、11 个 Responses WebSocket Upgrade、1 个 Models | Responses Body、身份关系和官方 WS 回归 |
| Codex Desktop 关闭远程后第一次 Compact | `d8d48f69d54724219b9d8ebd757638580bd531c68405fdef420cee1ed18ddc49` | 1 个 Models、1 个 Responses WebSocket Upgrade；无 Compact 路由 | 排除 WebSocket 中的独立 Compact 请求 |
| Codex Desktop 关闭 Provider WS 后 Compact | `7d915c55e8c87ac7432be90a8556d37384ac057e892b3702b5e7b8a649c3fc02` | 4 个 `POST /v1/responses`；其中 1 个 Turn Metadata 标记手工 Compact；无 `/v1/responses/compact` | 当前手工 Compact 真正行为 |

抓包位于 TLS 终止后的 Nginx → sub2apiplus HTTP/1.1 链路，因此：

- `cf-*`、`cdn-loop`、`x-forwarded-*`、`x-real-ip`、`Host` 和 `Content-Length` 不属于客户端 Profile。
- HTTP 版本、Header 顺序和大小写只作观察，不声明为客户端直连不变量。
- 无法从该层取得 ClientHello、ALPN 和客户端直连 HTTP/2 帧；首版 `tls-profile` 必须为空。
- `Accept-Encoding` 和请求压缩记录为观察值，首版继续由现有 Transport 管理，不为模仿压缩而改造 Transport。

## 3. Claude Desktop 2.1.215 契约

### 3.1 路由与响应模式

| 场景 | 方法与路径 | 响应模式 |
|---|---|---|
| Messages 流式 | `POST /v1/messages?beta=true` | SSE |
| Messages 非流式 | `POST /v1/messages?beta=true` | JSON |
| Count Tokens | `POST /v1/messages/count_tokens?beta=true` | JSON |

Messages 和 Count Tokens 必须使用独立构造规则。Count Tokens 不得继承 Messages 的 System、Metadata、流式字段和结构化输出规则。

### 3.2 身份 Header

首版固定身份：

- `User-Agent: claude-cli/2.1.215 (external, claude-desktop-3p, agent-sdk/0.3.215)`
- `Accept: application/json`，官方客户端即使 `stream=true` 也使用该值。
- `Content-Type: application/json`
- `Anthropic-Version: 2023-06-01`
- `X-App: cli`
- `X-Stainless-Lang: js`
- `X-Stainless-Package-Version: 0.94.0`
- `X-Stainless-OS: MacOS`
- `X-Stainless-Arch: arm64`
- `X-Stainless-Runtime: node`
- `X-Stainless-Runtime-Version: v26.3.0`
- `X-Stainless-Timeout: 900` 仅用于 Messages；当前 Count Tokens 样本未携带。
- `Anthropic-Dangerous-Direct-Browser-Access: true`

Messages 同时发送 `x-api-key` 和 `Authorization: Bearer <API Key>`；Count Tokens 保持只发送 `x-api-key`。当前官方抓包只观察到 `x-api-key`，Messages 双鉴权来自 `sub2apiplus` 的完整兼容实现，并已通过 AnyRouter 真实账号验证；两项鉴权字段都属于受保护 Header，账号自定义 Header 不得覆盖。`x-client-request-id` 和 `x-stainless-helper-method` 在当前三个端点样本中均不存在，兼容路径应删除客户端伪造值。

`X-Claude-Code-Session-Id` 是会话级 UUID：同一逻辑会话稳定，Messages 中必须与 `metadata.user_id` 内的 session 片段一致；Count Tokens 不添加 Metadata，但复用同一会话标识。`X-Stainless-Retry-Count` 初始为 `0`，真实重试允许递增。

### 3.3 Beta 规则

按当前抓包与 `sub2apiplus` 1M 兼容契约共同冻结以下顺序：

| 场景 | `anthropic-beta` |
|---|---|
| Messages 非流式 | `claude-code-20250219,context-1m-2025-08-07,interleaved-thinking-2025-05-14,mid-conversation-system-2026-04-07,effort-2025-11-24,fallback-credit-2026-06-01` |
| Messages 流式基线 | `claude-code-20250219,context-1m-2025-08-07,interleaved-thinking-2025-05-14,mid-conversation-system-2026-04-07,effort-2025-11-24,fallback-credit-2026-06-01` |
| Messages 流式或非流式且 `output_config.format.type=json_schema` | 将基础列表末尾的 `fallback-credit-2026-06-01` 替换为 `structured-outputs-2025-12-15` |
| Count Tokens | `claude-code-20250219,context-1m-2025-08-07,interleaved-thinking-2025-05-14,mid-conversation-system-2026-04-07,effort-2025-11-24,fallback-credit-2026-06-01,token-counting-2024-11-01` |

当前 Profile 以 `sub2apiplus` 2026-07-13 官方客户端兼容契约为基线，固定保留 `context-1m-2025-08-07`，并按结构化输出规则替换末尾 fallback beta。Count Tokens 在同一基础列表后追加 `token-counting-2024-11-01`。

### 3.4 Body、缓存与工具

- 保留最终映射后的 `model`、用户 `messages`、`max_tokens`、Thinking 和结构化输出语义。
- 流式与非流式由 Executor 分支决定，不把 Count Tokens 伪装成 Messages。
- Messages 生成规范 `metadata.user_id`；不得使用原始 API Key 作为生成种子或诊断字段。
- 保留用户 System 语义；只允许增加公开的 billing attribution 和 Agent SDK 身份块，不复制不透明产品 Prompt、Memory、工作区内容或账号状态。
- 官方流式样本存在 3 个 System text block，但 Golden Fixture 不保存其正文；第三个不透明产品块不属于首版可复制内容。
- 不自动补 `temperature=1`。仅删除没有附加语义的 `tool_choice: {"type":"auto"}`；存在 `disable_parallel_tool_use` 等字段时原样保留。
- 缓存断点按当前请求已有内容规范化，不注入完整工具基线。当前抓包中的缓存断点随子链路变化，不能冻结成所有请求强制模板。
- 只归一当前请求实际携带的已知工具；未知工具和 MCP 工具保持名称及 Schema。
- 工具映射必须是请求级对象。响应只改写 JSON/SSE 中 `tool_use.name`、`tool_reference.tool_name` 和等价的结构化工具字段，禁止替换普通文本。

## 4. Codex Desktop 0.145.0-alpha.18 契约

### 4.1 路由与首版边界

非官方客户端命中 Profile 后，上游固定使用 HTTP/SSE：

- `POST /v1/responses`
- `Accept: text/event-stream`
- Body 强制 `stream=true`、`store=false`

官方 Codex 客户端跳过 Profile，并保留现有 `GET /v1/responses` WebSocket Upgrade。抓包确认 WS 使用 `OpenAI-Beta: responses_websockets=2026-02-06` 和 `permessage-deflate`；这些字段不得加入兼容 Profile 的 HTTP/SSE 请求。

### 4.2 身份 Header 与生命周期

固定身份：

- `User-Agent: Codex Desktop/0.145.0-alpha.18 (Mac OS 26.5.2; arm64) unknown (Codex Desktop; 26.715.31925)`
- `originator: Codex Desktop`
- `x-codex-beta-features: remote_compaction_v2`
- `Content-Type: application/json`
- HTTP/SSE 使用 `Accept: text/event-stream`；Compact 使用 `Accept: application/json`。
- API Key 使用 `Authorization: Bearer <API Key>`；当前抓包没有 `x-api-key`。
- HTTP/SSE 不发送 `OpenAI-Beta` 和 `version`。

身份字段关系：

- Installation ID：安装级稳定 UUID。
- Session ID：逻辑任务级稳定 UUID。
- Thread ID：首版与 Session ID 相同。
- Window ID：`<session-id>:<非负整数>`；实现从 `:0` 开始即可，不伪造抓包中的真实窗口序号。
- Turn ID 和 `turn_started_at_unix_ms`：请求级生成。
- `x-client-request-id`、`session-id` 和 `prompt_cache_key` 都等于 Session ID。
- Header 的 `thread-id`、`x-codex-window-id`、`x-codex-turn-metadata` 必须与 `client_metadata` 同源。
- 普通 Turn Metadata 固定公开字段为 `request_kind=turn`、`thread_source=user`、`sandbox=none`、`workspace_kind=project`；不得注入真实本机路径、Git 地址、提交哈希或工作区状态。

### 4.3 Responses Body

命中 Profile 后：

- 强制 `store=false` 和 `stream=true`。
- 确保 `include` 包含 `reasoning.encrypted_content`。
- 缺省时补 `reasoning.context=all_turns` 和 `text.verbosity=low`。
- 保留客户端明确提供的 reasoning effort/summary、`parallel_tool_calls`、`tool_choice` 和 `stream_options`；抓包观察到的 `high`、`detailed`、`false`、`auto`、`sequential_cutoff` 不是所有调用的强制值。
- 写入 `prompt_cache_key` 和 `client_metadata`，并遵守前述同源关系。
- 把 `input` 中裸 role/content 消息规范为 Responses 消息结构，把 system role 改为 developer role。
- 不注入 Codex 基础 Instructions、隐藏 Prompt、产品 Memory、工作区或账号状态；用户已有 Instructions 保持原语义。
- 不固定 `x-openai-internal-codex-responses-lite`、`x-oai-attestation`、`x-openai-subagent` 或移动远程客户端后缀。

### 4.4 Compact

当前 Codex Desktop 的手工 `/compact` 命令没有调用 HTTP `/v1/responses/compact`。关闭 Provider WebSocket 后，抓包仍显示普通 `POST /v1/responses`，其中 Turn Metadata 使用：

- `request_kind=compaction`
- `compaction.implementation=responses_compaction_v2`
- `compaction.phase=standalone_turn`
- `compaction.reason=user_requested`
- `compaction.strategy=memento`
- `compaction.trigger=manual`

因此必须区分“Codex UI 命令”与“Responses Compact API”。项目已有 `/v1/responses/compact` 仍作为兼容端点保留，其首版规则明确标记为 `reference_derived`：

- `POST /v1/responses/compact`，上游 unary JSON，`Accept: application/json`。
- Body 只保留 `model`、`input`、`instructions`、`tools`、`parallel_tool_calls`、`reasoning`、`text`、`previous_response_id`。
- 删除 `stream`、`store`、`prompt_cache_key`、`client_metadata` 及其他普通 Responses 字段。
- 不应用普通 Responses Body 默认模板，不注入客户端 Metadata 或隐藏 Instructions。
- 身份 Header 仍使用当前 Codex Profile；响应保持现有独立 unary JSON 分支。

该端点不得被描述为当前 0.145.0-alpha.18 客户端抓包已验证。以后取得真实新版本样本时，必须发布新的 Profile ID，不原地改写本契约。

## 5. 与 sub2apiplus 参考实现的差异

| 项目 | 参考实现 | 本契约决定 |
|---|---|---|
| Claude 版本 | 2.1.209 / Agent SDK 0.3.209 | 更新为 2.1.215 / 0.3.215 |
| Claude Messages 鉴权 | 同时发送 `x-api-key` 和 Bearer | Messages 采用双鉴权并经 AnyRouter 实测；Count Tokens 只发送 `x-api-key` |
| Claude 压缩声明 | 固定 `gzip, deflate, br, zstd` | 当前观察为 `gzip, br`，首版交给现有 Transport |
| Claude Beta 尾部 | 旧版本按端点使用不同尾部 | 当前按 1M 基线固定六项；结构化输出替换末尾 fallback |
| Claude System | 参考实现会加入完整产品扩展 | 只允许公开身份块并保留用户语义，不复制不透明 Prompt |
| Codex 版本 | 0.144.0-alpha.4 | 更新为 0.145.0-alpha.18 |
| Codex UA 来源 | 旧 Desktop build | 使用关闭手机远程后的本地 Desktop build |
| Codex Instructions | 缺省时注入内置基础 Prompt | 明确禁止隐藏 Instructions 注入 |
| Codex Compact | 独立 HTTP 路由规则 | 保留该规则，但标记为参考实现派生、当前客户端未发出 |
| 未知 Profile | 默认回退 Desktop | fail-closed，拒绝未知、废弃或 Provider 不匹配 ID |

其余可复用部分包括：请求级工具 reverse map、结构化响应回写、稳定身份种子、请求级 Turn、Compact allowlist、受保护 Header 最终器以及官方客户端跳过逻辑。实现时应移植完整算法并适配本项目架构，不建立对 `.references` 的运行或构建依赖。

## 6. Golden Fixture 使用规则

- [`claude-desktop-2.1.215-v1.golden.json`](fixtures/claude-desktop-2.1.215-v1.golden.json) 冻结 Claude Header、Beta、端点和 Body 策略。
- [`codex-desktop-0.145.0-alpha.18-v1.golden.json`](fixtures/codex-desktop-0.145.0-alpha.18-v1.golden.json) 冻结 Codex Header、身份关系、Responses 和 Compact 策略。
- Fixture 中的占位符只表示生命周期关系，不得替换为真实抓包值。
- 后续实现测试必须验证“生成结果符合 Fixture”，不能把 Fixture 复制成第二套运行时常量。
- Profile 契约变化时新增文件和 Profile ID；已发布文件不得原地改变语义。
