# 保存点、回滚与恢复

流程回退是重新探索/分析/设计/审核/规划，不自动撤销代码；代码回滚是撤销某批代码/配置/迁移，必须明确目标基线和数据影响。前置准备记录 commit、分支和工作区差异；每个文档阶段在 `.pipeline/` 保存 ADR，在 Change 根保存 OpenSpec，并更新 `.pipeline/pipeline-state.yaml`；每个执行批次保存代码差异、`.pipeline/execution/` 执行记录和测试结果。

默认只写本地工作区，不自动 commit/push。优先 `git revert` 或针对性反向补丁；不得未经授权执行 `git reset --hard`、强制清理、生产回滚或数据删除。数据库和外部副作用必须有备份、补偿或人工处置方案。
