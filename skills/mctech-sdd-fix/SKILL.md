---
name: mctech-sdd-fix
description: 根据 Verify 结果修复实现缺陷，并重新验证
stage: remediation
mode: write
input:
  - verification_report
  - change
  - task
  - proposal
  - design
  - specs
  - repository
output:
  - code_changes
  - test_changes
  - fix_report
side_effects:
  - source_code
  - tests
authority: latest_spec
next:
  - mctech-sdd-verify
  - mctech-sdd-change
---
你现在处于缺陷修复阶段。

请根据最新的 /mctech-sdd-verify 验证结果修复实现问题。

当前 OpenSpec Change：
{{CHANGE}}

当前 Task：
{{TASK}}

Verify Report：
{{VERIFY_REPORT}}

请严格按照以下流程执行：

## 1. 定位问题

找到 Verify 报告中的：

- FAIL
- FIX_REQUIRED
- IMPLEMENTATION_BUG
- CONTRACT_DRIFT

逐项分析。

对于每个问题：

1. 找到对应 Spec。
2. 找到对应 Task。
3. 找到实际代码。
4. 判断实际实现与 Spec 的差异。
5. 找出根因。

## 2. 判断是否允许直接修复

只有以下情况允许直接修复：

- IMPLEMENTATION_BUG
- TEST_DEFECT
- CONTRACT_DRIFT
- 当前 Task 范围内的实现问题

以下情况禁止自行修改：

- SPEC_DEFECT
- ARCHITECTURE_PROBLEM
- NEW_REQUIREMENT
- DESIGN_CHANGE
- OUT_OF_SCOPE

如果属于禁止自行修改的情况：

停止执行，并报告原因以及建议使用的下一步命令。

## 3. 执行修复

对于允许修复的问题：

1. 只修改问题涉及的代码。
2. 不扩大当前 Task 范围。
3. 不改变已经确认的架构决策。
4. 不修改 Spec。
5. 不为了让测试通过而降低业务约束。
6. 同时补充或修改测试。
7. 保持已有功能兼容。

## 4. 验证

修复完成后：

1. 执行失败的测试。
2. 执行相关单元测试。
3. 执行必要的集成测试。
4. 检查是否引入新的失败。

## 5. 输出

输出：

### Fixed

已经修复的问题。

### Changed Files

修改的文件。

### Tests

执行的测试及结果。

### Spec Compliance

说明修复后是否符合 Spec。

### Remaining Issues

仍然存在的问题。

### Next Action

只能从以下选项选择：

- VERIFY_AGAIN
- AI_CHANGE_REQUIRED
- DECISION_REQUIRED
- ENVIRONMENT_FIX_REQUIRED
- OUT_OF_SCOPE
