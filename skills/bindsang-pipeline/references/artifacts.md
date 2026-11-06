# 产物规范：ADR + OpenSpec

Change 目录按阶段分层：`exploration/`、`analysis/`、`decision/`、`review/`、`planning/`、`execution/`、`verification/`、`bug/`。只创建本 Change 实际需要的目录；文档内容中文，目录名英文小写。

ADR 至少包含：状态、Change、阶段、版本、日期、背景、目标/非目标、事实证据、分析/方案/审查内容、决策结论、影响权衡、风险缓解、验收标准、回滚方式、未决问题、关联文档。

OpenSpec 映射：`proposal.md` 保存背景动机和范围；`specs/<capability>/spec.md` 保存需求与验收场景；`design.md` 保存设计；`tasks.md` 保存实施计划。ADR 保存决策依据、审查、人工确认和验证证据。两者互相引用，不重复堆叠全文。每轮设计/审核/执行/验证产生新版本，旧 ADR 标记已替代，不删除。
