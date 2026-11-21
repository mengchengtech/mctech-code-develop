# Git 阶段性提交（检查点）

避免单次 Change 积攒过多未提交变更；每个检查点应是**可回滚、可审查的阶段性成果**。默认仍**不自动 push**、只写本地工作区；`brief.md` 默认 `commit_policy: checkpoint`；`checkpoint` 默认即授权：主 Agent 自动按检查点提交、不逐次询问提示，用户明示禁止才停（如「不要 commit / 仅落盘」→ `commit_policy: off`）；`push`、发布、生产迁移仍须用户单独授权。

## 策略开关（`brief.md`；本表为 `commit_policy` 三态唯一载体）

| `commit_policy` | 含义 |
|---|---|
| `off` | 不 commit；仅落盘文件 |
| `checkpoint` | 主 Agent 自动按下表检查点提交、不逐次询问提示（**默认**）；用户明示禁止才停 |
| `end_only` | 仅 Change 正常结束或用户点名时一次提交 |

附带字段（主 Agent 维护）：

- `last_checkpoint_commit`：最近一次检查点 commit SHA（写入 `pipeline-state.yaml`）
- 每次 commit 后在 `run.log` 追加一行：`checkpoint Cn <sha> <说明>`

## 谁可以 commit

| 执行者 | 范围 |
|---|---|
| **主 Agent** | 全部检查点提交：Change 根 OpenSpec、`tasks.md`、`.pipeline/`（编排产物、验证报告、证据路径下的文档），以及业务源码、测试与对应执行记录所列改动 |
| **其它子 Agent** | 一律不 commit（含 executor；源码与产物改动由主 Agent 在检查点统一提交） |

禁止：`commit --amend` 覆盖他人/上一检查点（除非用户明确且仅整理上一检查点笔误）；禁止 `--no-verify` 除非用户明确；禁止 `push`（单独授权）。

## 提交消息格式

```text
<type>(<change-id>): <阶段> — <一句话成果>
```

| type | 用途 |
|---|---|
| `docs` | OpenSpec、ADR、`.pipeline/`、`tasks.md` |
| `feat` | 实现 Task / 集成步骤 |
| `fix` | verifier 指引的 fix 轮次 |
| `test` | 仅测试补充且与 feat 分开展示时（少见） |

示例：`docs(20251107-auth): exploration ADR — module map`；`feat(20251107-auth): task 2.3 — API handler`。

## 检查点一览

**原则**：子 Agent 单次派发 **返回终态**（如 `COMPLETED`、`APPROVE`、`PASS`）且产物验收通过后，在**进入下一派发或下一检查点门禁之前**提交。`NEED_INPUT` / `REVISE` / `FAIL` / 进行中的 `integrate` **不提交**（除非用户要求 WIP 分支）。**保存义务**（照常执行，提交时机按本表）：每个阶段结束后保存文档（OpenSpec 与 `.pipeline/` 各阶段 ADR）；每个执行批次结束后保存代码、`.pipeline/execution/` 执行记录与测试结果。

### A. 编排与规范（主 Agent）

| ID | 时机 | 纳入变更（典型） |
|---|---|---|
| **C0** | `preflight` 完成 | `.pipeline/brief.md`、`baseline.md`、`pipeline-state.yaml`、`decisions-log.md`、`run.log`（初始） |
| **C1** | explorer 单次派发 `COMPLETED` | 当次 exploration / analysis / bug / change-impact ADR |
| **C2** | designer 单次派发 `COMPLETED`（`recheck` 为 `CONFIRM`/`MINOR` 亦可） | 当轮 OpenSpec 补丁 + `decision/` ADR + `design.md` 修订 |
| **C3** | reviewer 结论 `APPROVE` | 当轮 `review/` ADR + 已通过稿 OpenSpec（若本轮有改） |
| **C4** | 人工确认门用户「实施/继续/批准」后 | `decisions-log.md` 确认要点 + 当前有效 OpenSpec / state |
| **C5** | planner / `replan` `COMPLETED` | `tasks.md` + `planning/` ADR |
| **C6** | 需求变更「影响确认」批准后 | 影响 ADR + 更新后的 `brief.md` / state |
| **C7** | verifier `PASS`（`task` 或 `change`） | 当轮 `verification/` ADR + `evidence/` 引用 + 本轮相关 `.pipeline` 更新；**不含**尚未验收的源码（源码见 C9） |

### B. 实施（主 Agent）

| ID | 时机 | 执行者 | 纳入变更（典型） |
|---|---|---|---|
| **C8** | executor 单 Task `COMPLETED` | 主 Agent | 该 Task 范围内源码/测试 + **`tasks.md` 本 Task 标题与已满足验收项勾选** + 对应 `执行记录` 中列出的文件 |
| **C9** | 同上 Task 对应 verifier `PASS` | 主 Agent | C8 之上可 `--amend` **仅当用户明确且 C8 尚未 push**；否则新增 `feat: task X verified`；并含验证 ADR |
| **C10** | 并行组全部 `COMPLETED` 且 `integrate` `COMPLETED` | 主 Agent | 整组+集成改动 + 各 `执行记录-<组>.md`、集成记录 |
| **C11** | `fix` 派发 `COMPLETED` 且同轮 verifier `PASS` | 主 Agent | fix 改动 + 执行记录 fix 节 |

推荐顺序：**C8（Task 做完）→ 跑 verifier → C9 或 C7**；避免大量 Task 堆叠后一次 commit。

### C. 里程碑与其它合适节点

| ID | 时机 | 说明 |
|---|---|---|
| **C12** | 设计审核 `APPROVE` 后、**进入 planner 之前** | 规范冻结点；便于实施段与规范段 bisect |
| **C13** | 高风险 Task 开始前（迁移/删数/权限变更） | **保存点**：当前 HEAD 记入 `baseline.md`「执行前保存点」与 state |
| **C14** | BUG 分诊结论为「实现缺陷」且将进入修复 | 分诊 ADR + `brief` 级别裁剪结论 |
| **C15** | Change 终态 `completed` / `completed_with_exceptions` | 收尾 `pipeline-state`、未合并的文档补丁；可与最后一次 C7/C11 合并为「close」提交 |
| **C16** | 用户「已手改」检查点 | 仓库中用户改动 + `decisions-log` + 同步后的 OpenSpec |

### 不建议单独 commit

- 单次并发的**中间实例**（等整波 + integrate）
- 审核 `REVISE` / 探索 `NEED_INPUT` 的半成品 ADR
- verifier `FAIL` 待 fix 的破损实现（除非 `commit_policy` 下用户要 WIP 分支名）
- 混入 Change 外无关文件

## 与回滚 / 恢复

- 每个检查点提交后：更新 `pipeline-state.yaml` 的 `last_checkpoint_commit`、`artifacts` 清单。
- **流程回退**是重新探索/分析/设计/审核/规划，不自动撤销代码；**代码回滚**是撤销某批代码/配置/迁移，必须明确目标基线和数据影响。流程回退（重跑 designer 等）**不自动** `reset` 历史 commit；回滚代码优先 `git revert` 到 `last_checkpoint_commit`、明确 SHA 或针对性反向补丁；不得未经授权执行 `git reset --hard`、强制清理、生产回滚或数据删除；数据库和外部副作用必须有备份、补偿或人工处置方案。
- **保存与基线记录**：前置准备记录 commit、分支和工作区差异（`baseline.md`）；每个文档阶段在 `.pipeline/` 保存 ADR、在 Change 根保存 OpenSpec，并更新 `pipeline-state.yaml`；每个执行批次保存代码差异、`.pipeline/execution/` 执行记录和测试结果。
- 恢复会话：读 `brief.md`、`pipeline-state.yaml`、`run.log` 末行与产物首行 verdict（仲裁细则见 `references/orchestrator-protocol.md`「恢复」）；对比 `last_checkpoint_commit` 与工作区；脏树且超出当前 Task 范围 → 停问用户。

## 脏工作区

- preflight：已有未提交改动 → 问用户：纳入本 Change / stash（用户做）/ 单独分支；记入 `baseline.md`。
- executor 派发前：仅允许改 Task 白名单内文件；否则先由主 Agent 按检查点提交或还原后再派。
