# 角色定义

各角色详细契约见 `agents/role-*.md`。委派时须指定**角色 + 模式**（若适用）及输入/输出路径。

## Subagent 注册

仓库已在 **`.cursor/agents/role-*.md`** 注册与下列契约对齐的 Cursor 子 Agent（`name` / `pipeline-role` 与表一致）；正文精简，完整规则仍以 `agents/role-*.md` 为准。

| 角色 | 契约文件 | Cursor 注册 | `name` | `pipeline-role` | 权限类 |
|---|---|---|---|---|---|
| explorer | `agents/role-explorer.md` | `.cursor/agents/role-explorer.md` | `role-explorer` | `explorer` | 只读 |
| designer | `agents/role-designer.md` | `.cursor/agents/role-designer.md` | `role-designer` | `designer` | 文档写入 |
| reviewer | `agents/role-reviewer.md` | `.cursor/agents/role-reviewer.md` | `role-reviewer` | `reviewer` | 只读 |
| planner | `agents/role-planner.md` | `.cursor/agents/role-planner.md` | `role-planner` | `planner` | 文档写入 |
| executor | `agents/role-executor.md` | `.cursor/agents/role-executor.md` | `role-executor` | `executor` | 改源码 |
| verifier | `agents/role-verifier.md` | `.cursor/agents/role-verifier.md` | `role-verifier` | `verifier` | 只读/验证命令 |

自定义 agent 的 `description` 建议含 `pipeline-role: <上表角色>`，以便主 Agent 按 `references/orchestrator-protocol.md` 解析。`tools` 须与权限类一致（改源码类必须有 Edit；只写产物类不得 Edit）。

| 角色 | Agent 文件 | 权限类 | 模式（主 Agent 派发） | 独立于 | 产物 | 返回 |
|---|---|---|---|---|---|---|
| explorer | `agents/role-explorer.md` | 只读 | `explore` / `analyze` / `bug-triage` / `change-impact` | designer、reviewer、planner、executor、verifier | 探索/分析/变更影响/BUG ADR | `COMPLETED` / `NEED_INPUT` / `BLOCKED` / `FAILED` |
| designer | `agents/role-designer.md` | 文档写入 | `decide-record` / `spec` / `change-update` / `revise` / `recheck` | explorer、reviewer、planner、executor、verifier | 决策与设计 ADR、OpenSpec | 同上；`recheck` → `CONFIRM`/`MINOR`/`ISSUE` |
| reviewer | `agents/role-reviewer.md` | 只读 | `design` / `decision` / `change` | designer、executor | 审核 ADR | `APPROVE` / `REVISE` / `REJECT` / `NEED_INFO` |
| planner | `agents/role-planner.md` | 文档写入 | `plan` / `replan` | explorer、designer、reviewer、executor、verifier | 实施计划 ADR、`tasks.md` | 同 explorer 状态集 |
| executor | `agents/role-executor.md` | 代码写入（限定范围） | `implement` / `fix` / `concurrent` / `integrate` / `resume` | reviewer、verifier | 代码差异、执行记录 | `COMPLETED` / `BLOCKED(<类型>)` / `FAILED` |
| verifier | `agents/role-verifier.md` | 只读/验证命令 | `task` / `change` | executor、designer | 验证报告 | `PASS` / `FAIL` / `BLOCKED` / `SCOPE_CHANGE` |

## 阶段与角色模式（派发参考）

| 阶段语义 | 角色 · 模式 |
|---|---|
| 探索 | explorer · `explore` |
| 分析 | explorer · `analyze` |
| 决策落稿 | designer · `decide-record`（批准来自主 Agent / 用户） |
| 规范与设计 | designer · `spec` |
| 规划 | planner · `plan` |
| 实施 | executor · `implement` |
| 缺陷修复 | executor · `fix` |
| 验证 | verifier · `task` / `change` |
| 需求变更 | explorer · `change-impact` → designer · `change-update` → planner · `replan` |
