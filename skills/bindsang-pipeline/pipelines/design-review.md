# 只设计评审

触发：明确要求出设计、方案评审或先设计不实现。

`preflight → explorer(探索/分析) → (designer → reviewer)* → human-confirm → awaiting-implementation`

不得修改业务代码。审核通过不等于用户批准。确认后无修改则保留 Change 等待实施；事实/需求问题回 explorer，方案问题回 designer，审核遗漏回 reviewer。用户新增目标时记录需求变更。
