# 主 Agent 调度协议（编排层）

本文件定义**调度者**行为；Change 生命周期、任务分类见 `SKILL.md`；阶段链见 `pipelines/`；角色契约见 `agents/role-*.md`。流程细节不重复写入本文件以外的主 Skill 正文时，以本文件为准。

## 选用（与主 Skill §3 配合）

1. 用户指名 `pipelines/<id>.md` → 严格按其执行。  
2. 用户点名单角色或只需一次工作 → **单点**（仍走 preflight 最小集或只读任务可跳过，由主 Agent 判断）。  
3. 用户点名多角色或描述顺序且无对应流水线 → **临时组合**；**默认不自动循环**，`REVISE`/`FAIL`/`BLOCKED` 等 → 汇报并建议，等用户后再派发。  
4. 未点名但命中流水线「触发」→ 读该流水线「直接使用 / 先问再用」。  
5. 其余 → 不启流水线，直接完成请求。

单点：确保 `<change-root>/.pipeline/brief.md` 至少含「目标」；派发、读返回、汇报。  
临时组合要自动循环 → 写 `custom-pipeline.md` 或选用标准流水线。

## 任务记录（`.pipeline/`）

| 文件 | 写者 | 用途 |
|---|---|---|
| `brief.md` | 主 Agent | 目标、约束、**可验证验收标准**、验证命令、验证环境、基线、开关（并发数等）、角色映射、`task_type` |
| `decisions-log.md` | 仅主 Agent | 已确认决策，**仅追加**一行：`- [阶段/轮次] 决定 — 原因`；取代旧决定时新行注明「取代 …」 |
| `run.log` | 仅主 Agent | 派发审计：`<序号> <流水线:阶段\|单点\|临时> <角色>/<模式> → <输出> \| <返回>` |
| `pipeline-state.yaml` | 仅主 Agent | 机器可读状态（见 `state-machine.md`） |

`decisions-log.md` **仅**在以下时机追加：用户新约束或裁决；检查点修订/手改要点；用户对未解问题/角色选择的答复；executor `BLOCKED` 回设计时记明的类型与事实。  
不记录过程、推测、子 Agent 意见。

开关写入 `brief.md`；用户说「并发 N」「自动/设计后暂停」等 → 更新 `brief.md`，下一检查点生效。

## 派发块

每次委派 prompt **只传**（不复述产物全文、不带调度者偏好）：

- Change 根路径  
- 角色；**模式**（见 `roles.md`）  
- 输入文件（含 `brief.md`、`decisions-log.md`）  
- 输出文件  
- 轮次；并发时：步骤编号、文件白名单  
- 追加约束（可选，**只收紧**角色规则）

子 Agent 须先读 `decisions-log.md`：视为约束，不得推翻；冲突时在产物中指出，不写该文件。

## 硬约束

1. **独立性**：`roles.md`「独立于」角色之间只传路径，不传被独立方推理或自检结论。  
2. **写入**：角色只写己方产物；仅 executor 可改业务源码。并发实例不写同一文件；调度者不合并产物。  
3. **git / 依赖**：角色不得 `commit/push/stash/checkout/reset/clean`（任务明确 authorize commit 时仅 executor 可 commit）。不得安装依赖。  
4. **不嵌套**：子 Agent 不得再派子 Agent。  
5. **读返回**：以角色「返回」为准；产物有首行 verdict 时**先看首行**。仅当：`FAIL`（失败项/环境阻塞）、`BLOCKED`、`NEED_INPUT`、未解问题数>0、reviewer `REVISE`/`NEED_INFO`、检查点需要摘要时，读产物对应节。  
6. **失败**：返回不合规 → 重派一次并指出问题；仍失败 → 汇报用户。限流/过载：并发数减半（最低 1）记入 `brief.md`，重派并告知。  
7. **提问合并**：对用户的阻塞问题合并为一次提问。

## 并发

- 同一条消息内多个子 Agent 调用 = 并发；分多条 = 串行。整批返回后再下一步。  
- 每批 ≤ `brief.md` 并发数（默认 3；用户指定优先）。超出则分波。  
- 条件：实例互不依赖、输出文件互异。  
- 只写产物类：调度者划分不重叠范围（目录/模块）。  
- 改源码类（全部满足才并发）：各有**文件白名单**且两两不交；不依赖对方产出；不含共享/生成文件（barrel、路由注册、i18n 汇总、lockfile、全局类型）→ 这些内容放在并行组之后的 executor **`integrate`** 模式。  
- explorer 可按区域并行（输出 `ADR-探索记录-<区域>.md`）。  
- 整批后处理：任一 `BLOCKED` → 不派后续波。

具体阶段能否并发由 `pipelines/*.md` 规定。

## 角色解析（工具环境支持时）

1. 用户/AGENTS.md 指定 agent  
2. `description` 或 frontmatter 含 `pipeline-role: <角色名>`（见 `roles.md`「Subagent 注册」）  
3. `roles.md` 登记之 `agents/role-*.md` 已注册为 subagent（`name` 字段与表内一致）  
4. 兜底：generalPurpose + 必读 `<skill>/agents/role-*.md` 绝对路径  

副本校验：若环境存在同名 agent 定义，与本 skill `agents/` 源文件 diff——仅 frontmatter 差异可忽略；正文不一致则停，提示用户用源文件更新注册副本。

- 「改源码」类 agent 须有 Edit；「只写产物」不得 Edit。  
- 互为「独立于」的角色不得同一 agent 兼任。  
- 派发时附协议补丁：遵守角色文件的规则、输出、返回、当前模式。

## 必须停下询问用户

任何开关下均停：轮次预算耗尽；验收标准需改；破坏性操作；人工意见标「不可行」；`BLOCKED(需确认)`；verifier 环境阻塞（见 `pipelines/implement-verify.md`）。各流水线检查点另有补充。

## 恢复

会话中断后继续：读 `brief.md`、`pipeline-state.yaml`、`run.log` 末行、相关产物首行 verdict，推断阶段，向用户一句话确认后继续。无 `run.log` 时按产物与 state 推断。

## 结束汇报

≤15 行：`change-root`、关键 verdict、`UNVERIFIED`/需人工项、遗留问题、产物路径；`pipelines/` 可要求补充项。
