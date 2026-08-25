---
name: mctech-sdd-explore
description: 探索当前系统实现，建立可靠的 Current State，不修改代码
stage: discovery
mode: read-only
input:
  - user_request
  - repository
  - existing_docs
output:
  - exploration
side_effects: none
authority: current_state
next:
  - mctech-sdd-analyze
  - mctech-sdd-spec
---

你现在处于系统探索阶段。
本阶段严禁修改任何代码、配置、数据库结构和测试文件，也不要提出具体重构方案。

目标：
理解当前代码库中与当前任务相关的真实实现。

当前任务：
{{TASK}}

要求：

1. 阅读相关代码、配置、数据库模型、测试和已有文档。
2. 找出功能涉及的模块、入口、核心类、方法和依赖关系。
3. 分析当前业务流程和数据流。
4. 找出所有相关 API、Service、Repository、Middleware、Interceptor 等入口。
5. 对跨模块功能，检查模块之间的调用关系。
6. 搜索重复实现、特殊实现和潜在遗漏。
7. 找出当前实现中的问题和技术债。
8. 对无法确认的内容明确标记为“待确认”。

特别要求：

- 只描述代码事实，不要自行设计新方案。
- 区分“已确认事实”和“推测”。
- 不修改任何代码。
- 不执行重构。
- 不因为发现问题就立即提出解决方案。

输出：

1. 当前架构
2. 数据模型
3. 核心流程
4. 模块依赖
5. 关键代码入口
6. 相关 API
7. 相关配置/缓存/数据库
8. 测试情况
9. 当前问题
10. 待确认问题

尽可能给出具体的文件、类、方法和代码位置。
