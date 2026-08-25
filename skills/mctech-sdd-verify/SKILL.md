---
name: mctech-sdd-verify
description: 根据最新 OpenSpec 验证实现完整性、规范符合性和发布风险
stage: verification
mode: read-only
input:
  - change
  - proposal
  - design
  - specs
  - tasks
  - repository
  - test_results
output:
  - verification_report
  - compliance_report
  - risks
side_effects: none
authority: verification
next:
  - mctech-sdd-implement
  - archive
---

你现在处于验证阶段。

当前 OpenSpec Change：
{{CHANGE}}

验证目标：
{{SCOPE}}

请以最新 OpenSpec Spec 为主要验收依据，对当前代码进行验证。

检查：

1. Spec 是否全部实现
2. Design 是否得到正确实现
3. Tasks 是否完成
4. 是否存在 Spec Drift
5. 是否存在遗漏的业务场景
6. 是否存在权限绕过
7. 是否存在兼容性问题
8. 是否存在数据库/API/缓存问题
9. 测试是否覆盖关键场景
10. 是否存在旧实现残留
11. 是否存在未记录的新实现

请区分：

PASS
FAIL
RISK
NOT VERIFIED

如果发现问题：

1. 给出具体文件、类、方法。
2. 说明违反哪个 Spec。
3. 判断该问题是否属于当前 Task 的职责范围。
4. 说明违反了哪个 Spec
5. 说明影响
6. 给出修复建议

默认不要直接修改代码。

最后输出：

1. Spec Compliance
2. Implementation Completeness
3. Test Coverage
4. Compatibility
5. Security / Risk
6. Remaining Issues
7. Release Recommendation
8. Final Report

Final Report格式如下：

PASS: 42
FAIL: 1
RISK: 5
NOT VERIFIED: 3

## Required Actions

### 需要修复

1. [IMPLEMENTATION_BUG]
  org-management 换公司未校验 grants
  Action:
  /ai-fix Task3
2. [TEST_DEFECT]
    .........
  Action:
  /ai-fix

### 需要决策

1. [DB_CONSTRAINT | SPEC_DEFECT | DESIGN_CHANGE | ARCHITECTURE_PROBLEM | NEW_REQUIREMENT]
  Sequelize 未声明复合唯一索引
  Question:
  生产数据库约束的 Source of Truth 是 migration/DDL
  还是 Sequelize Model？

### 延期处理

1. [CLEANUP | OUT_OF_SCOPE]
  UDP Model 仍注册
  Reason:
  属于 §7 Cleanup
2. [OUT_OF_SCOPE]
  service-operations 仍读取 UDP

### 未经验证的
1. [ENVIRONMENT]
  Playwright 浏览器缺失
