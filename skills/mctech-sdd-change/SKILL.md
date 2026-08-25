---
name: mctech-sdd-change
description: 分析实施中的需求变化，更新 OpenSpec 并重新规划剩余任务
stage: change_management
mode: openspec-write
input:
  - change_request
  - current_change
  - task_status
  - repository
  - existing_openspec
output:
  - impact_analysis
  - spec_changes
  - design_changes
  - task_changes
  - migration_plan
side_effects:
  - openspec
authority: latest_approved_spec
next:
  - mctech-sdd-plan
  - mctech-sdd-implement
---

当前 OpenSpec Change 已经处于实施阶段。

现在出现新的需求变化：

{{CHANGE_REQUEST}}

当前任务状态：

{{TASK_STATUS}}

不要立即修改代码。

请进入需求变更分析流程。

### Phase 1：Impact Analysis

分析：

1. Proposal 影响
2. Design 影响
3. Spec 影响
4. 已完成 Task 影响
5. 当前 Task 影响
6. 未执行 Task 影响
7. API 影响
8. 数据库影响
9. 前端影响
10. 缓存影响
11. 测试影响

### Phase 2：Change Classification

将影响分类为：

- 不受影响
- 实现修正
- Spec 修改
- Design 修改
- 已完成 Task 需要补偿
- 当前 Task 需要修改
- 未执行 Task 需要修改
- 新增 Task
- 删除 Task
- 需要架构师决策

### Phase 3：Update

只有需求变化确认后，才：

1. 更新 Proposal
2. 更新 Design
3. 更新 Spec
4. 更新 Tasks
5. 更新 Task 依赖关系

重要：

- 不要因为需求变化就重新执行所有 Task。
- 不要因为 Task 已完成就拒绝需求变化。
- 不要自行改变架构决策。
- 不要直接修改代码。
- 保留必要的变更原因和历史。

最终输出：

1. Change Impact
2. Spec Changes
3. Design Changes
4. Task Changes
5. New Tasks
6. Obsolete Tasks
7. Need Decision
