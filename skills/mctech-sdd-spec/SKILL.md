---
name: mctech-sdd-spec
description: 将需求和已确认的架构决策转化为正式 OpenSpec Change
stage: specification
mode: read-only
input:
  - user_request
  - exploration
  - analysis
  - decisions
  - existing_openspec
output:
  - proposal
  - design
  - specs
side_effects: openspec_only
authority: specification
next:
  - mctech-sdd-plan
---

你现在处于 OpenSpec 规范阶段。

当前任务：
{{TASK}}

请基于：

1. Exploration
2. Analysis
3. 已确认的 Architecture Decision
4. 当前需求

创建或更新 OpenSpec Change。

要求：

### Proposal
说明：
- 为什么需要修改
- 当前问题
- 修改目标
- 非目标

### Design
说明：
- 整体架构
- 数据模型
- 核心流程
- API
- 模块职责
- 缓存
- 兼容性
- 迁移
- 回滚

### Spec
描述系统必须满足的行为。

优先使用：
Given
When
Then

Spec 应该描述：
“系统应该怎样工作”，
而不是：
“代码应该怎么写”。

重要：

- 不修改业务代码。
- 不执行 tasks。
- 不自行改变已经确认的架构决策。
- 如果需求存在冲突，先报告。
- Spec 必须覆盖已经确认的关键需求。
- 检查 Proposal、Design、Spec 之间是否一致。

完成后检查：

1. 是否遗漏需求
2. 是否存在相互矛盾的 Spec
3. Design 是否能够支撑 Spec
4. 是否存在无法实现的要求
