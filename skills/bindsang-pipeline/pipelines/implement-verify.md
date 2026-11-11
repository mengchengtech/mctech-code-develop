# 只实现和验证（implement-verify）

`preflight → planner*` → (executor ↔ verifier)* → close`

*已有获批 `tasks.md` 可跳过 planner。不改 OpenSpec 除非 `SCOPE_CHANGE` 流程。

## 触发

- **直接使用**：已确认设计/OpenSpec，要求实施+验证。  
- **先问再用**：≥5 文件或验收需独立 HTTP/浏览器验证 → 说明代价后询问。  
- **不使用**：无设计 → `full-flow`；小改、已定位 bug；只验证 → 单点 verifier。

## 前置

合并一次提问。

1. preflight（若尚无 Change）。  
2. 基线写入 `brief.md` / `baseline.md`：`git branch`、`rev-parse --short HEAD`、`status --short`；脏工作区问用户如何处理。  
3. `brief.md`：**验收标准**、验证命令、验证环境（凭据只记变量名）、角色映射。  
4. 须有获批 OpenSpec + `tasks.md`（或等价设计中的实施步骤）；缺节 → 问：走 `design-review` 补全或终止。  
5. 设计含并行组 → 告知用户并发数、组数、波数。  
6. 测试计划含浏览器/HTTP 而无环境 → 询问并写入 `brief.md`。

独立运行时，设计须 reviewer `APPROVE` 或用户确认记入 `decisions-log.md`。

## 阶段

| 阶段 | 角色 | 模式 | 读 | 写 | 返回 → 下一步 |
|---|---|---|---|---|---|
| 实施 | executor | `implement` / `concurrent` / `integrate` / `resume` / `fix` | `brief.md`、`tasks.md`、OpenSpec、执行 ADR* | 源码 + `.pipeline/execution/执行记录-rN.md` | `COMPLETED` → 验证；`BLOCKED` → 受阻 |
| 验证 | verifier | `task` / `change` | 上列 + 执行 ADR | `.pipeline/verification/ADR-验证报告-rN.md` | `PASS` → 结束；`FAIL` → 循环 |

另读 `decisions-log.md`。

## 循环与预算

- **Git 检查点**：`commit_policy=checkpoint`（默认，须用户授权 commit）时，每 Task `COMPLETED` → commit（C8）；该 Task verifier `PASS` → commit 验证里程碑（C9/C7）。并行组在 `integrate` 完成后 commit（C10）。见 `references/git-checkpoints.md`。
- **实施-验证循环**：`FAIL` 且分类为实现类（见 verifier 表）→ executor `fix` → verifier。最多 **3 轮**；耗尽 → 停。  
- **环境阻塞**：verifier 报告环境阻塞 → 不派 fix；问用户：解除后重验 / **环境豁免**（`decisions-log.md` + `brief.md` 标注）/ 终止。  
- **实施受阻**（executor `BLOCKED`）：

| 类型 | 动作 |
|---|---|
| 拆分过粗 / 设计遗漏 / 设计事实有误 | 记入 `decisions-log.md`；独立运行停并建议 `design-review`；`full-flow` 自动回设计（计入回设计次数） |
| 需确认 | 主 Agent 询问破坏性操作 |

- **resume**：设计修订后，读全部执行 ADR + 新 `tasks.md`/design，只做未完成或受影响 Task。

## 并发

仅 executor `implement`/`concurrent`。条件：

1. `design.md` 或规划 ADR 标并行组且 reviewer 检查项 8 通过（无审核则串行）。  
2. `orchestrator-protocol.md`「改源码」并发规则。  

派发：步骤编号、**文件白名单**、输出 `执行记录-<组>.md`。全部组完成且无 `BLOCKED` → 串行 `integrate`。

## 检查点

无固定检查点；适用「必须停下询问」+ 环境阻塞 + 破坏性 `需确认`。

## 验证失败路由

verifier `SCOPE_CHANGE` 或 SPEC/DESIGN/NEW_REQUIREMENT 类 → 主 Agent 路由 explorer/designer/planner，非 executor 擅自改 Spec。

## 结束

汇报：改动文件、验证结论、`UNVERIFIED`/需人工、环境豁免、遗留问题（≤15 行 + `orchestrator-protocol.md`）。
