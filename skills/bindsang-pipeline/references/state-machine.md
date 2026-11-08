# 流水线状态机

状态文件为 `<change-root>/.pipeline/pipeline-state.yaml`，记录 schema_version、change_id、change_root、task_type、roles（自定义时）、complexity、status、current_stage、stage_round、Git 基线（`baseline_commit`）、`last_checkpoint_commit`、审批标志、产物清单（路径含 `.pipeline/` 前缀）、next_action 和更新时间。`brief.md` 可含 `commit_policy`（见 `git-checkpoints.md`）。

阶段状态：`not_started`、`in_progress`、`completed`、`needs_revision`、`blocked`、`skipped`。Change 状态：`in_progress`、`awaiting_human`、`awaiting_info`、`blocked`、`completed`、`completed_with_exceptions`、`cancelled`、`rolled_back`。仅主 Agent 更新状态；回退不删除后续产物，而是标记过期。恢复时核对状态文件、Git 基线、工作区和产物；不一致则暂停并报告。
