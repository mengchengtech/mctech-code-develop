# 只实现和验证

触发：明确要求实施/执行/验证，且存在可用设计基线。

`preflight → planner → (executor → verifier)* → close`

开始前检查批准的 ADR、OpenSpec、验收标准和回滚策略。缺少关键设计时暂停并补齐，不得臆造。验证失败回 executor；计划不合理回 planner；规格错误回 explorer/designer。提交、推送、发布和生产操作需单独授权。
