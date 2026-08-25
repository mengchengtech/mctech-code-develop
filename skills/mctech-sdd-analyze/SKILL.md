---
name: mctech-sdd-analyze
description: 开启探索模式，分析思考发挥想象积极探索方案，探察问题，澄清需求。还可以基于当前系统事实分析问题、比较方案并提出推荐方案
stage: analysis
mode: read-only
input:
  - user_request
  - exploration
  - repository
output:
  - analysis
  - options
  - recommendation
  - open_questions
side_effects: none
authority: proposal
next:
  - mctech-sdd-decide
  - mctech-sdd-spec
---

你现在处于方案分析阶段。
本阶段仍然禁止修改代码。

当前任务：
{{TASK}}

请基于已有的探索结果和当前代码，对问题进行深入分析。

要求：

1. 明确当前问题的根因。
2. 分析问题涉及的模块和边界。
3. 提出 2～3 个可行方案。
4. 比较各方案的：
   - 正确性
   - 安全性
   - 性能
   - 复杂度
   - 改造成本
   - 兼容性
   - 可维护性
5. 分析对现有系统的影响。
6. 分析数据库、API、缓存、前端和测试影响。
7. 分析迁移和回滚方案。
8. 给出推荐方案及推荐理由。

重要：

- 不修改代码。
- 不修改 OpenSpec。
- 不把推荐方案视为已经确定的架构决策。
- 无法确认的问题标记为“需要决策”。

输出：

1. 问题分析
2. 方案 A
3. 方案 B
4. 方案 C（如有必要）
5. 方案对比
6. 推荐方案
7. 风险
8. 需要人工决策的问题
