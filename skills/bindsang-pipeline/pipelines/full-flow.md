# 全流程开发

适用：新功能、大型重构、跨模块改造。

`preflight → explorer(探索/分析) → (designer → reviewer)* → human-confirm → planner → (executor → verifier)* → close`

审核 `REVISE` 回 designer；`REJECT` 回 explorer 分析；`NEED_INFO` 回 explorer 补证据。验证 `FAIL` 回 executor，`SCOPE_CHANGE` 回 explorer/designer。L1 可合并探索分析并轻量审核；L2 标准执行；L3 多轮审核、里程碑规划和多批验证。
