# bindsang-pipeline 使用指南

面向使用者的实操说明：何时启用、如何启动主 Skill、五条标准流水线、单点/自定义组合、恢复续跑，以及检查点指令。编排细节以 [`SKILL.md`](SKILL.md)、[`references/orchestrator-protocol.md`](references/orchestrator-protocol.md) 与各 [`pipelines/*.md`](pipelines/) 为准。

---

## 1. 前置：让 Cursor 认得本 Skill

| 方式 | 做法 |
|---|---|
| **项目 Skill 路径** | 确保仓库存在 `skills/bindsang-pipeline/SKILL.md`（本仓库已具备）。在 Cursor 设置中将该目录（或整个 `skills/`）加入 **Project Rules / Skills** 扫描路径（与团队 `AGENTS.md` 约定一致）。 |
| **对话中显式加载** | 在新会话第一条消息中 `@` 引用 [`SKILL.md`](SKILL.md)，或说明「按 bindsang-pipeline 编排」。 |
| **子 Agent（可选）** | 在支持自定义 Subagent 的环境中，按 [`roles.md`](roles.md)「Subagent 注册」表注册六个 agent：`role-explorer`、`role-designer`、`role-reviewer`、`role-planner`、`role-executor`、`role-verifier`（源文件在 `agents/role-*.md`）。未注册时，主 Agent 可用 **generalPurpose + 必读角色文件路径** 模拟，协议不变。 |
| **无多 Agent 环境** | 主 Agent 按 `agents/role-*.md` 与 `pipelines/<id>.md` **分阶段顺序执行**，仍须落盘 `.pipeline/` 与 OpenSpec，遵守相同门禁。 |

---

## 2. 一分钟决策：用不用流水线

### 2.1 应该用（命中任一条 → 考虑 bindsang）

| 场景 | 建议 `task_type` |
|---|---|
| 新功能、大重构、跨模块、公共接口/契约变更 | `full-flow` |
| 只要方案 / 设计 / 架构评审，暂不写业务代码 | `design-review` |
| 已有获批 OpenSpec / 设计，要按 Task 实施并验收 | `implement-verify` |
| 需求规则、行为、范围相对**已批准基线**发生变化 | `requirement-change` |
| 用户 / 业务 / 监控报告的 BUG、崩溃、数据不一致 | `bug-feedback` |
| 继续未完成的 Change | 读 `<change-root>/.pipeline/pipeline-state.yaml` 续跑 |
| 点名 `explorer` / `designer` / … 或描述角色顺序 | 单点派发或 §6 自定义组合 |
| 关键词：流水线、子智能体、多 Agent、bindsang | 由主 Agent 按 §3 分类 |

### 2.2 不应该用（直接让主 Agent 完成，不建 Change）

- 单模块小改、一两处边界清晰修改  
- **已定位根因** 的缺陷且未要求流水线  
- 样式、文案、无行为变化的配置微调  
- 仅依赖升级（无配套改造）  
- 代码解释、走读、一次性代码评审（非独立 reviewer 委派）  
- 概念咨询、与仓库无关问答  
- 用户明确：「不要流水线 / 不用子 Agent / 你直接改」

### 2.3 轻量路径（L1）

用户要小改但未禁止流水线时：主 Agent 可合并探索+分析、简化审核，**仍须** `brief.md`、基线、必要验证与 `.pipeline/` 记录。

### 2.4 「先问再用」

当流水线标注「先问再用」（跨多文件、不熟代码、多方案、验收不明等）时，主 Agent 应用**一句话**说明多 Agent 代价后**询问**；用户同意再 `preflight`。

---

## 3. 五条标准流水线与路径

| `task_type` | 典型路径（缩写） | 详细定义 |
|---|---|---|
| `full-flow` | preflight → 探索/分析 → 设计↔审核 → **确认门** → 规划 → 实施↔验证 → 结束 | [`pipelines/full-flow.md`](pipelines/full-flow.md) |
| `design-review` | preflight → 探索* → 设计↔审核* → 复核* → **确认门** → 等待实施 | [`pipelines/design-review.md`](pipelines/design-review.md) |
| `implement-verify` | preflight → 规划* → 实施↔验证* → 结束 | [`pipelines/implement-verify.md`](pipelines/implement-verify.md) |
| `requirement-change` | 登记 → 变更影响 → **影响确认** → 设计↔审核 → **确认门** → replan → 实施↔验证 | [`pipelines/requirement-change.md`](pipelines/requirement-change.md) |
| `bug-feedback` | 分诊 → 路由 → 修复方案* → 审核* → 确认门* → 规划 → 实施↔验证 | [`pipelines/bug-feedback.md`](pipelines/bug-feedback.md) |

`*` = 可按复杂度/级别跳过或缩短；`**确认门**` = 停等用户指令。

**产物位置**

- Change 根：`changes/<change-id>/` 或项目既定的 `openspec/changes/<change-id>/`
- OpenSpec：`proposal.md`、`design.md`、`tasks.md`、`specs/…`
- 编排记录：`<change-root>/.pipeline/`（见 [`references/artifacts.md`](references/artifacts.md)）

---

## 4. 启动方式总览

### 4.1 Skill 触发（推荐给编排者）

在 Cursor 对话中使用下列任一方式，主 Agent 应加载 `bindsang-pipeline` 并按协议执行：

| 方式 | 示例 |
|---|---|
| **@ 主 Skill** | `@skills/bindsang-pipeline/SKILL.md` 然后描述任务 |
| **@ 具体流水线** | `@skills/bindsang-pipeline/pipelines/full-flow.md` 实现 xxx |
| **自然语言 + 类型** | 「用 bindsang 全流程做：…」「走 design-review，先不要写代码」 |
| **显式 task_type** | 「`task_type: implement-verify`，Change 在 `openspec/changes/foo/`，继续实施」 |
| **点名角色** | 「只派 explorer，模式 explore，调研模块 A」 |
| **禁止编排** | 「不要流水线，直接改」→ 主 Agent 不得建 Change、不得派子 Agent |

**主 Skill 描述中的自动触发词**（见 `SKILL.md` frontmatter）：流水线、子智能体、多 Agent、bindsang；以及各场景关键词（新功能、设计评审、需求变更、外部 BUG 等）。

### 4.2 用户 Prompt 模板（复制改写）

#### A. 全流程新功能

```text
@skills/bindsang-pipeline/SKILL.md
按 full-flow 编排以下需求：
- 目标：<一句话>
- 非目标：<可选>
- 验收标准：<可验证条目>
- 验证命令：<如 npm test / 手工步骤>
Change 根目录：changes/<YYYYMMDD-短名>/
复杂度：L2（或让主 Agent 评估）
不要自动 commit/push。
```

#### B. 只设计 + 审核

```text
@skills/bindsang-pipeline/pipelines/design-review.md
出 OpenSpec 与设计 ADR，不要改业务源码。
目标：…
验收标准：…
设计后暂停确认（pause_after_design / 默认确认门）。
```

#### C. 已有 Spec，实施 + 验证

```text
@skills/bindsang-pipeline/pipelines/implement-verify.md
Change 根：openspec/changes/<id>/
已有获批 design.md 与 tasks.md，请规划（若缺）并执行+验证。
验收标准：见 brief 或 <粘贴>
验证环境：<如本地 / 测试账号变量名>
```

#### D. 需求变更

```text
@skills/bindsang-pipeline/pipelines/requirement-change.md
在 Change `<id>` 上登记需求变更：
- 变更前行为：…
- 变更后行为：…
- 原因：…
先走 change-impact，影响确认后再更新 OpenSpec。
```

#### E. 外部 BUG

```text
@skills/bindsang-pipeline/pipelines/bug-feedback.md
BUG 现象：…
复现步骤：…
环境/版本：…
严重级别：L2（或让主 Agent 分诊）
关联 Change（如有）：<related_change_id>
```

#### F. 恢复未完成 Change

```text
@skills/bindsang-pipeline/SKILL.md
继续 Change：changes/<id>/
先读 .pipeline/pipeline-state.yaml、brief.md、run.log 末行与各产物首行 verdict，
一句话确认当前阶段后继续，不要覆盖已有 decisions-log。
```

#### G. 单点角色（不调完整流水线）

```text
@skills/bindsang-pipeline/references/orchestrator-protocol.md
单点派发（仍需最小 preflight 或只读任务）：
- Change 根：changes/<id>/
- 角色：verifier，模式：task
- 输入：.pipeline/brief.md、tasks.md、最近执行 ADR
- 输出：.pipeline/verification/ADR-验证报告-rN.md
```

#### H. 自定义角色顺序（临时流水线）

```text
@skills/bindsang-pipeline/SKILL.md
自定义顺序：explorer → designer → reviewer → designer → reviewer
目标：…（不写业务代码）
请写入 .pipeline/custom-pipeline.md，task_type: custom；
默认不自动 REVISE 循环，每轮结束汇报等我指令。
```

### 4.3 主 Agent 派发子 Agent 的 Prompt 块（编排者参考）

每次委派**只传路径与约束**，不复述全文（见 orchestrator-protocol「派发块」）：

```text
Change 根：<绝对或仓库相对路径>
角色：designer
模式：spec
必读：.pipeline/brief.md、.pipeline/decisions-log.md
输入：.pipeline/exploration/ADR-探索记录.md（如有）
输出：proposal.md、design.md、specs/…、.pipeline/decision/ADR-设计决策-r1.md
轮次：1
附加约束：（可选，只收紧，如「不得扩展 scope」）
请遵守 skills/bindsang-pipeline/agents/role-designer.md 全文。
```

并发实施时增加：**步骤编号、文件白名单**、输出 `执行记录-<组>.md`（见 `implement-verify.md`）。

### 4.4 在 Cursor 中注册子 Agent（可选）

将 `agents/role-*.md` 注册为 Subagent 时，frontmatter 已含：

- `name`: 如 `role-explorer`
- `pipeline-role`: `explorer` | `designer` | …
- `tools`: 须与 `roles.md` 权限类一致（executor 需可改源码；reviewer 只读）

主 Agent 解析顺序：`AGENTS.md` 指定 → `pipeline-role` → 已注册 subagent → generalPurpose + 角色文件路径。

---

## 5. 按场景：完整使用说明

### 5.1 新功能 / 大重构（full-flow）

**触发**：直接使用 — 新功能、大重构、跨模块、用户要完整流水线。  
**先问再用**：≥5 文件或 ≥2 模块、不熟代码、多方案、验收不明。

**你要提供**：目标、验收标准、验证命令/环境、是否允许并发（默认 3）。

**`brief.md` 开关**（用户可说出口令，主 Agent 写入 brief）：

| 开关 | 默认 | 用户说 | 效果 |
|---|---|---|---|
| `auto_confirm` | off | 自动 / 不用确认 | 跳过设计确认门（full-flow） |
| `pause_after_design` | off | 设计后暂停 | 强制确认门 |
| `design_recheck` | off | 设计者复核 | APPROVE 后 designer `recheck` |
| `concurrency` | 3 | 并发 N | 探索/实施并发上限 |

**设计确认门指令**（与 design-review 相同，见下表 §5.2）。

**预算（自动循环）**：设计审核 REVISE 最多 3 轮；从实施回设计最多 2 次；实施-验证 FAIL 修复最多 3 轮。

**示例 Prompt**：见 §4.2 A。

---

### 5.2 只设计评审（design-review）

**触发**：出设计/方案并评审；不要立即实施。  
**不适用**：要实施 → `full-flow` / `implement-verify`；极小改动。

**路径**：`preflight` → 可选 explorer → `designer.spec` ↔ `reviewer.design` → 可选 `designer.recheck` → **确认门** → `awaiting-implementation`。

**确认门 — 用户指令表**（主 Agent 停等）：

| 用户说 | 动作 |
|---|---|
| 实施 / 确认 / 继续 | 可接 `implement-verify` 或结束等待 |
| 修订：&lt;意见&gt; | 写入人工意见 → designer `revise` → 默认再 reviewer |
| 已手改 | 以现稿为准；要点进 decisions-log；可跳过评审直实施 |
| 再评审 | reviewer 重审当前稿 |
| 复核 | designer `recheck` 一次 |
| 补探索：&lt;范围&gt; | explorer → designer `revise` → reviewer |
| 并发：N | 写 brief → 回检查点 |
| 结束 | 停；汇报状态 |

**示例 Prompt**：见 §4.2 B。

---

### 5.3 实施与验证（implement-verify）

**触发**：已确认设计/OpenSpec，要实施+验证。  
**前置硬条件**：获批 OpenSpec + `tasks.md`（或等价）；独立运行须 reviewer `APPROVE` 或用户确认记入 `decisions-log.md`。

**路径**：`preflight` → `planner.plan`（可跳过若已有 tasks）→ `executor` ↔ `verifier`。

**executor 模式**：`implement` | `concurrent` | `integrate` | `resume` | `fix`  
**verifier 模式**：`task`（单批）| `change`（整 Change）

**特殊返回**：

- verifier `SCOPE_CHANGE` / SPEC 类 → 回 explorer/designer/planner，**不是** executor 私自改 Spec  
- verifier 环境阻塞 → 问用户：解除 / **环境豁免**（记 decisions-log）/ 终止  
- executor `BLOCKED(拆分过粗|设计遗漏|设计事实有误)` → 独立运行停并建议 design-review；full-flow 内自动回设计

**示例 Prompt**：见 §4.2 C。

---

### 5.4 需求变更（requirement-change）

**触发**：用户明确改规则/行为；或 verifier `SCOPE_CHANGE` / `NEW_REQUIREMENT`。

**双确认门**：

1. **影响确认**（change-impact 后）：批准范围 / 缩小 / 放弃 / 修订  
2. **规范确认门**（审核 APPROVE 后）：同 design-review；**默认不得**用 `auto_confirm` 跳过

**执行中 Change**：暂停受影响 Task（`paused`），不覆盖历史 Task 文案；`planner.replan` 写补偿 Task。

**示例 Prompt**：见 §4.2 D。

---

### 5.5 外部 BUG（bug-feedback）

**与 implement-verify 内 FAIL 区分**：verifier 在本 Change 内发现的问题 → `executor.fix` 循环，**不**新建 bug-feedback。

**级别裁剪**（主 Agent 记入 brief / state）：

| 级别 | 设计+审核 | 确认门 | 规划 |
|---|---|---|---|
| L1 | 可跳过 | 可跳过（auto_confirm） | 轻量单 Task |
| L2 | spec + 1 轮 reviewer | 建议停 | 标准 plan |
| L3 | 完整 design-review 段 | **必须**停 | plan + 里程碑 |

**分诊后路由**：实现缺陷 → 修复链；Spec/规则错 → `requirement-change` 或新 Change；无法复现 → 停等补充。

**确认门指令**：修复/继续、修订、不是 BUG 是需求变更、仅记录不修、结束。

**示例 Prompt**：见 §4.2 E。

---

### 5.6 恢复 / 继续 Change

**触发词**：继续、恢复、`change-id`、接着做。

**主 Agent 必读**：`pipeline-state.yaml`、`brief.md`、`run.log` 末行、相关 ADR **首行 verdict**；核对 Git 与工作区。

**示例 Prompt**：见 §4.2 F。

---

### 5.7 单点派发

**适用**：只探索一块代码、只跑一次 reviewer、只验证当前批次等。

**协议**：仍建议有 `brief.md` 最小目标；派发表记入 `run.log`。

**示例 Prompt**：见 §4.2 G。

---

### 5.8 自定义角色组合（task_type: custom）

**适用**：用户指定顺序且**不匹配**五条标准流水线，例如 `explorer → designer` 或 `planner → executor → verifier`。

**落盘**：`.pipeline/custom-pipeline.md`；`pipeline-state.yaml` 标记 `task_type: custom` 与角色序列。

**默认**：临时组合**不自动** REVISE/FAIL 循环；每步结束汇报，等用户再派。

**示例 Prompt**：见 §4.2 H。

---

## 6. 角色与模式速查

派发时必须 **角色 + 模式**（见 [`roles.md`](roles.md)）：

| 角色 | 常用模式 |
|---|---|
| explorer | `explore` / `analyze` / `bug-triage` / `change-impact` |
| designer | `decide-record` / `spec` / `change-update` / `revise` / `recheck` |
| reviewer | `design` / `decision` / `change` |
| planner | `plan` / `replan` |
| executor | `implement` / `fix` / `concurrent` / `integrate` / `resume` |
| verifier | `task` / `change` |

**返回码（主 Agent 读首行）**：reviewer `APPROVE|REVISE|REJECT|NEED_INFO`；verifier `PASS|FAIL|BLOCKED|SCOPE_CHANGE`；explorer/planner `COMPLETED|NEED_INPUT|BLOCKED|FAILED`；executor `COMPLETED|BLOCKED(<类型>)|FAILED`。

---

## 7. 人工反馈后的自动路由（主 Agent）

用户在一次确认中可能给出多条意见，主 Agent 按最早受影响阶段回退（见 `SKILL.md` §8）：

- 事实/调研不足 → explorer  
- 目标/影响分析错 → explorer `analyze`  
- 方案/接口/模型错 → designer  
- 设计大体对、补审查 → reviewer  
- Task 拆分错 → planner  
- 实质扩范围/改规则 → **新需求变更** 或新 Change（不得伪装成普通修订）

---

## 8. 常用用户口令（检查点与开关）

下列口令可在**任意检查点**或会话中说，主 Agent 写入 `brief.md` / `decisions-log.md` 后生效：

| 口令 | 含义 |
|---|---|
| 并发：N | `brief.concurrency = N` |
| 自动 / 不用确认 | `auto_confirm = on`（full-flow / L1 bug；**需求变更规范门默认仍须停**） |
| 设计后暂停 | `pause_after_design = on` |
| 设计者复核 | `design_recheck = on` |
| 修订：… | 人工意见文件 + designer `revise` |
| 已手改 | 以仓库现稿为准，要点进 decisions-log |
| 不是 BUG，是需求变更 | 切换 `requirement-change` |
| 环境豁免 | verifier 环境阻塞时记入 decisions-log，标注 UNVERIFIED 项 |
| 不要 commit / 可以 commit | 默认不 commit；仅用户授权后 executor 可 commit |

---

## 9. 结束时期待输出

主 Agent 应给出（≤15 行 + 路径）：

- Change 根、`pipeline-state` 终态  
- 关键 verdict、改动文件、验证证据  
- `UNVERIFIED` / 需人工项、遗留风险、回滚点  
- 下一步建议（如接 `implement-verify`）

---

## 10. 文档索引

| 文档 | 用途 |
|---|---|
| [`SKILL.md`](SKILL.md) | 主 Agent 契约、分类、裁剪、门禁 |
| [`USAGE.md`](USAGE.md) | 本使用指南 |
| [`roles.md`](roles.md) | 角色、模式、Subagent 注册 |
| [`pipelines/`](pipelines/) | 五条流水线阶段表与预算 |
| [`references/orchestrator-protocol.md`](references/orchestrator-protocol.md) | 派发、并发、恢复、选用 |
| [`references/artifacts.md`](references/artifacts.md) | 目录与 ADR/OpenSpec 分工 |
| [`authoring.md`](authoring.md) | 扩展新流水线/角色 |

仓库总览入口：根目录 [`README.md`](../../README.md)「Agent 流水线」一节。
