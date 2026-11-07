# bindsang-pipeline（Agent + Subagent 开发流水线）

仓库 **推荐的多 Agent 编排入口**（根 [`README.md`](../../README.md)）。OpenSpec Change + `.pipeline/` 编排；阶段能力定义于 `agents/role-*.md`。

- **[`USAGE.md`](USAGE.md)**：使用场景、启动方式（Skill / Prompt 模板）、检查点口令与子 Agent 注册说明。
- `SKILL.md`：主入口、触发条件、任务分类、复杂度裁剪、自定义角色组合和调度规则。
- `roles.md`：角色索引表，仅保留角色、Agent 文件、权限类、模式、独立性、产物和返回值。
- `agents/role-*.md`：角色职责、规则、约束、**模式**、输出结构、返回码与完成判据；含 `pipeline-role` / `tools` frontmatter 供子 Agent 注册。
- `pipelines/`：`full-flow`、`design-review`、`implement-verify`、`requirement-change`、`bug-feedback`（阶段表、预算、检查点见各文件）。
- `references/orchestrator-protocol.md`：调度协议（派发、硬约束、并发、恢复）。
- `references/`：交互协议、产物规范、状态机、回滚和反馈路由。

Change 根目录（`changes/<id>/` 或 `openspec/changes/<id>/`）存放 OpenSpec；流水线状态与各阶段 ADR 写在 `<change-root>/.pipeline/` 下（见 `references/artifacts.md`）。

本 Skill 不绑定具体 Agent 工具；支持原生多 Agent 委派，也可由单 Agent 按角色契约顺序模拟执行。
