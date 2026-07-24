# 组件基线与上游同步

本文记录 CLIProxyAPI Pro 使用的组件版本、远端关系和上游同步规则。顶层仓库通过 Git 子模块锁定两个组件的精确提交，任何组件升级都必须通过独立变更集完成。

## 当前基线

基线日期：2026-07-24

| 组件 | fork | 官方上游 | 锁定提交 | 版本描述 |
|---|---|---|---|---|
| CLIProxyAPI | `https://github.com/itv3/CLIProxyAPI.git` | `https://github.com/router-for-me/CLIProxyAPI.git` | `b6c4cb4601bd1e7413d54c05be7c6a1ee379b436` | `v7.2.97-1`（上游基线 `v7.2.97`） |
| CPA-Manager-Plus | `https://github.com/itv3/CPA-Manager-Plus.git` | `https://github.com/seakee/CPA-Manager-Plus.git` | `8f43c7ce98bc9261e4732d8c726c9629c5d551df` | `v1.11.6-1`（上游基线 `v1.11.6`） |

## 远端约定

每个组件统一使用以下远端名称：

- `origin`：`itv3` 名下的 fork，只用于 Pro 分支和提交推送。
- `upstream`：官方仓库，只用于获取和合并官方更新。

本地应禁止向 `upstream` 推送，避免误操作官方仓库。

## 上游同步流程

1. 在组件仓库执行 `git fetch upstream --prune`。
2. 从 Pro 开发分支创建独立的上游同步分支。
3. 合并或变基到目标 `upstream/main`，单独处理冲突。
4. 执行组件自身测试、顶层 CI 和 Pro 兼容测试。
5. 更新本文件中的锁定提交和版本描述。
6. 在顶层仓库提交新的子模块指针，不将功能开发与上游同步混入同一提交。

## 最小侵入约束

- 统一账号的稳定 UUID、绑定历史、草稿状态机和聚合数据只存放在 CPA-Manager-Plus。
- CLIProxyAPI 不新增 Pro 账号数据库，不向用量事件写入 `pro_account_id`，不新增模型请求代理路径。
- CLIProxyAPI 的 Pro 相关能力必须实现为不包含 Pro 业务概念的通用能力。
- 账号级模型白名单集中接入 Auth/模型 Registry 和调度路径，不在各 Executor 或 Translator 中重复实现。
- OAuth 草稿能力集中在 Hook、OAuth Session 和共享持久化策略，不改变普通 Disabled 凭证的既有语义。
