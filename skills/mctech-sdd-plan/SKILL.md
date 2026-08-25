---
name: mctech-sdd-plan
description: 根据 OpenSpec 将目标拆分为有依赖关系、可执行、可验证的 Tasks
stage: planning
mode: read-only
input:
  - proposal
  - design
  - specs
  - existing_tasks
  - repository
output:
  - tasks
  - task_dependencies
  - execution_plan
side_effects: openspec_only
authority: execution_plan
next:
  - mctech-sdd-implement
---

你现在处于实施计划阶段。

请根据当前 OpenSpec 的：

- proposal
- design
- specs

制定可执行的任务计划。

要求：

1. 按依赖关系拆分任务。
2. 每个任务有明确边界。
3. 每个任务尽量可以独立验证。
4. 不要简单按照目录拆任务。
5. 先完成基础能力，再进行业务迁移。
6. 数据库/API/缓存等基础变化应优先。
7. 标明任务依赖关系。
8. 标明可能冲突的任务。
9. 为关键任务安排测试。
10. 最后安排清理旧实现。

任务建议分类：

Foundation
Infrastructure
Migration
Business
Verification
Cleanup

每个 Task 必须包含：

- 目标
- 前置依赖
- 修改范围
- 需要遵守的 Spec
- 验收条件
- 测试要求

不要修改业务代码。

最后检查：
所有 Spec 是否都能通过 Tasks 得到实现。
