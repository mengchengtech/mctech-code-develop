# 角色定义

各角色详细契约（含由 `skills/sdd-*` 合并的工作步骤与模式）见 `agents/role-*.md`。委派时须指定**角色 + 模式**（若适用）及输入/输出路径。

## Subagent 注册

| 角色 | agent 文件 | `name` | `pipeline-role` | 权限类 |
|---|---|---|---|---|
| explorer | `role-explorer.md` | `role-explorer` | `explorer` | 只读 |
| designer | `role-designer.md` | `role-designer` | `designer` | 文档写入 |
| reviewer | `role-reviewer.md` | `role-reviewer` | `reviewer` | 只读 |
| planner | `role-planner.md` | `role-planner` | `planner` | 文档写入 |
| executor | `role-executor.md` | `role-executor` | `executor` | 改源码 |
| verifier | `role-verifier.md` | `role-verifier` | `verifier` | 只读/验证命令 |

自定义 agent 的 `description` 建议含 `pipeline-role: <上表角色>`，以便主 Agent 按 `references/orchestrator-protocol.md` 解析。`tools` 须与权限类一致（改源码类必须有 Edit；只写产物类不得 Edit）。

| 角色 | Agent 文件 | 权限类 | 模式（主 Agent 派发） | 独立于 | 产物 | 返回 |
|---|---|---|---|---|---|---|
| explorer | `agents/role-explorer.md` | 只读 | `explore` / `analyze` / `bug-triage` / `change-impact` | designer、reviewer、planner、executor、verifier | 探索/分析/变更影响/BUG ADR | `COMPLETED` / `NEED_INPUT` / `BLOCKED` / `FAILED` |
| designer | `agents/role-designer.md` | 文档写入 | `decide-record` / `spec` / `change-update` / `revise` / `recheck` | explorer、reviewer、planner、executor、verifier | 决策与设计 ADR、OpenSpec | 同上；`recheck` → `CONFIRM`/`MINOR`/`ISSUE` |
| reviewer | `agents/role-reviewer.md` | 只读 | `design` / `decision` / `change` | designer、executor | 审核 ADR | `APPROVE` / `REVISE` / `REJECT` / `NEED_INFO` |
| planner | `agents/role-planner.md` | 文档写入 | `plan` / `replan` | explorer、designer、reviewer、executor、verifier | 实施计划 ADR、`tasks.md` | 同 explorer 状态集 |
| executor | `agents/role-executor.md` | 代码写入（限定范围） | `implement` / `fix` / `concurrent` / `integrate` / `resume` | reviewer、verifier | 代码差异、执行记录 | `COMPLETED` / `BLOCKED(<类型>)` / `FAILED` |
| verifier | `agents/role-verifier.md` | 只读/验证命令 | `task` / `change` | executor、designer | 验证报告 | `PASS` / `FAIL` / `BLOCKED` / `SCOPE_CHANGE` |

## SDD 能力映射（供迁移与派发参考）

| 原 `sdd-*` | 角色 · 模式 |
|---|---|
| sdd-explore | explorer · `explore` |
| sdd-analyze | explorer · `analyze` |
| sdd-decide | designer · `decide-record`（批准来自主 Agent / 用户） |
| sdd-spec | designer · `spec` |
| sdd-plan | planner · `plan` |
| sdd-implement | executor · `implement` |
| sdd-fix | executor · `fix` |
| sdd-verify | verifier · `task` / `change` |
| sdd-change | explorer · `change-impact` → designer · `change-update` → planner · `replan` |

独立 `/sdd-*` 调用宜逐步弃用；编排以 `bindsang-pipeline` 主 Skill + 上表模式为准。
