---
name: mctech-sdd-analyze
description: Analyze a feature or refactoring problem based on the existing system and identify root causes, constraints, candidate solutions, trade-offs, migration risks, and recommended approaches before architecture decisions are made.
argument-hint: "[problem-or-feature]"
disable-model-invocation: true
---

# MCTech SDD Analyze

你现在处于 SDD 的方案分析阶段。

## 目标

从当前系统事实出发，分析问题根因并提出可比较的解决方案。

分析目标：

$ARGUMENTS

## 核心原则

1. 基于实际代码和 Exploration 结果。
2. 先分析问题，再讨论方案。
3. 至少比较两个合理方案，除非确实只有一个可行方案。
4. 推荐方案不等于最终架构决策。
5. 不修改业务代码。
6. 不修改 OpenSpec。
7. 不自行批准重大架构变化。

## 分析步骤

### 1. Problem Definition

明确：

- 当前行为
- 期望行为
- 两者差异
- 为什么需要改变

### 2. Root Cause

分析真正根因。

避免把表象问题直接当成根因。

### 3. System Boundary

明确：

- 哪些模块属于本次变化
- 哪些模块不属于本次变化
- 哪些系统是外部依赖

### 4. Candidate Solutions

提出 2～3 个可行方案。

每个方案说明：

- 工作方式
- 修改模块
- 优点
- 缺点
- 风险
- 实施复杂度

### 5. Compare Solutions

至少比较：

- 正确性
- 安全性
- 性能
- 复杂度
- 可维护性
- 兼容性
- 改造成本
- 测试成本
- 迁移成本
- 回滚难度

大型重构还必须分析：

- API 兼容
- 数据迁移
- 缓存迁移
- 前端兼容
- 服务间兼容
- 灰度发布
- 回滚

### 6. Recommended Solution

给出推荐方案及理由。

明确：

该推荐方案仍需要架构决策确认。

### 7. Decision Required

列出必须由架构师确认的问题。

## 输出格式

# Analysis Report

## 1. Problem Definition

## 2. Root Cause

## 3. Scope

## 4. Constraints

## 5. Solution A

## 6. Solution B

## 7. Solution C

## 8. Comparison

## 9. Recommended Solution

## 10. Risks

## 11. Migration Considerations

## 12. Rollback Considerations

## 13. Decision Required

不要修改代码或 OpenSpec。