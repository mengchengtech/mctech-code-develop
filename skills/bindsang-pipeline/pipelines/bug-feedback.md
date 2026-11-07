# 外部 BUG 反馈（bug-feedback）

`preflight → 分诊(explorer) → [修复方案(designer) ↔ 审核(reviewer)*] → human-confirm(按风险) → planner → (executor ↔ verifier)* → close`

**内部验证缺陷**（当前 Change 的 verifier `FAIL`）不走本流水线，走 `implement-verify.md` 的 `fix` 循环。  
**需求/规则变化**（非实现错误）→ 转 `requirement-change.md` 或新 Change，并在 `brief.md`/`decisions-log.md` 记录与 BUG Change 的关联。

## 触发

- **直接使用**：用户/业务/监控报告 BUG、崩溃、数据不一致、性能退化、异常结果。  
- **先问再用**：现象模糊且可能跨系统；或同时夹杂「新需求」描述 → 说明需先分诊再选流水线，询问是否走 BUG 流水线。  
- **不使用**：已知根因的局部代码修复且用户不要流水线；纯咨询；应归类为 `requirement-change` 或 `full-flow` 的新能力。

## 前置

合并一次用户提问（无法复现时必须问清环境/步骤/账号/版本）。

1. preflight；`brief.md` 标明 `task_type: bug-feedback`。  
2. 记录 **BUG 来源**、报告时间、影响范围、严重级别（建议：S1–S4 或项目约定）、是否生产。  
3. 验收标准 = **修复后预期行为**（可验证）；缺则询问。  
4. 若关联既有 OpenSpec Change：在 `brief.md` 记 `related_change_id`；无则修复范围仅限 BUG，不扩需求。  
5. 验证命令 / 环境写入 `brief.md`（同 `implement-verify.md` 前置）。  
6. `run.log`、`decisions-log.md` 就绪。

## 严重级别与路径裁剪（主 Agent，记入 `brief.md` 与 `pipeline-state.yaml`）

| 级别 | 典型情形 | 设计+审核 | 确认门 | 规划 |
|---|---|---|---|---|
| **L1** | 单点逻辑错误、明确堆栈、无数据/安全/兼容 | 可跳过；explorer 分诊 ADR 即足 | 可跳过（`auto_confirm`） | 轻量单 Task |
| **L2** | 单模块、需小范围设计或回归说明 |  designer `spec`（修复向）+ reviewer 1 轮 | 建议停 | 标准 `plan` |
| **L3** | 数据、安全、跨服务、兼容、生产回填 | 完整 `design-review` 段 + reviewer | **必须**停 | `plan` + 里程碑 |

裁剪不得省略：分诊证据、基线、验证、记录。L1 仍须 verifier，不得未验证宣称修复。

## 阶段

| 阶段 | 角色 | 模式 | 读 | 写 | 返回 → 下一步 |
|---|---|---|---|---|---|
| 分诊 | explorer | `bug-triage`；必要时 `explore` | `brief.md`、用户输入 | `.pipeline/bug/ADR-BUG分诊与复现.md` | `COMPLETED`/`NEED_INPUT` |
| 定性 | 主 Agent | — | 分诊 ADR | `decisions-log.md`（分类结论） | 路由（见下） |
| 修复方案 | designer | `spec`（窄范围）或跳过 L1 | 分诊 ADR、`brief.md` | OpenSpec 补丁或 `.pipeline/decision/ADR-设计决策-rN.md` | → 审核或规划 |
| 审核 | reviewer | `design` | 上列 | `.pipeline/review/ADR-设计审核-rN.md` | L2/L3：`APPROVE` → 确认门 |
| 确认门 | 用户 | — | — | — | 指令（L3 必须；L2 建议） |
| 规划 | planner | `plan` | 已批方案、`tasks.md` 补丁 | `tasks.md`、规划 ADR | → 实施循环 |
| 实施+验 | executor, verifier | 见 `implement-verify.md` | | | `PASS` → close |

各阶段另读 `decisions-log.md`。

### 分诊后路由（主 Agent）

| 结论 | 动作 |
|---|---|
| **无法复现** | `NEED_INPUT` → 停等补充；不猜测修复 |
| **实现缺陷**（符合 Spec/既有规则） | 进入修复方案/规划/实施（按级别裁剪） |
| **Spec/规则错误** 或 **新需求** | 转 `requirement-change.md` 或新 Change；本 Change 可 `awaiting_human` 或 `cancelled` |
| **环境/数据外部因素** | 记 `decisions-log.md`；停等运维/用户；非 executor 擅自改需求 |

explorer 在分诊 ADR 末尾给 **建议路由**；仅主 Agent 可切换流水线类型。

## 循环与预算

- **未解问题 / 无法复现**：同 `design-review.md`「未解问题」；分诊阶段最多 **2 轮** explorer（`bug-triage` + 一次 `explore` 补证）。  
- **修复方案审核**：L2/L3 适用 `design-review.md` 审核循环（最多 **3 轮** `REVISE`）。L1 无审核轮次。  
- **实施-验证**：同 `implement-verify.md`（`fix` 优先；最多 **3 轮**）。  
- verifier 分类为 `NEW_REQUIREMENT`/`SPEC_DEFECT` → **停止 fix**，转 `requirement-change` 或用户裁决。

## 检查点（确认门）

L3 **必须**；L2 默认启用；L1 在 `auto_confirm=on` 时可跳过。

展示：分诊摘要、根因、修复范围、风险、回滚、验证计划。

| 用户指令 | 动作 |
|---|---|
| 修复 / 继续 | → planner 或 `implement-verify` 段 |
| 修订：&lt;意见&gt; | `.pipeline/review/人工意见-K.md` → designer `revise` |
| 不是 BUG，是需求变更 | → `requirement-change` |
| 仅记录，不修 | 结束；`pipeline-state` → `awaiting_human` |
| 结束 | 汇报路径与未决项 |

高危害（生产数据、权限）未经确认不得 `executor`。

## 并发

默认串行。仅当修复方案含并行组时，遵循 `implement-verify.md` 与 `orchestrator-protocol.md`。

## 与全流程衔接

- 从 `full-flow` **中断**转入：保留原 Change 的 OpenSpec 关联，新建 BUG Change 或在本 Change 标记 `bug_feedback` 阶段。  
- 修复合并后：可选在同一 Change 终验 `verifier` `change` 模式。

## 结束

汇报：根因、改动文件、验证结论、是否转需求变更、遗留问题（≤15 行）。`pipeline-state.yaml` → `completed` / `completed_with_exceptions` / `awaiting_human`。
