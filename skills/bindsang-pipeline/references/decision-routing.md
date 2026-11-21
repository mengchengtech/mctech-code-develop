# 人工反馈分类与自动路由（唯一路由载体）

用户反馈（确认门意见、修改请求、新输入）先拆成原子事项并逐项分类路由；选取**最早受影响阶段**，重新执行其后的依赖阶段，其后依赖产物标记过期但不删除；主 Agent 必须说明路由依据。改变已批准需求基线 → 登记需求变更（`pipelines/requirement-change.md`），不得把实质需求变更伪装成普通修订。

## 反馈分类路由

| 反馈类型 | 回退位置 |
|---|---|
| 代码现状、依赖关系事实错误 | explorer 探索 |
| 需求语义、业务规则、影响面错误 | explorer 分析 |
| 接口、数据模型、架构方案错误 | designer |
| 审核遗漏或证据不足 | reviewer |
| 任务拆分、顺序、依赖错误 | planner |
| 实现偏离设计或代码缺陷 | executor |
| 验收定义改变 | explorer 分析，必要时 designer |
| 新目标或范围扩大 | 新 Change / requirement-change |

## 修改请求（流水线期间，用户未指明执行者）

自 Change 建立至「结束汇报」为止，用户提出设计 / 源码 / 验证类修改请求且未指明执行者 → 按下表路由派发完成，一行告知，不追加确认回合：

| 请求 | 路由 |
|---|---|
| 改设计 | 检查点「修订」流程；不在检查点时，意见原文按 `- [K-n]` 写入 `.pipeline/review/人工意见-K.md` 后单点派 designer `revise`（见 `pipelines/design-review.md`「人工修订」） |
| 改源码 | 意见原文按 `- [K-n]` 写入人工意见文件后派 executor `fix`；被中断的产物先按 `references/orchestrator-protocol.md` 硬约束 7「接续」完成，再修复本请求；逐片任务传片编号与文件白名单 |
| 补查事实 / 定位 | explorer |
| 重新验证 | verifier |

此类派发 `run.log` 标 `人工修订`，不计循环额度（见 `references/orchestrator-protocol.md`「记账」；逐片验收停点的人工修复除外）；要点记入 `decisions-log.md`。

## 例外（从窄解释，拿不准即派发）

| # | 情形 | 处理 |
|---|---|---|
| ① | 用户明确要求主 Agent 亲自做 | 主 Agent 直接做 |
| ② | 只涉及主 Agent 职责文件（`brief.md`、`pipeline-state.yaml`、`run.log`、`decisions-log.md`、人工意见文件、派发回退复制的角色文件副本） | 主 Agent 直接做 |
| ③ | 与本 Change 产物链无关 | 不派发也不代做，提示用户另行处理 |
| ④ | 单文件、纯文案 / 注释 / 格式重排，不影响行为、不改测试文件与配置 | 可由主 Agent 直接做，做后一行告知并记入 `decisions-log.md` |
| ⑤ | 环境 / 依赖类请求（装依赖、改构建 / 包管理脚本） | 不派发，按「必须停下询问用户」处理（硬约束 3 禁止角色安装依赖） |

## 优先级

涉及验收标准变更、破坏性操作、任一记账上限耗尽 → 仍按 `references/orchestrator-protocol.md`「必须停下询问用户」，本文件路由不改变这些停点。
