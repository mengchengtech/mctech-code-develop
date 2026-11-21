# bindsang-pipeline 使用指南

面向使用者的实操说明：何时启用、如何启动主 Skill、五条标准流水线、单点/自定义组合、恢复续跑，以及检查点口令。编排细节以 [`SKILL.md`](SKILL.md)、[`references/orchestrator-protocol.md`](references/orchestrator-protocol.md) 与各 [`pipelines/*.md`](pipelines/) 为准。

---

## 1. 前置：环境加载与子 Agent 注册

| 方式 | 做法 |
|---|---|
| **项目 Skill 路径** | 确保仓库存在 `skills/bindsang-pipeline/SKILL.md`（本仓库已具备）；按所用环境的 Skill / Rules 扫描约定纳入该目录（或整个 `skills/`），与团队 `AGENTS.md` 约定一致。 |
| **对话中显式加载** | 在新会话第一条消息中 `@` 引用 [`SKILL.md`](SKILL.md)，或说明「按 bindsang-pipeline 编排」。 |
| **子 Agent** | 六个角色已注册于环境子代理注册（本仓库为 `~/.zcode/agents/role-*.md`），与 [`roles.md`](roles.md) 表一致。主 Agent 派发时引用 `name`（如 `role-explorer`）及 `pipeline-role`；**默认不在派发参数中指定 `model`**（模型在子 Agent 注册文件的 `model` 中配置）。仅用户显式要求某角色用某模型时，写入 `brief.role_models` 并在当次派发覆盖。完整契约见 `skills/bindsang-pipeline/agents/role-*.md`；注册缺失时按 `roles.md`「环境子代理注册」回退分级兜底。 |
| **无多 Agent 环境** | 主 Agent 按 `agents/role-*.md` 与 `pipelines/<id>.md` **分阶段顺序执行**，仍须落盘 `.pipeline/` 与 OpenSpec，遵守相同门禁。 |

---

## 2. 一分钟决策：用不用流水线

- 应该用 / 不应该用（场景 → `task_type` 触发表、不建 Change 的不使用清单、点名角色与关键词分类）：唯一载体 = [`SKILL.md`](SKILL.md) §2/§3 触发表与总判据。
- 轻量路径 L1：小改但未禁止流水线时，可合并探索与分析、设计审核快速检查、规划简化为任务清单；基线 / 风险 / 验证 / 记录门禁不减，详见 `SKILL.md` §5。
- 「先问再用」（跨多文件、不熟代码、多方案取舍、验收不明等）：主 Agent 用一句话说明多 Agent 代价后询问，用户同意再 preflight（各流水线条件见 `pipelines/*.md`「触发」）。
- 继续 / 恢复未完成 Change：读 `<change-root>/.pipeline/pipeline-state.yaml` 续跑（示例见 §4.2 F）；单点派发与自定义组合机制见 `SKILL.md` §3.1。

---

## 3. 五条标准流水线与路径

| `task_type` | 典型路径（缩写） | 详细定义 |
|---|---|---|
| `full-flow` | preflight → 探索/分析 → 设计↔审核 → **确认门** → 规划 → 实施↔验证 → 结束 | [`pipelines/full-flow.md`](pipelines/full-flow.md) |
| `design-review` | preflight → 探索＊ → 设计↔审核＊ → 复核＊ → **确认门** → 等待实施 | [`pipelines/design-review.md`](pipelines/design-review.md) |
| `implement-verify` | preflight → 规划＊ → 实施↔验证＊ → 结束 | [`pipelines/implement-verify.md`](pipelines/implement-verify.md) |
| `requirement-change` | 登记 → 变更影响 → **影响确认** → 设计↔审核 → **确认门** → replan → 实施↔验证 | [`pipelines/requirement-change.md`](pipelines/requirement-change.md) |
| `bug-feedback` | 分诊 → 路由 → 修复方案＊ → 审核＊ → 确认门＊ → 规划 → 实施↔验证 | [`pipelines/bug-feedback.md`](pipelines/bug-feedback.md) |

`＊` = 可按复杂度/级别跳过或缩短；`**确认门**` = 停等用户指令。

**产物位置**

- Change 根：`changes/<change-id>/` 或项目既定的 `openspec/changes/<change-id>/`
- OpenSpec：`proposal.md`、`design.md`、`tasks.md`、`specs/…`
- 编排记录：`<change-root>/.pipeline/`（见 [`references/artifacts.md`](references/artifacts.md)）

---

## 4. 启动方式总览

### 4.1 Skill 触发（推荐给编排者）

在对话中使用下列任一方式，主 Agent 应加载 `bindsang-pipeline` 并按协议执行：

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

- 场景：只出 OpenSpec 与设计 ADR、不改业务源码，设计后停在确认门（`pause_after_design` 或默认确认门）；与 A 的差异 = 换用 `pipelines/design-review.md`、无实施段。按 §3 路径表改写 A/C 使用。

#### C. 已有 Spec，实施 + 验证

```text
@skills/bindsang-pipeline/pipelines/implement-verify.md
Change 根：openspec/changes/<id>/
已有获批 design.md 与 tasks.md，请规划（若缺）并执行+验证。
验收标准：见 brief 或 <粘贴>
验证环境：<如本地 / 测试账号变量名>
```

#### D. 需求变更

- 场景：在既有 Change 上登记需求变更（变更前 / 后行为、原因），先走 `change-impact` 影响分析再更新 OpenSpec；与 A/C 的差异 = 含**双确认门**（影响确认 + 规范确认门，规范门默认不得 `auto_confirm` 跳过，见 §5.4）。按 §3 路径表改写 A/C 使用。

#### E. 外部 BUG

- 场景：报告 BUG 现象 / 复现步骤 / 环境版本，由主 Agent 分诊定级别（L1–L3 级别裁剪，见 §5.5）；与 A 的差异 = 走 `pipelines/bug-feedback.md` 分诊 → 路由 → 修复链，Spec / 规则错时升级为需求变更。按 §3 路径表改写 A/C 使用。

#### F. 恢复未完成 Change

```text
@skills/bindsang-pipeline/SKILL.md
继续 Change：changes/<id>/
先读 .pipeline/pipeline-state.yaml、brief.md、run.log 末行与各产物首行 verdict，
一句话确认当前阶段后继续，不要覆盖已有 decisions-log。
```

#### G. 单点角色（不调完整流水线）

- 场景：只派一个角色跑一次工作（如 explorer 调研模块、verifier `task` 验证当前批次）；与 A/C 的差异 = 不进流水线循环，仍需最小 preflight，派发表记入 `run.log`（见 §5.7）。按 §3 路径表改写 A/C 使用。

#### H. 自定义角色顺序（临时流水线）

- 场景：用户指定角色顺序且**不匹配**五条标准流水线，写 `.pipeline/custom-pipeline.md` 并标 `task_type: custom`；与 A 的差异 = 默认**不自动** REVISE / FAIL 循环，每步结束汇报等用户指令（机制见 `SKILL.md` §3.1）。按 §3 路径表改写 A/C 使用。

### 4.3 主 Agent 派发子 Agent 的 Prompt 块（编排者参考）

每次委派**只传路径与约束**，不复述产物全文；派发块首行固定句与字面模板唯一载体 = `references/orchestrator-protocol.md`「派发块」（Change 根、角色 / 模式、输入 / 输出、轮次、并发白名单、追加约束只收紧）。并发实施时增加：**步骤编号、文件白名单**、输出 `执行记录-<组>.md`（见 `pipelines/implement-verify.md`）。

### 4.4 使用环境子代理

子 Agent 注册于环境子代理注册（本仓库为 `~/.zcode/agents/role-*.md`；`name` + `pipeline-role` + 委派用 `description`，见 [`roles.md`](roles.md)）。测试示例：

```text
使用 role-explorer 子 Agent：Change 根 changes/demo/，模式 explore，
输入 .pipeline/brief.md，输出 .pipeline/exploration/ADR-探索记录.md
```

主 Agent 解析顺序：`AGENTS.md` 指定 → `pipeline-role` / `name` → 环境子代理注册 → 宿主通用 agent（general-purpose）+ `agents/role-*.md` 路径兜底（回退分级与全部落兜底时停问见 `roles.md`「环境子代理注册」）。

**模型**：调整各角色默认模型 → 改环境子代理注册文件（本仓库为 `~/.zcode/agents/role-*.md`）的 `model`，而非在派发 prompt 里写模型名。临时按角色换模型（用户口头指定）→ 主 Agent 记入 `brief.role_models` 后仅对该角色传入 `model`。

同步更新契约时：先改 `skills/bindsang-pipeline/agents/role-*.md`，再对齐环境子代理注册副本的摘要、`description` 与 `model`（`references/orchestrator-protocol.md`「副本校验」）。

---

## 5. 按场景：使用说明

### 5.1 新功能 / 大重构（full-flow）

- 触发：新功能、大重构、跨模块；「先问再用」条件见 `pipelines/full-flow.md`「触发」。要提供：目标、验收标准、验证命令 / 环境、是否允许并发（默认 3）。
- 开关与口令：开关定义（默认值 / 生效时机 / 切换不重置计数）唯一载体 = `references/orchestrator-protocol.md`「任务记录」开关表；口令 → 开关映射见 §8。
- 确认门与预算：设计确认门同 §5.2；自动循环预算唯一载体 = `references/orchestrator-protocol.md`「记账」（`revisions` / `rollbacks` / `fixes` 键与耗尽三项选择）。
- 示例 Prompt：见 §4.2 A。

### 5.2 只设计评审（design-review）

- 触发：出设计 / 方案并评审、不立即实施；要实施 → `full-flow` / `implement-verify`，极小改动不适用（见 `pipelines/design-review.md`「触发」）。
- 路径：`preflight` → 可选 explorer → `designer.spec` ↔ `reviewer.design` → 可选 `designer.recheck` → **确认门** → `awaiting-implementation`。
- 确认门 = 停等用户的裁决点：八条指令（实施 / 修订 / 已手改 / 再评审 / 复核 / 补探索 / 并发 / 结束）+ 含糊回复追问一次，指令表唯一完整输出 = `pipelines/design-review.md`「检查点（确认门）」。
- 示例 Prompt：见 §4.2 B。

### 5.3 实施与验证（implement-verify）

- 触发与前置硬条件：已确认设计 / OpenSpec 要实施 + 验证；须有获批 OpenSpec + `tasks.md`；独立运行须 reviewer `APPROVE` 或用户确认记入 `decisions-log.md`（见 `pipelines/implement-verify.md`「前置」）。
- 路径：`preflight` → `planner.plan`（已有 tasks 可跳过）→ `executor` ↔ `verifier`。executor 模式：`implement` / `concurrent` / `integrate` / `resume` / `fix`；verifier 模式：`task`（单批）/ `change`（整 Change）。
- **特殊返回**：verifier `SCOPE_CHANGE` / SPEC 类 → 回 explorer / designer / planner，**不是** executor 私自改 Spec；verifier 环境阻塞 → 问用户：解除 / **环境豁免**（记 `decisions-log.md`）/ 终止；executor `BLOCKED(拆分过粗|设计遗漏|设计事实有误)` → 独立运行停并建议 design-review，`full-flow` 内自动回设计。
- 逐片验收（开关 `piece_acceptance=on`）：片验证 `PASS` 后停等 继续 / 修复 / 终止，见 `pipelines/implement-verify.md`「逐片验收」。
- 示例 Prompt：见 §4.2 C。

### 5.4 需求变更（requirement-change）

- 触发：用户明确改规则 / 行为；或 verifier `SCOPE_CHANGE` / `NEW_REQUIREMENT`。
- **双确认门**：① **影响确认**（change-impact 后）：批准范围 / 缩小 / 放弃 / 修订；② **规范确认门**（审核 APPROVE 后）：同 design-review；**默认不得**用 `auto_confirm` 跳过。
- 执行中 Change：暂停受影响 Task（`paused`），不覆盖历史 Task 文案；`planner.replan` 写补偿 Task。
- 示例 Prompt：见 §4.2 D。

### 5.5 外部 BUG（bug-feedback）

- 与 implement-verify 内 FAIL 区分：verifier 在本 Change 内发现的问题 → `executor.fix` 循环，**不**新建 bug-feedback（区分唯一全文 = `SKILL.md` §11）。
- **级别裁剪**（主 Agent 记入 brief / state）：

| 级别 | 设计+审核 | 确认门 | 规划 |
|---|---|---|---|
| L1 | 可跳过 | 可跳过（auto_confirm） | 轻量单 Task |
| L2 | spec + 1 轮 reviewer | 建议停 | 标准 plan |
| L3 | 完整 design-review 段 | **必须**停 | plan + 里程碑 |

- 分诊后路由（无法复现 / 实现缺陷 / Spec·规则错 / 环境因素）见 `pipelines/bug-feedback.md`「分诊后路由」；确认门指令（修复 / 修订 / 不是 BUG 是需求变更 / 仅记录不修 / 结束）见 `pipelines/bug-feedback.md`「检查点（确认门）」。
- 示例 Prompt：见 §4.2 E。

### 5.6 恢复 / 继续 Change

- 触发词：继续、恢复、`change-id`、接着做。
- 主 Agent 必读：`pipeline-state.yaml`、`brief.md`、`run.log` 末行、相关 ADR **首行 verdict**；核对 Git 与工作区（一致性仲裁见 `references/orchestrator-protocol.md`「恢复」）。
- 示例 Prompt：见 §4.2 F。

### 5.7 单点派发

- 适用：只探索一块代码、只跑一次 reviewer、只验证当前批次等。
- 协议：仍建议有 `brief.md` 最小目标（至少含「目标」）；派发表记入 `run.log`（细则见 `references/orchestrator-protocol.md`「选用」）。
- 示例 Prompt：见 §4.2 G。

### 5.8 自定义角色组合（task_type: custom）

- 适用：用户指定顺序且**不匹配**五条标准流水线（机制唯一载体 = `SKILL.md` §3.1）。
- 落盘：`.pipeline/custom-pipeline.md`；`pipeline-state.yaml` 标记 `task_type: custom` 与角色序列。
- 默认：临时组合**不自动** REVISE / FAIL 循环；每步结束汇报，等用户再派。
- 示例 Prompt：见 §4.2 H。

---

## 6. 角色与模式速查

派发必须**角色 + 模式**；模式词表与返回码词表唯一来源 = [`roles.md`](roles.md)「角色与模式」表（契约与流水线不得增删模式名），回退分级与独立性等级见同文件「环境子代理注册」。

---

## 7. 人工反馈后的自动路由（主 Agent）

用户反馈与修改请求按**最早受影响阶段**回退；唯一路由载体 = [`references/decision-routing.md`](references/decision-routing.md)（反馈分类路由 + 修改请求路由 + 例外表 + 优先级），主 Agent 必须说明路由依据。

---

## 8. 常用用户口令（检查点与开关）

下列口令可在**任意检查点**或会话中说，主 Agent 写入 `brief.md` / `decisions-log.md` 后生效；开关定义唯一载体 = `references/orchestrator-protocol.md`「任务记录」开关表，本表只保留口令 → 开关映射：

| 口令 | 写入 / 动作 |
|---|---|
| 并发：N | `concurrency = N` |
| 自动 / 不用确认 | `auto_confirm = on`（需求变更规范确认门默认仍须停） |
| 设计后暂停 | `pause_after_design = on` |
| 设计者复核 | `design_recheck = on` |
| 修订：… | 人工意见文件 + designer `revise`（见 §5.2） |
| 已手改 | 以仓库现稿为准，要点进 `decisions-log.md` |
| 不是 BUG，是需求变更 | 切换 `requirement-change` |
| 环境豁免 | verifier 环境阻塞时记入 `decisions-log.md`，标注 UNVERIFIED 项 |
| 可以 commit | `commit_policy = checkpoint`（默认已自动按检查点提交、不逐次提示，本口令为显式再确认；三态见 `references/git-checkpoints.md`） |
| 不要 commit / 仅落盘 | `commit_policy = off` |
| 分步提交 / 每步 commit | 与默认 `checkpoint` 相同（可显式写入 brief） |

---

## 9. 结束时期待输出

结束汇报模板唯一载体 = `references/orchestrator-protocol.md`「结束汇报」（≤20 行；必含产物路径与 verdict、改动摘要、`UNVERIFIED` / 需人工项、遗留问题、回滚点、对账 d / 预估六项；`pipelines/` 可要求补充项）。

---

## 10. 文档索引

| 文档 | 用途 |
|---|---|
| [`SKILL.md`](SKILL.md) | 主 Agent 契约、分类与总判据、加载图、复杂度裁剪、门禁 |
| [`USAGE.md`](USAGE.md) | 本使用指南 |
| [`roles.md`](roles.md) | 角色、模式词表唯一来源、环境子代理注册 |
| [`pipelines/`](pipelines/) | 五条流水线阶段表与预算 |
| [`references/orchestrator-protocol.md`](references/orchestrator-protocol.md) | 派发块、硬约束、启动窗口、记账、并发、恢复、结束汇报、维护 |
| [`references/artifacts.md`](references/artifacts.md) | 目录与 ADR / OpenSpec 分工 |
| [`references/git-checkpoints.md`](references/git-checkpoints.md) | `commit_policy` 三态、检查点 C0–C16 时机与消息格式（唯一载体） |
| [`references/decision-routing.md`](references/decision-routing.md) | 反馈分类路由与修改请求路由（唯一载体） |

检查点提交：默认 `brief.md` 含 `commit_policy: checkpoint`——主 Agent 自动按检查点提交、不逐次询问提示，用户明示禁止才停（「不要 commit / 仅落盘」→ `commit_policy: off`）。检查点消息格式与 C0–C16 时机唯一载体 = [`references/git-checkpoints.md`](references/git-checkpoints.md)；不宜单独 commit 的半成品 / 并发中间态亦见该文件。

仓库总览入口：根目录 [`README.md`](../../README.md)「Agent 流水线」一节。
