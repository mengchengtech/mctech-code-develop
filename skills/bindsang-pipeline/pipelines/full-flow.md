# 全流程开发（full-flow）

`preflight → explorer → (designer ↔ reviewer)* → human-confirm → planner → (executor ↔ verifier)* → close`

设计审核细节见 `design-review.md`；实施验证见 `implement-verify.md`（阶段、预算、检查点、并发在其内生效，本文只写衔接与覆盖）。

## 触发

- **直接使用**：新功能/大重构/跨模块；用户要求完整流水线或多 Agent 协作开发。  
- **先问再用**：≥2 项：粗估 ≥5 文件或 ≥2 模块；不熟悉代码；多方案或公共接口；验收不明。一句话说明代价后询问。  
- **不使用**：单模块小改、已定位 bug、样式文案、依赖升级、纯问答 → 不启流水线。  
- 只要设计段 → `design-review`；只要实施段 → `implement-verify`。

## 前置

合并一次用户提问。

1. preflight（`SKILL.md` §4）：`brief.md`、`baseline.md`、`pipeline-state.yaml`、`decisions-log.md`、`run.log`（空）。  
2. `brief.md` 补：可验证验收标准、验证命令（AGENTS.md/package.json）、验证环境、开关默认、角色映射。  
3. 角色解析（`orchestrator-protocol.md`）。  
4. 验收标准缺失 → 问清再继续。

## 阶段

| 阶段 | 角色 | 模式 | 读 | 写 | 返回 → 下一步 |
|---|---|---|---|---|---|
| 探索 | explorer | `explore`（可按区域并发） | `brief.md` | `.pipeline/exploration/ADR-探索记录*.md` | `COMPLETED`/`NEED_INPUT` → 分析或设计前暂停 |
| 分析 | explorer | `analyze` | 探索 ADR、`brief.md` | `.pipeline/analysis/ADR-需求与影响分析.md` | → 设计循环 |
| 设计+审 | designer, reviewer | 见 `design-review.md` | OpenSpec 草案输入 | OpenSpec + 决策/审核 ADR | `APPROVE` → 确认门 |
| 确认门 | 用户 | — | 设计+审核 ADR、OpenSpec | — | 用户指令 |
| 规划 | planner | `plan` | 已批 OpenSpec、决策 ADR | `tasks.md`、规划 ADR | → 实施循环 |
| 实施+验 | executor, verifier | 见 `implement-verify.md` | `tasks.md`、OpenSpec | 代码 + 执行/验证 ADR | `PASS` → close |

探索 `NEED_INPUT`：合并问用户或派 `explore` 补查（见 `design-review`「未解问题」规则）。

## 覆盖项

1. **回设计计数**：自实施阶段回到 designer 的次数 = 记账键 `rollbacks j/上限`（与审核循环分开；回设计后 `revisions` k 清零重计；默认上限与耗尽三项选择见 `references/orchestrator-protocol.md`「记账」）。  
2. **实施受阻**：executor `BLOCKED(拆分过粗|设计遗漏|设计事实有误)` → 按 `implement-verify.md` 回设计；通过后 executor **`resume`**，不经确认门。`需确认` → 主 Agent 询问。  
3. **确认门**：等同 `design-review` 检查点 + `implement-verify` 前置中需告知项（并发、验证环境）。

## 开关（记入 `brief.md`）

开关默认值、效果与生效时机的唯一载体 = `references/orchestrator-protocol.md`「任务记录」开关表（`auto_confirm` / `pause_after_design` / `design_recheck` / `concurrency` / `commit_policy` / `piece_acceptance` / `role_models`；切换开关不重置计数）；本流水线不另设开关。`commit_policy` 三态与检查点提交授权语义见 `references/git-checkpoints.md`。

## 结束

同 `implement-verify.md`「结束」；另记审核轮次、回设计次数。
