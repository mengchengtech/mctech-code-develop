# 只设计评审（design-review）

`preflight → explorer(可选) → (designer ↔ reviewer)* [→ designer recheck] → human-confirm → awaiting-implementation`

不修改业务代码。审核 `APPROVE` ≠ 用户批准。

## 触发

- **直接使用**：出设计/方案并评审；不要立即实施。  
- **先问再用**：要设计且多方案或公共接口 → 说明多 Agent 代价后询问。  
- **不使用**：要实施 → `full-flow`/`implement-verify`；单点 reviewer；极小改动。  

## 前置

1. preflight + `brief.md`（验收标准必填）。  
2. 已有外部设计 → 复制进 Change 根 OpenSpec/`design.md` 并记源路径；可从 reviewer 轮次起。  
3. explorer 可选；无则 designer 自读源码。

## 阶段

| 阶段 | 角色 | 模式 | 读 | 写 | 返回 → 下一步 |
|---|---|---|---|---|---|
| 设计 | designer | `spec`（或首轮无 OpenSpec 时探索后 `spec`） | `brief.md`、探索 ADR* | OpenSpec、`.pipeline/decision/ADR-设计决策-rN.md` | → 评审 |
| 评审 | reviewer | `design` | 上列 + 最新设计 ADR | `.pipeline/review/ADR-设计审核-rN.md` | `APPROVE`/`REVISE`/… |
| 复核 | designer | `recheck` | `brief.md`、最新审核 ADR | 设计 ADR 末尾复核节 | `CONFIRM`/`MINOR`/`ISSUE` |
| 确认门 | 用户 | — | — | — | 指令表 |

各阶段另读 `decisions-log.md`。

## 循环与预算

- **未解问题**（explorer/designer 返回 `NEED_INPUT` 或文内未解问题）：需用户答 → 一次提问写入 `decisions-log.md`；需补代码 → explorer `explore` 一次；仍不足 → 再问用户。  
- **审核循环**：`REVISE` → designer `revise`（读最新审核 ADR + `.pipeline/review/人工意见-K.md`）→ reviewer。自动循环最多 **3 轮**；审核 ADR 的 N 递增。连续两轮 designer 仍有多条不可接受项 → 停问用户。  
- **复核**：`brief.md` 开关 `design_recheck=on` 时，`APPROVE` 后 designer `recheck`；`ISSUE` 计入当前审核轮次。每份 `APPROVE` 设计最多复核一次。  
- **人工修订**：用户意见原文 → `.pipeline/review/人工意见-K.md`；要点 → `decisions-log.md`；designer `revise`；默认再 reviewer；用户说「修订后不用评审」则回确认门。**人工驱动不计入** 3 轮自动循环。  
- 预算耗尽 → 停，汇报路径，不得放宽标准。

## 检查点（确认门）

展示：OpenSpec/`design.md` 路径、摘要≤15 行、minor/未解问题、指令一行。**停等用户**。

| 用户指令 | 动作 |
|---|---|
| 实施 / 确认 / 继续 | 可接 `implement-verify` 或结束等待 |
| 修订：&lt;意见&gt; | 人工意见流程 → 回检查点 |
| 已手改 | 以现稿为准；要点 `decisions-log.md`；可跳过评审直实施（用户另要求除外） |
| 再评审 | reviewer 重审当前稿 |
| 复核 | 临时 `recheck` 一次 |
| 补探索：&lt;范围&gt; | explorer → designer `revise` → reviewer |
| 并发：N | 写 `brief.md` → 回检查点 |
| 结束 | 停；汇报状态 |

含糊回复 → 追问一次。

## 并发

探索可按区域（`orchestrator-protocol.md`）。设计/评审/复核：单实例。

## 路由（与主 Skill §8 一致）

`REJECT`/`NEED_INFO` → explorer；`REVISE` → designer；用户扩范围 → 需求变更或新 Change。
