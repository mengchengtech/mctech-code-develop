# 主 Agent 与子 Agent 交互协议

委派请求必须包含：Change ID、阶段、轮次、角色、目标、输入文件、约束、输出路径和完成判据。子 Agent 回执必须包含：状态（COMPLETED/NEED_INPUT/BLOCKED/FAILED）、产物路径、摘要、证据、未决事项和建议路由。建议路由仅供参考，只有主 Agent 可以验收产物并推进状态。

```yaml
protocol_version: "1.0"
message_type: "TASK"
change_id: "YYYYMMDD-example"
stage: "analysis"
role: "explorer"
inputs: ["brief.md"]
constraints: ["只读，不修改业务代码"]
outputs: ["analysis/ADR-需求与影响分析.md"]
acceptance: ["结论区分事实、推断和未知"]
```
