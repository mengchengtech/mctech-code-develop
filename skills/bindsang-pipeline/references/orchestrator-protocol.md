# 主 Agent 调度协议（编排层）

本文件定义**调度者**行为；Change 生命周期、任务分类见 `SKILL.md`；阶段链见 `pipelines/`；角色契约见 `agents/role-*.md`。流程细节不重复写入本文件以外的主 Skill 正文时，以本文件为准。

## 选用（触发判定唯一载体 = SKILL §3）

任务分类与触发表见 `SKILL.md` §3（唯一载体）；本节只保留单点与临时组合细则。

- **单点**：用户点名单角色或只需一次工作。仍走 preflight 最小集或只读任务可跳过，由主 Agent 判断；确保 `<change-root>/.pipeline/brief.md` 至少含「目标」；派发、读返回、汇报。
- **临时组合**：用户点名多角色或描述顺序且无对应流水线。要自动循环 → 写 `<change-root>/.pipeline/custom-pipeline.md`（见 `SKILL.md` §3.1）或选用标准流水线；**默认不自动循环**，`REVISE`/`FAIL`/`BLOCKED` 等 → 汇报并建议，等用户决定后再派发。

## 任务记录（`.pipeline/`）

| 文件 | 写者 | 用途 |
|---|---|---|
| `brief.md` | 主 Agent | **字段清单唯一载体**：目标、背景、范围、非目标、约束、**可验证验收标准**、验证命令、验证环境、基线、开关（见下方开关表）、角色映射、`role_models`（仅用户显式指定时）、`task_type`、用户原始输入；新字段 `estimate`（预估派发批次）、`piece_acceptance`（默认 off）、`independence`（三槽位独立性等级：异构 / 同模型独立 / 宿主默认） |
| `decisions-log.md` | 仅主 Agent | 已确认决策，**仅追加**一行：`- [阶段/轮次] 决定 — 原因`；取代旧决定时新行注明「取代 …」 |
| `run.log` | 仅主 Agent | 派发台账，一行一实例：派发时先记 `<批次序号>[a/b/c] <流水线:阶段\|单点\|临时> <角色>/<模式> → <输出> \| dispatched`，返回后补记 `\| <返回>`。批次序号即 d：串行每批 1 个；同批并发的多实例共用同一序号、各加后缀 a/b/c |
| `pipeline-state.yaml` | 仅主 Agent | 机器可读状态。字段：`schema_version`、`change_id`、`change_root`、`task_type`、`roles`（自定义时）、`complexity`、`status`、`current_stage`、`stage_round`、Git 基线（`baseline_commit`）、`last_checkpoint_commit`、审批标志、产物清单（路径含 `.pipeline/` 前缀）、`next_action`、更新时间；计数键 `dispatch_batches`（d）、`revisions`、`rollbacks`、`fixes`、`impacts`（schema 与预算见「记账」） |

阶段状态枚举：`not_started`、`in_progress`、`completed`、`needs_revision`、`blocked`、`skipped`；Change 状态：`in_progress`、`awaiting_human`、`awaiting_info`、`blocked`、`completed`、`completed_with_exceptions`、`cancelled`、`rolled_back`。仅主 Agent 更新状态；回退不删除后续产物，而是标记过期；恢复时核对状态文件、Git 基线、工作区和产物，不一致按「恢复」节仲裁。

`decisions-log.md` **仅**在以下时机追加：用户新约束或裁决；检查点修订/手改要点；用户对未解问题/角色选择的答复；executor `BLOCKED` 回设计时记明的类型与事实；恢复仲裁重写记录。
不记录过程、推测、子 Agent 意见。

**开关表（开关定义唯一载体，记入 `brief.md`）**：

| 开关 | 默认 | 效果 |
|---|---|---|
| `auto_confirm` | off | on 时跳过确认门 |
| `pause_after_design` | off | on 时覆盖 `auto_confirm`，确认门必停 |
| `design_recheck` | off | on 时 `APPROVE` 后 designer `recheck` 一步 |
| `concurrency` | 3 | 并发批每批上限（细则见「并发」节） |
| `commit_policy` | checkpoint | 三态定义与授权语义唯一载体 = `references/git-checkpoints.md`，本表不重复枚举 |
| `piece_acceptance` | off | on 时逐片验证 `PASS` 后进入逐片验收停点（动作定义见 `pipelines/implement-verify.md`「逐片验收」） |
| `role_models` | （空） | 用户显式指定角色→模型时写入（如 `explorer: composer-2.5-fast`），仅覆盖所列角色当次派发 |

生效时机：开关变更在下一次派发或下一检查点生效（先到者为准）；**切换开关不重置已计数**，按新上限比较。新建 `brief.md` 默认 `commit_policy: checkpoint`（省略时视同 `checkpoint`）。

## 启动窗口（流水线首派发前）

五条流水线（`full-flow` / `design-review` / `implement-verify` / `requirement-change` / `bug-feedback`）在首次派发前必开一次确认窗口（时机：建 Change 根与 `.pipeline/` 之后、首次派发之前）；**单点与临时组合不开窗**。三项合并为一次提问：

1. **构成**：流水线阶段链、每阶段角色 × 模式、循环预算（记账键默认上限见「记账」）、预估 `estimate`（口径见「记账」）。
2. **建议选项**：默认 / 自定义 + 各开关当前值（默认值见上「任务记录」开关表）；按任务给方向建议（如改动点已明确 → 调低设计侧预算或直接 `implement-verify`；公共接口、可拆多行为 → 逐片）。
3. **待澄清项**：验收标准、验证命令、验证环境、**独立性等级确认**（三槽位等级定义见 `roles.md`「环境子代理注册」）。

用户选定后把开关与 `independence` 写入 `brief.md` 再开始派发；各流水线前置节保留其特有待澄清项。

## 派发块（路径指针注入）

每次委派 prompt **只传指针与路径**，不把角色文件或产物全文拼进派发参数、不带调度者偏好。首行固定句：`先读取 <skill 目录>/agents/<角色文件>.md 并严格遵守其规则、写入边界、输出结构与返回格式。`（`<skill 目录>` = 本 SKILL.md 所在目录；首次派发前试读一个角色文件，读不到 → 把 `agents/` 复制到 `<change-root>/.pipeline/roles/` 并改用该路径，告知用户一行。）

其余内容：Change 根路径；角色 + **模式**（见 `roles.md`）；输入文件（含 `brief.md`、`decisions-log.md`）；输出文件；轮次；并发时另传步骤编号与文件白名单；追加约束（可选，**只收紧**，见硬约束 9）。派 **executor `implement`** 时无需重复勾选约定（已固化于 `agents/role-executor.md`），仅 Change 有额外勾选口径时再写追加约束。

字面模板（按行传；可选行无则删，不留「无」占位）：

```text
先读取 <skill 目录>/agents/<角色文件>.md 并严格遵守其规则、写入边界、输出结构与返回格式。
Change 根：<路径>    角色 / 模式：<角色>[/<模式>]
输入：<brief.md、decisions-log.md、上游产物路径>    输出：<路径>
轮次：<rN>｜并发白名单：<文件列表>｜追加约束：<只收紧>（均无则删本行）
```

**不传**：`model`（或环境的等效「本次调用模型」参数）。默认使用已注册子 Agent 定义中的模型（见 `roles.md`「环境子代理注册」）；仅当用户在本次输入或 `brief.md` 的 `role_models` 中为该 `pipeline-role` 显式指定模型时，当次派发才可传入对应 `model`。

子 Agent 回执须包含：状态（`COMPLETED` / `NEED_INPUT` / `BLOCKED` / `FAILED`）、产物路径、摘要、证据、未决事项、建议路由。**建议路由仅供参考，只有主 Agent 可以验收产物并推进状态。**

子 Agent 须先读 `decisions-log.md`：视为约束，不得推翻；冲突时在产物中指出，不写该文件。

## 硬约束

1. **独立性**：`roles.md`「独立于」角色之间只传路径，不传被独立方推理或自检结论。  
2. **写入**：角色只写己方产物；仅 executor 可改业务源码。并发实例不写同一文件；调度者不合并产物。  
3. **git / 依赖**：角色不得执行改变工作区状态的 git 命令（`commit`/`push`/`stash`/`checkout`/`reset`/`clean`），检查点提交一律由主 Agent 执行（见硬约束 6）。`commit_policy`（默认 `checkpoint`）默认即授权：主 Agent 自动按检查点提交、不逐次询问提示，用户明示禁止才停（如「不要 commit / 仅落盘」→ `commit_policy: off`）；`push`、发布、生产迁移仍须用户单独授权。不得安装依赖。  
4. **不嵌套**：子 Agent 不得再派子 Agent。  
4b. **模型**：主 Agent **不得**为「提速/省钱/复杂度」自行挑选或更换子 Agent 模型；不得在无用户授权时在派发参数中写 `model`。例外：`brief.md` 的 `role_models` 或用户当轮指令中的显式映射。  
5. **读返回**：以角色「返回」为准；产物有首行 verdict 时**先看首行**。仅当：`FAIL`（失败项/环境阻塞）、`BLOCKED`、`NEED_INPUT`、未解问题数>0、reviewer `REVISE`/`NEED_INFO`、检查点需要摘要时，读产物对应节。  
6. **检查点提交**：`brief.commit_policy` 为 `checkpoint`（或未写明，默认 `checkpoint`）时即默认授权，子 Agent 终态返回后、**下一派发前**主 Agent 自动按 `references/git-checkpoints.md` 判断是否 commit（文档类与源码类均由主 Agent 执行），不逐次询问提示；用户明示禁止（`commit_policy: off`）才停。commit 后更新 `pipeline-state.last_checkpoint_commit` 与 `run.log`。  
7. **失败**：返回不合规 → 重派一次并指出问题；仍失败 → 汇报用户。**接续判定**：重派时若输出文件已存在、且契约规定产物首行带 verdict/status 而首行缺失或为占位 → 本次未完成，派发注明「接续」按契约从断点续写；若返回已给出 verdict 而首行缺失或未定稿 → 按「返回不合规」重派，不判接续。限流/过载：并发数减半（最低 1）记入 `brief.md`，重派并告知。  
8. **提问合并**：对用户的阻塞问题合并为一次提问。  
9. **追加约束只收紧**：派发块的追加约束不得放宽角色文件的规则。  
10. **读取边界**：调度者整节读取产物仅限白名单——验证 `FAIL` → 报告失败项与环境阻塞节；实施 `BLOCKED` → 执行记录「偏离」节；未解问题数>0 → 「未解问题」节；并发白名单外改动定性 → 设计「改动清单/实施步骤」；流水线正文（检查点、停点、汇报、异常处理）明文要求的节。恢复核对不受限。  
11. **派发后校验**（只读，调度者自做）：每个角色返回后运行 `git status --short`，与 `baseline.md` 基线及各实施记录「改动文件」比对；出现白名单外或 Change 外改动（`.pipeline/` 编排产物除外）→ 不采信本次返回，列出文件交用户决定，不自行还原。非 git 仓库跳过并在 `run.log` 注明。

## 记账

**计数唯一来源 = `pipeline-state.yaml`**（schema 见「任务记录」）；`run.log` 行内标签仅作审计与展示。本表为循环预算唯一载体：

| 键 | 计数对象 | 默认上限 | 耗尽动作 |
|---|---|---|---|
| `revisions k/N` | 设计修订（`REVISE` 自动循环） | 3 | 三项选择 |
| `rollbacks j/上限` | 回设计 / 回方案 | 2 | 三项选择 |
| `fixes <键> k/R` | 修复轮（键 = Task / 集成 / 整体，与验证报告后缀一致） | 3 | 三项选择 |
| `impacts k/上限` | 影响分析 / 分诊补查 | 2 | 三项选择 |

- **不变量**（每次覆写 `pipeline-state.yaml` 前自查五条）：k ≤ N；j ≤ 上限；d 单调不减；`current_stage` 在既有枚举内；`next_action` 非空。分母与 `brief.md` 不一致 → 以 `brief.md` 为准重写并记 `decisions-log.md`。
- `d = dispatch_batches` 只作成本与对账口径，任何情形下不触发停线。
- 循环计数只数规则自动触发的修订 / 回设计 / 回方案 / 修复派发；人工修订、再评审、补探索、失败重派**不计入**。例外：逐片验收停点的人工修复**计入修复轮**（`fixes` 键，`run.log` 标 `修复 <片> k/R`；动作定义见 `pipelines/implement-verify.md`「逐片验收」，计数规则以本表为唯一来源）。回设计后设计修订计数 k 清零（下一次修订记 1/N）。
- **补探索**（`01-explore-补-N.md`）：整个任务最多自动 1 轮（以该文件是否已存在判定）；第 2 次须停下询问，获批后记 `decisions-log.md`。回设计 / 回方案的重定位探索不占补探索额度。
- **成本软提示**：派发返回后 d > `estimate` × 1.5 → 向用户输出一行 `成本提示：d（批次）<d> / 预估 <estimate>`，不停线不询问；每次跨越阈值提示一次（预估重算上调后再次跨越，视为新的跨越）。
- **`estimate` 口径** = 各流水线阶段表派发行数为基线、循环增量按记账上限代入；仅用于启动告知、软提示与结束对账，不作上限。设计或开关变更影响构成（片数、粒度、开关）时重算并更新 `brief.md`。

## 并发

- 同一条消息内多个子 Agent 调用 = 并发；分多条 = 串行。整批返回后再下一步。  
- 每批 ≤ `brief.md` 并发数（默认 3；用户指定优先）。超出则分波。  
- 条件：实例互不依赖、输出文件互异。  
- 只写产物类：调度者划分不重叠范围（目录/模块）。  
- 改源码类（全部满足才并发）：各有**文件白名单**且两两不交；不依赖对方产出；不含共享/生成文件（barrel、路由注册、i18n 汇总、lockfile、全局类型）→ 这些内容放在并行组之后的 executor **`integrate`** 模式。  
- explorer 可按区域并行（输出 `ADR-探索记录-<区域>.md`）。  
- **验证一律串行**：验证派发不与其他实例并发（共享工作区、端口、构建缓存）。  
- **白名单核对整批返回后做一次**：将 `git status --short` 改动逐一归因到已返回实例的白名单（并发在途时不做逐实例核对——他片白名单内的合法改动也会出现在 status 中，逐实例核对必然误报）；白名单外改动按各实例实施记录「改动文件」归因，无法归因 → 停下询问用户，不臆断。  
- **跨片失败归因**：证据仅指向他片白名单文件的失败，verifier 标 `他片相关`，不计本片 `FAIL` 与修复轮；反馈对应片修复，待其 `PASS` 后重验本片。  
- **run.log 批次后缀**：同批并发多实例共用批次序号、各行加后缀 a/b/c（格式见「任务记录」`run.log` 行）。  
- 整批后处理：任一 `BLOCKED` → 不派后续波。

具体阶段能否并发由 `pipelines/*.md` 规定。

## 角色解析（工具环境支持时）

1. 用户/AGENTS.md 指定 agent  
2. `description` 或 frontmatter 含 `pipeline-role: <角色名>`（见 `roles.md`「环境子代理注册」）  
3. `roles.md` 登记之 `agents/role-*.md` 已注册为 subagent（`name` 字段与表内一致）  
4. 兜底：generalPurpose + 必读 `<skill>/agents/role-*.md` 绝对路径  

**模型**（与上表独立）：派发时不默认传 `model`。解析顺序：`brief.role_models[<pipeline-role>]` 或用户当轮显式指定 → 环境子代理注册文件（本仓库为 `~/.zcode/agents/role-*.md`）中的 `model` → 环境默认。主 Agent 不在此链条中插入自选模型。

副本校验：若环境存在同名 agent 定义，与本 skill `agents/` 源文件 diff——仅 frontmatter 差异可忽略；正文不一致则停，提示用户用源文件更新注册副本。

- 「改源码」类 agent 须有 Edit；「只写产物」不得 Edit。  
- 互为「独立于」的角色不得同一 agent 兼任。  
- 派发时附协议补丁：遵守角色文件的规则、输出、返回、当前模式。

## 必须停下询问用户

任何开关下均停：轮次预算耗尽；验收标准需改；破坏性操作；人工意见标「不可行」；`BLOCKED(需确认)`；verifier 环境阻塞（见 `pipelines/implement-verify.md`）。各流水线检查点另有补充。

任一记账上限耗尽 → 停下给**三项选择**（统一措辞）：① `追加 <数>`（对应上限上调，更新 `brief.md` 并记 `decisions-log.md` 后继续）；② `接受当前结果`（记 `decisions-log.md` 并进入「结束汇报」）；③ `终止`（保留产物，按「结束汇报」模板汇报）。

## 恢复

会话中断后继续：

1. 读 `brief.md`、`pipeline-state.yaml`、`run.log` 末行、相关产物首行 verdict，据此判定所处阶段与在途批次是否完成；向用户一句话确认后继续。无 `run.log` 时按产物与 state 推断。  
2. **一致性仲裁**：`pipeline-state.yaml` 与产物不一致 → **以产物为准**重写 state，并在 `decisions-log.md` 记一行；state 计数分母与 `brief.md` 不一致 → 以 `brief.md` 为准重写并记一行。  
3. **接续判定**：产物首行 verdict/status 缺失或为占位 → 该次未完成，按硬约束 7 以「接续」重派。  
4. **同批并发中断**：已返回实例以产物为准；未返回实例重派（记新批次）。  
5. 逐片任务：片是否完成以该片最新验证报告首行 `PASS` 为准。

## 结束汇报

Change 结束（含终止）时按本模板汇报（模板唯一载体），总计 ≤20 行，无内容的项整项省略；`pipelines/` 可要求补充项。必含六项：

1. 产物路径（Change 根、`.pipeline/` 产物目录）与各角色关键 verdict；
2. 改动文件摘要：基线以来改动文件 ≤3 行，超出按目录归并并指向 `git diff --stat`（非 git 仓库写改动文件数）；
3. `UNVERIFIED` / 需人工 / 环境豁免项；
4. 遗留问题与遗留 minor（≤5 条标题）；
5. 回滚点：最近检查点 `last_checkpoint_commit` 与回滚方式（见 `references/git-checkpoints.md`「与回滚 / 恢复」）；
6. 对账：`d（批次）/ 预估`、各记账键轮数（revisions / rollbacks / fixes / impacts）。

## 维护

修改本文件、`SKILL.md`、`pipelines/`、其他 `references/` 文件后，交付前须：markdownlint 0 错误；全文相对引用核对（被引文件存在、无指向已删文件的引用）；推演一条受影响场景（状态转移、计数与停点符合预期）。新增角色或流水线须先经设计评审（`design-review`）后落地。
