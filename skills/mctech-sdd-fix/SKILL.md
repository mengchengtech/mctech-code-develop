---
name: mctech-sdd-fix
description: Fix implementation defects identified by MCTech SDD Verify without changing approved requirements, specifications, architecture, or task scope. Use after verification reports an implementation bug, test defect, or contract drift.
argument-hint: "[verification-issue-or-task]"
disable-model-invocation: true
---

# MCTech SDD Fix

你现在处于 SDD 的缺陷修复阶段。

修复目标：

$ARGUMENTS

## 输入

必须优先读取最新：

1. Verification Report
2. OpenSpec
3. Spec
4. Design
5. Architecture Decision
6. 对应 Task
7. 当前代码

## 允许修复的问题

只允许直接处理：

- IMPLEMENTATION_BUG
- TEST_DEFECT
- CONTRACT_DRIFT
- CONFIGURATION_DEFECT
- 当前 Task 范围内的明确实现错误

## 禁止自行处理

以下问题不能直接修复：

- SPEC_DEFECT
- DESIGN_CHANGE
- ARCHITECTURE_PROBLEM
- NEW_REQUIREMENT
- OUT_OF_SCOPE

如果属于以上类型：

停止修改，并报告应该使用的下一步 Skill。

## 修复步骤

### 1. Locate

找到：

- Verify Issue
- 对应 Spec
- 对应 Task
- 实际代码

### 2. Root Cause

说明：

Spec
→ Expected Behavior
→ Actual Behavior
→ Root Cause

### 3. Minimal Fix

使用最小修改原则：

- 不扩大范围
- 不改变架构
- 不改变 Spec
- 不顺便重构无关代码

### 4. Tests

必须增加或调整能够复现问题的测试。

至少覆盖：

- 原失败场景
- 正常场景
- 相关边界场景

### 5. Verification

执行：

1. 原失败测试
2. 相关测试
3. 必要的集成测试
4. 必要的构建 / 类型检查

## 特别规则

如果为了修复问题必须改变 Spec：

不要修改 Spec。

停止并建议：

/mctech-sdd-change

如果必须改变架构：

/mctech-sdd-decide

如果需要修改另一个 Task：

停止并报告。

## 输出

# Fix Report

## Issue

## Classification

## Root Cause

## Fix

## Changed Files

## Tests Added / Changed

## Tests Executed

## Result

## Spec Compliance

## Remaining Issues

## Next Action

通常：

/mctech-sdd-verify
