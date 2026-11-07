# 产物规范：ADR + OpenSpec

## 目录约定

| 位置 | 内容 |
|---|---|
| `<change-root>/` | OpenSpec：`proposal.md`、`design.md`、`tasks.md`、`specs/<capability>/spec.md` |
| `<change-root>/.pipeline/` | 状态与简报、`decisions-log`、自定义流水线定义、各阶段 ADR、执行/验证记录 |

`<change-root>` 默认为 `changes/<change-id>/`；若项目使用 `openspec/changes/<change-id>/`，以其为准。

`.pipeline/` 内按阶段分子目录：`exploration/`、`analysis/`、`decision/`、`review/`、`planning/`、`execution/`、`verification/`、`bug/`。只创建本 Change 实际需要的目录；文档内容中文，目录名英文小写。

ADR 至少包含：状态、Change、阶段、版本、日期、背景、目标/非目标、事实证据、分析/方案/审查内容、决策结论、影响权衡、风险缓解、验收标准、回滚方式、未决问题、关联文档。ADR 路径均在 `.pipeline/<阶段>/` 下。

OpenSpec 映射：`proposal.md` 保存背景动机和范围；`specs/<capability>/spec.md` 保存需求与验收场景；`design.md` 保存设计（宜含改动清单、测试计划、**实施步骤**与并行组/集成步骤，供 executor 并发）；`tasks.md` 保存实施计划。ADR 保存决策依据、审查、人工确认和验证证据。两者互相引用，不重复堆叠全文。每轮设计/审核/执行/验证产生新版本，旧 ADR 标记已替代，不删除。

## 编排记录（`.pipeline/` 根下）

- `brief.md`：目标、约束、验收标准、验证命令、验证环境、开关、角色映射（对齐原 dev-pipeline `00-task.md` 职能）。
- `decisions-log.md`：仅主 Agent 追加的已确认决策。
- `run.log`：派发审计一行一条。
- `.pipeline/review/人工意见-K.md`：用户检查点修订原文（首行可 `VERDICT: REVISE`）。
- `.pipeline/verification/evidence/`：verifier 截图、trace、日志（可选）。

## Verdict 约定

评审/验证 ADR **首行**写结论，便于主 Agent「读返回」：`APPROVE`/`REVISE`/…、`PASS`/`FAIL`/…；designer `recheck` 返回 `CONFIRM`/`MINOR`/`ISSUE`；executor 返回 `COMPLETED` 或 `BLOCKED(<类型>)`。
