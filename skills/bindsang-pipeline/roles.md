# 角色定义

各角色详细契约见 `agents/role-*.md`。委派时须指定**角色 + 模式**（若适用）及输入/输出路径。

## 环境子代理注册

子 Agent 按环境子代理注册（本仓库为 `~/.zcode/agents/role-*.md`，`name` / `pipeline-role` 与下表一致）；注册副本正文精简，完整规则仍以 `agents/role-*.md` 为准。

- 各角色默认推理模型在注册文件 frontmatter 的 `model`（或环境等价字段）中配置，随子 Agent 创建/注册时设定；主 Agent 派发只指定 `name`（或 `subagent_type`）与任务块（见 `references/orchestrator-protocol.md`「派发块」），**默认不传 `model`**。仅当用户在 prompt 或 `brief.md` → `role_models` 中显式指定某 `pipeline-role` 使用某模型时，方可对该角色当次派发覆盖 `model`（解析链见 `references/orchestrator-protocol.md`「角色解析」）。
- **回退分级**：单个角色注册缺失 → 用宿主通用 agent（general-purpose）并在派发块首行注入契约文件路径兜底；回退后**全部角色**均落宿主通用 agent → **停下询问用户**（指定 agent / 接受降级并记 `decisions-log.md` / 终止），不得静默降级开工。
- **独立性**：互为「独立于」的角色不得同一注册兼任；默认绑定或回退至同一 agent 时，必须派发为**全新独立实例**，输入只含文件路径、不含所列角色的推理或自检结论。设计评审 / 实施 / 验证三槽位的独立性等级（`异构` / `同模型独立` / `宿主默认`）记入 `brief.md` 的 `independence`。
- 自定义 agent 的 `description` 建议含 `pipeline-role: <下表角色>`，以便主 Agent 按 `references/orchestrator-protocol.md`「角色解析」解析。`tools` 须与权限类一致（改源码类必须有 Edit；只写产物类不得 Edit）。

## 角色与模式（本表是模式词表的唯一来源，契约与流水线不得增删模式名）

| 角色 | 契约文件 | `name` / `pipeline-role` | 权限类 | 模式（主 Agent 派发） | 阶段语义 | 独立于 | 产物 | 返回 |
|---|---|---|---|---|---|---|---|---|
| explorer | `agents/role-explorer.md` | `role-explorer` / `explorer` | 只读 | `explore` / `analyze` / `bug-triage` / `change-impact` | 探索 · `explore`；分析 · `analyze`；需求变更 · `change-impact` | designer、reviewer、planner、executor、verifier | 探索/分析/变更影响/BUG ADR | `COMPLETED` / `NEED_INPUT` / `BLOCKED` / `FAILED` |
| designer | `agents/role-designer.md` | `role-designer` / `designer` | 文档写入 | `decide-record` / `spec` / `change-update` / `revise` / `recheck` | 决策落稿 · `decide-record`（批准来自主 Agent / 用户）；规范与设计 · `spec`；需求变更 · `change-update` | explorer、reviewer、planner、executor、verifier | 决策与设计 ADR、OpenSpec | 同 explorer 状态集；`recheck` → `CONFIRM`/`MINOR`/`ISSUE` |
| reviewer | `agents/role-reviewer.md` | `role-reviewer` / `reviewer` | 只读 | `design` / `decision` / `change` | — | designer、executor | 审核 ADR | `APPROVE` / `REVISE` / `REJECT` / `NEED_INFO` |
| planner | `agents/role-planner.md` | `role-planner` / `planner` | 文档写入 | `plan` / `replan` | 规划 · `plan`；需求变更 · `replan` | explorer、designer、reviewer、executor、verifier | 实施计划 ADR、`tasks.md` | 同 explorer 状态集 |
| executor | `agents/role-executor.md` | `role-executor` / `executor` | 代码写入（限定范围） | `implement` / `fix` / `concurrent` / `integrate` / `resume` | 实施 · `implement`；缺陷修复 · `fix` | reviewer、verifier | 代码差异、执行记录 | `COMPLETED` / `BLOCKED(<类型>)` / `FAILED` |
| verifier | `agents/role-verifier.md` | `role-verifier` / `verifier` | 只读/验证命令 | `task` / `change` | 验证 · `task` / `change` | executor、designer | 验证报告 | `PASS` / `FAIL` / `BLOCKED` / `SCOPE_CHANGE` |

需求变更链：explorer · `change-impact` → designer · `change-update` → planner · `replan`。
