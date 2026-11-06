# 外部 BUG 反馈

触发：用户报告 BUG、异常结果、崩溃、数据不一致或性能退化。

`preflight → bug-intake/reproduce/diagnose(explorer) → fix-design(designer) → reviewer(按风险) → human-confirm(按风险) → planner → (executor → verifier)*`

Explorer 记录预期/实际、环境版本、复现步骤、影响、日志证据和假设。无法复现则等待补充，不猜测修复。明确代码缺陷可轻量处理；涉及数据、安全、兼容或跨服务时完整审核确认。若实为业务规则改变，转需求变更。
