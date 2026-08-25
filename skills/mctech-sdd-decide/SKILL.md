---
name: mctech-sdd-decide
description: 将架构师确认的方案和约束正式记录为 Architecture Decision
stage: decision
mode: read-only
input:
  - user_request
  - analysis
  - exploration
  - architect_decisions
output:
  - decisions
  - constraints
side_effects: none
authority: architecture_decision
next:
  - mctech-sdd-spec
---

你现在处于架构决策确认阶段。

当前任务：
{{TASK}}

以下内容是架构师已经确认的决策。
请将以下内容视为已经确定的架构约束，不要自行修改：

{{DECISIONS}}

请将这些决策整理成正式的架构决策记录。

要求：

1. 明确最终采用的方案。
2. 明确不采用的方案。
3. 记录关键设计约束。
4. 记录兼容性要求。
5. 记录迁移要求。
6. 记录回滚要求。
7. 记录重要的非功能性要求。
8. 检查决策之间是否存在冲突。

重要：

- 不要自行改变架构师的决策。
- 如果发现冲突，停止并指出冲突。
- 不修改代码。
- 不执行任务。
- 不擅自增加架构约束。

请基于这些已经确定的架构决策，重新整理最终技术方案。
如果发现这些决策之间存在矛盾，不要自行修改，请列出冲突供我确认。

输出：

1. Decision
2. Architecture Constraints
3. Compatibility Constraints
4. Migration Constraints
5. Rollback Constraints
6. Conflicts / Questions
