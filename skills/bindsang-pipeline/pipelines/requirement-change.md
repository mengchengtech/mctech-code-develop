# 需求变更

触发：明确提出需求变更，或描述为既有功能扩展、优化、行为改变。

`preflight → change-intake → explorer(分析) → (designer → reviewer)* → human-confirm → planner → (executor → verifier)*`

记录变更来源、前后行为、原因、受影响 ADR/OpenSpec/接口/数据、兼容迁移、灰度和回滚。执行中变更先暂停受影响任务，评估已完成工作，再重新设计审核和确认；历史决策不覆盖。
