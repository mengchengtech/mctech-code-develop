# 需求变更（requirement-change）

`preflight → 变更登记(主 Agent) → explorer(change-impact) → [人工确认影响] → (designer ↔ reviewer)* → human-confirm → planner(replan) → (executor ↔ verifier)* → close`

适用于：新业务规则、行为改变、Spec/Design 变更、API 语义变化、执行中范围扩大。**不是** verifier 判定的纯 `IMPLEMENTATION_BUG`（走 `implement-verify.md` `fix`）。

## 触发

- **直接使用**：用户明确变更需求；或对既有功能的扩展/优化/行为改变；或 explorer/verifier 路由的 `SCOPE_CHANGE` / `NEW_REQUIREMENT`。  
- **先问再用**：一句话里同时「修 BUG」和「改规则」→ 说明将走变更影响分析，询问是否确认按需求变更处理。  
- **不使用**：纯缺陷修复且 Spec 不变；全新大功能无既有 Change 基线 → 考虑 `full-flow` 新 Change。

## 前置

1. preflight；`brief.md` 标明 `task_type: requirement-change`。  
2. **变更登记**（主 Agent 写入 `brief.md` 与 `decisions-log.md`）：来源、时间、提出人/渠道（若知）、变更前后行为、原因、期望结果。  
3. 若有进行中 Change：读 `pipeline-state.yaml`、`tasks.md`、已有执行 ADR；**暂停**受影响 Task（`status: paused`），不覆盖历史 Task 描述。  
4. 关联受影响 OpenSpec / ADR 路径列表写入 `brief.md`。  
5. 验收标准更新为**变更后**行为；验证命令/环境同 `implement-verify.md`。

## 阶段

| 阶段 | 角色 | 模式 | 读 | 写 | 返回 → 下一步 |
|---|---|---|---|---|---|
| 登记 | 主 Agent | — | 用户输入、既有 Change | `brief.md`、`decisions-log.md` | → 影响分析 |
| 影响分析 | explorer | `change-impact` | 变更请求、OpenSpec、`tasks.md`、代码现状 | `.pipeline/analysis/ADR-变更影响分析.md` | `COMPLETED`/`NEED_INPUT` |
| 影响确认 | 用户 | — | 影响 ADR | `decisions-log.md`（批准范围） | 批准 → 更新规范 |
| 决策落稿 | designer | `decide-record`* | 已批方案、`decisions-log.md` | `.pipeline/decision/ADR-架构决策-rN.md` | *仅重大架构 |
| 更新规范 | designer | `change-update` | 影响 ADR、批准记录 | Change 根 OpenSpec | → 审核 |
| 审核 | reviewer | `change` | OpenSpec 增量、影响 ADR | `.pipeline/review/ADR-设计审核-rN.md` | `APPROVE` → 确认门 |
| 确认门 | 用户 | — | — | — | 指令表 |
| 重规划 | planner | `replan` | 新 OpenSpec、影响 ADR | `tasks.md`、规划 ADR | → 实施循环 |
| 实施+验 | executor, verifier | 见 `implement-verify.md` | | | `PASS` → close |

`*` 影响 ADR 含 `ARCHITECTURE_CHANGE` 且用户已选方案时执行；否则跳过。

各阶段另读 `decisions-log.md`。

### `change-impact` 阶段门禁

- explorer **不得**在本阶段写入 OpenSpec（与 `role-explorer.md` 一致）。  
- `Decision Required` 未关闭 → 不得进入 `change-update`。  
- 主 Agent 将用户确认要点追加 `decisions-log.md` 后再派 designer。

## 执行中变更（进行中 Change）

1. **暂停**：`pipeline-state.yaml` 标记受影响 `current_stage` / Task 为 `paused`。  
2. **评估已完成代码**：影响 ADR 必填「已完成 Task / 代码」节；保留或补偿 Task（X.1）由 planner `replan` 写明。  
3. **禁止**：直接在原 Task 文案上覆盖历史；删除已合并代码须用户授权。  
4. 规范更新并 `APPROVE` 后：executor 用 `resume` 或新 Task 批次，见 `implement-verify.md`。

## 循环与预算

- **影响分析**：最多 **2 轮** explorer（`change-impact` + 一次补查 `explore`）。  
- **规范+审核**：同 `design-review.md`（`REVISE` 最多 **3 轮**；`revise`/`change-update` 交替）。  
- **人工修订**：`.pipeline/review/人工意见-K.md`；不计入自动 3 轮。  
- **实施-验证**：同 `implement-verify.md`；`SCOPE_CHANGE` 再次触发时 **重新从影响分析** 或停等用户（不得 executor 私自改 Spec）。  
- 预算耗尽 → 停，汇报，不得放宽标准。

## 检查点（影响确认 & 确认门）

**影响确认**（`change-impact` 后）：展示分类、Proposal/Spec/Task/代码影响、补偿建议、Decision Required。**停等用户**批准范围。

| 用户指令 | 动作 |
|---|---|
| 批准变更范围 | → designer `change-update` |
| 缩小范围 | 更新 `brief.md` + 再分析或修订 ADR |
| 放弃变更 | `cancelled`；恢复暂停 Task（若安全） |
| 修订：&lt;意见&gt; | 人工意见 → designer |

**规范确认门**（审核 `APPROVE` 后）：同 `design-review.md` 检查点；默认 **必须**停（需求变更不做 `auto_confirm` 默认跳过）。

| 用户指令 | 动作 |
|---|---|
| 实施 / 继续 | → `replan` → 实施循环 |
| 修订 | 人工意见流程 |
| 新 Change | 冻结本 Change，开新 `change-id` 并记关联 |

## 并发

规范阶段无并发。实施阶段见 `implement-verify.md`。

## 与 BUG 流水线区分

| 情形 | 流水线 |
|---|---|
| 代码未做到现有 Spec | `bug-feedback` 或 `implement-verify` fix |
| 业务规则/SPEC 要变 | 本文件 |
| BUG 分诊发现规则问题 | 主 Agent 切换 `task_type` 并登记关联 |

## 复杂度

L1 小范围文案/单点行为：可合并影响分析与一次 `change-update`+单轮 reviewer；仍须 `replan` 与验证。  
L3 跨模块/数据迁移：完整阶段 + 双确认门（影响 + 规范）。

## 结束

OpenSpec 与 `tasks.md` 一致；`pipeline-state` 反映暂停项已关闭或例外记录。汇报：变更摘要、Task 增删、验证、回滚、关联 BUG/原 Change（≤15 行）。
