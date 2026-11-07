# 编写指南（bindsang-pipeline）

新增角色或流水线时读。完成后按末尾清单自检。协议见 `references/orchestrator-protocol.md`；产物见 `references/artifacts.md`。

## 新增角色

1. 写 `agents/role-<名>.md`，建议含：角色目标、**模式**表、职责、规则、输出结构（含首行 verdict 若有）、完成判据、**返回**（行数上限可选）。  
2. 在 `roles.md` 增一行：权限类、模式、独立于、产物、返回。  
3. 评审/验证类在「独立于」填设计方或执行方。

可选 frontmatter（Cursor 等环境）：

```yaml
---
name: role-xxx
description: …由 bindsang-pipeline 主 Agent 指派…
tools: Read, Grep, Glob, Bash, Write  # 改源码类加 Edit
---
```

**自检**：tools 与权限类一致；规则含 `decisions-log.md` 只读；路径以派发为准；模式输入/输出写清；不写死某条流水线阶段名（用 Change 根相对路径）。

## 编写流水线

1. 触发：直接使用 / 先问再用 / 不使用。  
2. 前置：`brief.md` 必填项。  
3. 阶段表：角色、模式、读、写、返回 → 下一步。  
4. 循环与预算：回头转移最多 N 轮；耗尽 → 停。  
5. 检查点：展示内容；用户指令表。  
6. 并发：引用 orchestrator-protocol「并发」，只写本流水线补充。  
7. 登记 `pipelines/<id>.md`；更新主 Skill 任务类型表（若新 id）。

### 流水线自检

1. 每个 verdict 有转移；终态明确。  
2. 回头有预算。  
3. 角色均在 `roles.md`；独立角色不同 agent。  
4. 并发输出文件互异；改源码有白名单与集成步骤。  
5. 含 executor 则有 verifier（或写明例外）。  
6. 检查点覆盖「必须停下询问」。  
7. 不复述 orchestrator-protocol 全文，只引用。

## OpenSpec 与实施设计桥接

implement 依赖 `tasks.md` + `design.md`。若需 dev-pipeline 式**并行组/集成步骤**，在 planner ADR 或 `design.md` 明确「实施步骤」节；executor 并发模式读取该节。
