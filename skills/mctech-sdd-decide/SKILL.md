---
name: mctech-sdd-decide
description: Record and formalize architecture decisions for an SDD change after the solution has been reviewed and approved. Use to establish constraints that later specification and implementation must follow.
argument-hint: "[decision-or-topic]"
disable-model-invocation: true
---

# MCTech SDD Decide

你现在处于 SDD 的架构决策阶段。

## 目标

将已经确认的架构方案和关键技术取舍正式记录下来，作为后续 Spec 和实现的约束。

决策主题：

$ARGUMENTS

## 重要原则

你不是架构决策者。

只有以下内容可以被记录为最终决策：

- 用户明确批准的方案
- 已存在且明确有效的架构决策
- 明确的项目级约束

如果用户没有明确选择方案，而不同方案会产生重大架构影响：

不要自行决定。

应停止并提出 Decision Required。

## 决策内容

### Context

为什么需要这个决策。

### Decision

最终采用什么方案。

### Alternatives

考虑过哪些方案。

### Rejected Alternatives

为什么不采用其他方案。

### Architecture Constraints

后续实现必须遵守的架构约束。

### API Constraints

API 相关约束。

### Data Constraints

数据库、数据模型和数据一致性约束。

### Compatibility Constraints

兼容性要求。

### Migration

迁移要求。

### Rollback

回滚要求。

### Non-functional

Requirements

包括：

- 性能
- 安全
- 可用性
- 可观测性

## 冲突检查

检查：

- 当前 Decision 与已有 Design 是否冲突
- 与已有 Spec 是否冲突
- 与已有架构决策是否冲突

如果存在冲突：

不要自行覆盖旧决策。

明确报告冲突。

## 输出格式

# Architecture Decision

## Context

## Decision

## Alternatives

## Rejected Alternatives

## Architecture Constraints

## API Constraints

## Data Constraints

## Compatibility Constraints

## Migration

## Rollback

## Non-functional Requirements

## Conflicts

如果项目使用 ADR 或 OpenSpec Design 文件，应按照项目现有结构写入。

不要修改业务代码。