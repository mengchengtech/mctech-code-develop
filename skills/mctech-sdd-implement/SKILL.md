---
name: mctech-sdd-implement
description: 严格按照 OpenSpec Task 实现代码并完成相关测试
stage: implementation
mode: write
input:
  - change
  - proposal
  - design
  - specs
  - decisions
  - task
  - repository
output:
  - code_changes
  - tests
  - implementation_report
side_effects:
  - source_code
  - tests
  - build_artifacts
authority: task
next:
  - mctech-sdd-verify
  - mctech-sdd-change
---

你现在处于实施阶段。

当前 OpenSpec Change：
{{CHANGE}}

当前 Task：
{{TASK}}

执行前必须阅读：

1. 最新 proposal
2. 最新 design
3. 最新 specs
4. 当前 Task
5. 相关 Decision
6. 当前代码状态

要求：

1. 只执行当前 Task。
2. 严格遵循最新 Spec。
3. 不自行扩大任务范围。
4. 不自行改变架构。
5. 不修改其他 Task 的职责。
6. 修改代码时同时补充或修改测试。
7. 执行相关测试、lint、typecheck 或构建检查。
8. 保持已有功能兼容，除非 Spec 明确要求改变。

如果发现：

- Spec 与代码冲突
- 需求无法实现
- 需要修改架构
- 需要修改其他 Task
- 发现未记录的重要业务规则

必须停止执行并报告。

不要自行做架构决策。

完成后输出：

1. 修改文件
2. 实现内容
3. 测试结果
4. 是否符合 Spec
5. 是否影响其他 Task
6. 遗留问题

只有当前 Task 完成后才可以结束。
