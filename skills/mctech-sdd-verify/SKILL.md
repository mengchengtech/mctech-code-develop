---
name: mctech-sdd-verify
description: Verify an implemented OpenSpec change against the latest specification, design, task requirements, tests, API contracts, data constraints, and security behavior. Classify failures and determine the correct next action.
argument-hint: "[task-id|change-name|final]"
disable-model-invocation: true
---

# MCTech SDD Verify

你现在处于 SDD 的规范验证阶段。

验证范围：

$ARGUMENTS

## 目标

判断实现是否符合最新 Spec，而不仅仅是判断测试是否通过。

## 权威顺序

验证时优先依据：

1. 最新批准的 Spec
2. Architecture Decision
3. Design
4. Task
5. Tests
6. 当前实现

测试不是需求本身。

如果测试与 Spec 冲突：

标记为 TEST_DEFECT。

不要为了测试通过而修改业务代码。

## 验证内容

### 1. Spec Coverage

确认每一个 Spec 是否已经实现。

### 2. Behavioral Verification

检查：

- 正常流程
- 异常流程
- 边界条件
- 权限拒绝
- 权限绕过
- 状态变化
- 并发
- 重试

### 3. API Contract

检查：

- Request
- Response
- Status Code
- Schema
- Validation
- Backward Compatibility

### 4. Data

检查：

- 表
- 字段
- 索引
- 唯一约束
- 数据一致性
- Migration

### 5. Cache

检查：

- 写入
- 读取
- 失效
- 更新
- 删除
- 权限变化后的缓存一致性

### 6. Security

对于认证、授权、权限相关 Change，重点检查：

- 未认证访问
- 无权限访问
- 权限继承
- 权限组合
- 租户切换
- 公司切换
- 身份变化
- 权限缓存
- 旧权限模型残留
- 绕过路径

### 7. Tests

执行相关测试。

必要时检查测试是否真正覆盖 Spec。

## 问题分类

每个问题必须归类为：

- IMPLEMENTATION_BUG
- TEST_DEFECT
- CONTRACT_DRIFT
- CONFIGURATION_DEFECT
- SPEC_DEFECT
- DESIGN_CHANGE
- ARCHITECTURE_PROBLEM
- NEW_REQUIREMENT
- ENVIRONMENT_PROBLEM
- OUT_OF_SCOPE
- UNKNOWN

## 状态

使用：

- PASS
- FAIL
- RISK
- NOT VERIFIED

## Required Action

### IMPLEMENTATION_BUG

/mctech-sdd-fix

### TEST_DEFECT

/mctech-sdd-fix

### CONTRACT_DRIFT

/mctech-sdd-fix

### SPEC_DEFECT

/mctech-sdd-change

### DESIGN_CHANGE

/mctech-sdd-decide

### ARCHITECTURE_PROBLEM

/mctech-sdd-decide

### NEW_REQUIREMENT

/mctech-sdd-change

### ENVIRONMENT_PROBLEM

先修复环境，然后重新 Verify。

### OUT_OF_SCOPE

记录并 Deferred。

## 重要原则

不要：

- 因为发现问题就修改代码
- 因为测试失败就改变 Spec
- 因为发现旧代码就扩大 Change
- 把 OUT_OF_SCOPE 当成当前 Change 失败
- 把 NOT VERIFIED 当成 FAIL

## 输出格式

# Verification Report

## Summary

PASS: X
FAIL: X
RISK: X
NOT VERIFIED: X

## PASS

## FAIL

每项包含：

- ID
- Severity
- Classification
- Spec
- Task
- Location
- Problem
- Impact
- Recommended Action

## RISK

## NOT VERIFIED

## DEFERRED

## OUT OF SCOPE

## Required Actions

最后给出：

NEXT_ACTION:

- /mctech-sdd-fix
- /mctech-sdd-change
- /mctech-sdd-decide
- ENVIRONMENT_FIX
- PASS
