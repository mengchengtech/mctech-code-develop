# 角色定义

| 角色 | Agent 文件 | 权限类 | 模式 | 独立于 | 产物 | 返回 |
|---|---|---|---|---|---|---|
| explorer | `agents/role-explorer.md` | 只读 | 可并行 | designer、reviewer、planner、executor、verifier | 探索记录、需求与影响分析 | 事实证据、影响面、未知项、分析结论 |
| designer | `agents/role-designer.md` | 文档写入 | 串行主导 | explorer、reviewer、planner、executor、verifier | 设计 ADR、OpenSpec proposal/specs/design | 设计方案、备选方案、权衡、风险、未决项 |
| reviewer | `agents/role-reviewer.md` | 只读 | 独立审核 | designer、executor | 审核 ADR | APPROVE / REVISE / REJECT / NEED_INFO 及证据 |
| planner | `agents/role-planner.md` | 文档写入 | 串行主导 | explorer、designer、reviewer、executor、verifier | 实施计划 ADR、OpenSpec tasks | 任务图、依赖、批次、验收点、回滚点 |
| executor | `agents/role-executor.md` | 代码写入（限定范围） | 按批次串行；批次间可并行评估 | reviewer、verifier | 代码差异、执行记录、测试代码 | 修改摘要、文件清单、命令、风险、阻塞 |
| verifier | `agents/role-verifier.md` | 只读/验证命令 | 与 executor 串行交接 | executor、designer | 验证报告、测试证据 | PASS / FAIL / BLOCKED / SCOPE_CHANGE 及证据 |
