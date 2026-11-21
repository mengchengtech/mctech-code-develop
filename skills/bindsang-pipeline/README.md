# bindsang-pipeline（Agent + Subagent 开发流水线）

仓库 **推荐的多 Agent 编排入口**（根 [`README.md`](../../README.md)）。OpenSpec Change + `.pipeline/` 编排；阶段能力定义于 `agents/role-*.md`。

- **[`USAGE.md`](USAGE.md)**：使用场景、启动方式（Skill / Prompt 模板）、检查点口令与子 Agent 注册说明。
- `SKILL.md`：主入口、触发条件与总判据、加载图、任务分类、复杂度裁剪、自定义角色组合和调度规则。
- `roles.md`：角色索引表（角色、契约文件、权限类、**模式词表唯一来源**、独立性、产物、返回值）与环境子代理注册。
- `agents/role-*.md`：角色职责、规则、约束、**模式**、输出结构、返回码与完成判据；含 `pipeline-role` / `tools` frontmatter。
- 环境子代理注册（本仓库为 `~/.zcode/agents/role-*.md`）：六个角色子 Agent（`model` 在注册文件配置；主 Agent 派发默认不传 `model`，见 `references/orchestrator-protocol.md`）。
- `pipelines/`：`full-flow`、`design-review`、`implement-verify`、`requirement-change`、`bug-feedback`（阶段表、记账键预算、检查点见各文件）。
- `references/orchestrator-protocol.md`：调度协议（派发块、硬约束、启动窗口、记账、并发、恢复、结束汇报、维护）。
- `references/`：编排协议与记账（`orchestrator-protocol.md`）、产物规范（`artifacts.md`）、决策路由（`decision-routing.md`）、**Git 检查点**与 `commit_policy` 三态（`git-checkpoints.md`）。

Change 根目录（`changes/<id>/` 或 `openspec/changes/<id>/`）存放 OpenSpec；流水线状态与各阶段 ADR 写在 `<change-root>/.pipeline/` 下（见 `references/artifacts.md`）。

本 Skill 不绑定具体 Agent 工具；支持原生多 Agent 委派，也可由单 Agent 按角色契约顺序模拟执行。
