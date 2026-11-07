# dev-pipeline（已弃用）

本目录为历史 **调度协议 + 三条流水线**（`dev` / `design-review` / `impl-verify`），产物落在仓库根 `.pipeline/<yyyymmdd>-slug>/`。

**请改用：[`skills/bindsang-pipeline`](../bindsang-pipeline/)**

| 本目录 | bindsang-pipeline |
|---|---|
| `SKILL.md` 调度协议 | `references/orchestrator-protocol.md` + `SKILL.md` |
| `pipelines/dev.md` | `pipelines/full-flow.md` |
| `pipelines/design-review.md` | `pipelines/design-review.md` |
| `pipelines/impl-verify.md` | `pipelines/implement-verify.md` |
| `agents/pipeline-*.md` | `agents/role-*.md`（含 `pipeline-role`） |
| `00-task.md` / `decisions.md` | `<change-root>/.pipeline/brief.md` / `decisions-log.md` |

根目录说明见仓库 [`README.md`](../../README.md)「Agent 流水线（推荐入口）」。

保留本目录仅供对照迁移；新 Change 不要再加载 `dev-pipeline` Skill。
