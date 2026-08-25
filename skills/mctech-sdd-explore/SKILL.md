---
name: mctech-sdd-explore
description: Explore the existing system before making design or implementation decisions. Use to understand current architecture, modules, call flows, data flows, APIs, dependencies, tests, legacy behavior, and technical constraints for an SDD change.
argument-hint: "[feature-or-module]"
disable-model-invocation: true
---

# MCTech SDD Explore

你现在处于 SDD 的系统探索阶段。

## 目标

建立当前系统的可靠事实，而不是设计解决方案。

探索目标：

$ARGUMENTS

## 核心原则

1. 只基于实际代码、配置、文档、测试和数据库定义判断。
2. 不把推测当成事实。
3. 不修改业务代码。
4. 不修改 OpenSpec。
5. 不提前确定最终架构方案。
6. 对无法确认的问题明确标记。

## 探索范围

根据任务需要检查：

- 项目目录结构
- 模块职责
- API / Controller
- Service
- Repository / DAO
- Middleware / Interceptor
- 权限和认证逻辑
- 数据库模型和 DDL
- 缓存
- 消息队列
- 外部服务
- 前端调用
- 配置
- 测试
- 定时任务
- 异步流程
- 错误处理
- 日志和监控

对于跨模块功能，必须追踪：

入口
→ Controller / API
→ Service
→ 核心业务逻辑
→ 数据访问
→ 缓存 / MQ / 外部服务

必要时同时分析反向依赖：

- 谁调用当前模块？
- 当前模块调用谁？

## 重点检查

### 1. Current State

说明当前系统实际上如何工作。

### 2. Module Map

列出相关模块及职责。

### 3. Call Flow

描述关键调用链。

### 4. Data Flow

描述关键数据流。

### 5. API

列出相关接口、请求、响应和兼容行为。

### 6. Data Model

列出相关：

- 表
- 字段
- 索引
- ORM Model
- 数据关系

### 7. Dependencies

包括：

- Redis
- MQ
- 外部 API
- 服务间调用
- 配置中心

仅列实际存在的依赖。

### 8. Existing Tests

说明：

- 已有测试
- 覆盖场景
- 缺失场景

### 9. Legacy / Compatibility

重点寻找：

- 旧实现
- 兼容代码
- Deprecated API
- Feature Flag
- 双写 / 双读
- 迁移逻辑

### 10. Problems

记录已经确认的问题。

不要把潜在问题描述成确定问题。

## 输出格式

# Exploration Report

## 1. Current State

## 2. Architecture / Module Map

## 3. Call Flow

## 4. Data Flow

## 5. API

## 6. Data Model

## 7. Dependencies

## 8. Existing Tests

## 9. Legacy / Compatibility

## 10. Confirmed Problems

## 11. Confirmed Facts

## 12. Open Questions

## 13. Potential Risks

每个重要结论尽可能提供：

- 文件路径
- 类 / 函数
- 配置项
- 数据表
- 测试名称

不要修改任何文件。