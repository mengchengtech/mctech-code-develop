# 权限重构完整流流程

这是一个真实的案例，在自主运维平台开发了一个月后，基本功能和简单的权限已经做完了的情况下，对整个系统的权限功能包括权限重新定义，权限模型，授权方式和验权方式都做了一次推倒性的重构。以下是首次使用ai分析给出的流程的粗粒度基础上结合实际项目开发和反复调整，得出的以下完整的实践过程。
每一步与 AI 实际交互的联天内容都保存到 `step*` 开头的文件里，可以对照着查阅

从 step9开始，切换到 `mctech-sdd-*` 命令模式，不再每次都贴大段的 Prompt 给 AI 模型了

## 整体流程

采用 8 个阶段：

① 项目探索
      ↓
② 权限现状建模
      ↓
③ 重构方案分析
      ↓
④ 架构决策 / 人工确认
      ↓
⑤ 生成 Spec 文档
      ↓
⑥ 任务拆分
      ↓
⑦ Agent 分任务实施
      ↓
⑧ 集成验证 + 归档

在这些步骤中有一个很重要的原则：
①～③ 不允许修改代码。
④ 必须由架构师确认关键设计。
⑤～⑥ 固化设计。
⑦ 才允许 Agent 修改代码。

## Step 1：项目探索

这一步最重要的是只探索事实，不要让 AI 设计方案。
实际使用的 Prompt 如下：

```plain
你现在处于“系统探索阶段”，目标是理解现有项目中的权限项、授权和验权体系。
本阶段严禁修改任何代码、配置、数据库结构和测试文件，也不要提出具体重构方案。
请对当前代码库进行系统性探索，重点分析：
1. 权限相关的数据模型
   - 用户
   - 用户组/角色
   - 权限
   - 权限项
   - 资源
   - 操作
   - 菜单
   - API
   - 数据权限
   - 租户权限
2. 授权流程
   - 谁负责给用户/角色授权
   - 授权入口有哪些
   - 授权数据保存在哪里
   - 授权后如何刷新缓存
   - 是否存在多套授权逻辑
3. 验权流程
   - HTTP API 验权
   - Gateway 验权
   - Service 内部验权
   - RPC 调用
   - MQ Consumer
   - 定时任务
   - WebSocket
   - 前端权限控制
4. 搜索所有权限判断代码
   不要只搜索 PermissionService。
   请通过：
   - 方法调用
   - 注解
   - middleware/filter/interceptor
   - permission code
   - role code
   - resource/action
   - 用户上下文
   - token/session
   - ACL
   - RBAC
   等方式尽可能完整地发现权限相关实现。
5. 分析模块依赖关系。
6. 分析数据库表、Redis、缓存以及权限相关配置。
7. 分析前端和后端之间的权限约定。
8. 分析现有测试。
最终不要修改代码。

请输出：
A. 当前权限架构概览
B. 权限数据模型
C. 授权流程
D. 验权流程
E. 权限相关模块清单
F. 权限相关 API 清单
G. 权限相关数据库/缓存清单
H. 所有已发现的验权入口
I. 模块依赖关系
J. 当前实现中的问题和技术债
K. 尚未确认的信息

对于每一个结论，请尽可能给出对应的代码文件、类、方法或配置位置。

注意：
- 区分“代码事实”和“你的推测”
- 不要因为发现某一种实现就假设整个系统都是这种实现
- 如果存在多个实现，请分别列出
- 如果无法确认，请明确标记为“待确认”
```

输出结果可以让AI保存到项目指定的目录中，这一步的目标主要是收集信息归纳总结分析现状。完成后生成的目录结构大致如下：

```plain
permission-exploration/
├── architecture.md
├── data-model.md
├── authorization-flow.md
├── verification-flow.md
├── permission-entrypoints.md
├── module-dependencies.md
├── cache.md
└── open-questions.md
```

这些不是最终 Spec，它们只是系统事实库。

## Step 2：让 AI 专门做“权限验权入口扫描”

软件系统里最容易漏的就是：“某个业务模块里面藏着一段自己的权限判断。”

上一步是从面上扫描系统里与权限相关的模块，代码，数据结构等信息；这一步是精细化扫描，只关注权限实现相关的代码，目的是把整个系统中统一的和非统一的权限相关的功能全部识别出来。

这一步实际使用的 Prompt 如下：

```plain
基于刚才的系统探索，现在只做“权限验权入口完整性分析”。
不要修改代码，也不要设计新方案。
目标是尽可能找出当前系统所有实际执行权限判断的位置。
请按照以下类别扫描：

1. HTTP Controller / API
2. Gateway Filter
3. Middleware
4. Interceptor
5. Annotation / Decorator
6. Service 方法
7. RPC Client / Server
8. MQ Producer / Consumer
9. Scheduled Job
10. WebSocket
11. GraphQL（如果存在）
12. 文件/对象存储访问
13. 数据库查询层
14. 前端路由
15. 前端按钮/组件权限
16. 其他自定义权限判断

对每个入口记录：
- 模块
- 文件
- 类/函数
- 当前验权方式
- 使用的权限标识
- 权限数据来源
- 用户身份来源
- 是否依赖缓存
- 是否存在绕过路径
```

最终输出的这个权限项矩阵非常重要，后面还会用到。

## Step 3：让 AI 分析“应该怎么重构”

这一步进入方案分析。让 AI 先根据前两步的结果先自己决策该怎么重构，设计新版权限功能。这一步需要先告诉 AI 项目里无法分析出来的一些背景信息，比如为什么要重构，现在的权限项存在哪些问题，希望有哪些功能，给 AI 重构指一个方向；另一方面要告诉 AI 希望在设计结果中体现出自己要关注的哪些方面的问题和实现细节。以及要求实现多种方案做对比

注意这一步里仍然：不要写代码，也不应该生成 Spec 文档。

这一步实际用的 Prompt 是这样的：

```plain
现在进入“重构方案分析阶段”。
基于前面已经完成的系统探索结果，对现有权限项授权和验权体系进行重构方案分析。
本阶段仍然禁止修改代码。

请重点解决以下问题：
1. 当前权限模型存在哪些根本性问题？
2. 授权模型和验权模型是否应该解耦？
3. Permission、PermissionItem、Resource、Action 应该如何定义？
4. 用户、角色、权限之间应该采用什么关系？
5. 权限项应该表示什么？
6. 验权时应该使用 permission code，还是 resource + action，还是其他模型？
7. Gateway 是否应该验权？
8. Service 是否应该再次验权？
9. 内部 RPC 是否需要验权？
10. MQ、定时任务如何处理？
11. 前端权限和后端权限如何保持一致？
12. Redis 缓存应该缓存什么？
13. 权限变更后如何失效？
14. 如何处理多租户？
15. 如何兼容现有 API？
16. 如何进行数据库迁移？
17. 如何灰度迁移？
18. 如何回滚？

至少提出 2～3 个可行架构方案。每个方案说明：
- 架构
- 优点
- 缺点
- 性能
- 安全性
```

## Step 4：架构师做决策

这里是整个流程最关键的一步。AI 给出了 A / B / C 三种方案。不要让 Agent 自己决定最终架构。只做方案性的决策，比如选择的开发语言，采用的框架和协议等等，不要过多的干预实现细节，那是后面步骤要做的事。

这一步实际使用的 Prompt 如下：

```plain
以下是架构师已经确认的决策。
请将以下内容视为已经确定的架构约束，不要自行修改：
1. 验权采用 Website + Service 双层模型。
2. Website 负责 API 级粗粒度验权。
3. Service 负责业务资源级细粒度验权。
4. 所有 Service 不允许自行实现权限判断算法。
5. Permission 采用 Resource + Action 模型。
6. PermissionItem 表示可授权的权限项。
7. Redis 保存经过计算后的用户权限集合。
8. 权限变化后必须触发缓存失效。
9. 未上线项目，不需要兼容现有 permission。
10. 未上线项目，不考虑迁移。
11. 未上线项目，不考虑灰度发布和回滚。

请基于这些已经确定的架构决策，重新整理最终技术方案。

如果发现这些决策之间存在矛盾，不要自行修改，请列出冲突供我确认。
```

AI 在收到以上信息后会和以前的结果做合并，然后检查是否存在冲突，是否存在不确定的点需要人工再确认。然后把合并后的结果按类别一项一项列出来。存在冲突或要再次确认的点在会在最后单独以带编号的表格形式列出来供人工再决策

开发人员只需要回复上面编号决策结果即可，例如A1同意, B2a放弃,C1-3否

## Step 5：正式进入 OpenSpec

现在才真正开始生成Spec文档，可直接在 Prompt 里让 Agent 创建 Change。这里不能让 AI “自由发挥”。

实际使用的 Prompt 是这样的：

```plain
现在基于已经确认的架构决策，为权限系统重构创建 OpenSpec Change。
Change 名称：refactor-permission-authorization
目标：重构现有权限项授权和验权体系，使系统形成统一的权限模型、授权模型和验权模型，并支持 Gateway + Service 双层验权、权限缓存。新项目不考虑灰度迁移和旧权限模型兼容。

需要创建：
1. proposal
2. design
3. specs
4. tasks

要求：
proposal：
说明为什么重构、当前问题、目标和非目标。
design：
详细描述：
- Permission Model
- PermissionItem Model
- Resource / Action
- Authorization Model
- Verification Model
- Gateway
- Service
- Authorization SDK
- Cache
- Permission Change Event
- Compatibility
- Migration
```

## Step 6：不要马上执行 Tasks，再做一次“任务审查”

对方案转化的任务进行 Review 这是很多 AI Coding 流程容易缺失的一步。目的是避免单一模型对  设计方案实现不全面，或者上下文限制（比如压缩了），导致 AI 对整个设计转化不全，或失真。检查的时候使用的上下文要远小于设计和生成任务时的上下文

实际中使用的 Prompt 如下：

```plain
现在不要执行任何 task。
请审查当前 OpenSpec Change 的 tasks 是否适合由 AI Agent 分批执行。
重点检查：
1. 每个任务是否有明确边界？
2. 是否存在一个任务同时修改过多模块？
3. 是否存在任务依赖未声明？
4. 是否存在循环依赖？
5. 是否有任务需要先建立基础能力？
6. 是否存在多个任务同时修改同一核心文件？
7. 是否存在两个任务可能产生冲突？
8. 是否有数据库迁移任务？
9. 是否有缓存迁移任务？
10. 是否有兼容层任务？
11. 是否有测试任务？
12. 是否有灰度任务？
13. 是否有回滚任务？
14. 是否有最终清理旧实现的任务？

请把 tasks 分为：
Foundation
Infrastructure
Migration
Business
Verification
Cleanup

给出明确的执行顺序。
仍然不要修改代码。
```

## Step 7：开始执行，但是“一次只执行一个任务”

到这里就和前面完全不同了。之前是让 AI 只思考，产出设计，不让编写任何代码；到这一步角色变成了只是按 tasks.md 里的任务编辑代码/测试用例，不要胡乱思考，增加设计方案之外的想法。

一次大型的重构或新功能开发生成的 tasks.md 不会只有几项。一般来说任务都会分为 1.1/1.2/1.3....,2.1/2.2/2.3...., .....。不要告诉 AI：“开始执行整个 tasks.md。”。好的方式是一次只执行一个大类的命令，例如第一次执行 1.x 完成后做一次检查，跑完之前测试用例。检查没问题后再做 2.x 任务。

遇到设计问题时，不要让 Agent 自己拍脑袋。比如 Agent 执行 Task 5 时发现了问题：

```plain
老系统的 permissionCode 和新的 resource/action 无法一一对应。
```

不要让它自己增加一个兼容字段，应该让它停。这会极大降低 Agent “越改越偏”的概率。

实际上使用的 Prompt 是这样的：

```plain
现在执行 OpenSpec Change：refactor-permission-authorization
只执行 Task 1
不要执行其他 task。

执行前：
1. 阅读 proposal
2. 阅读 design
3. 阅读 specs
4. 阅读 Task 1
5. 检查当前代码状态
6. 确认 Task 1 的依赖已经满足

执行要求：
- 严格按照 OpenSpec spec 实现
- 不修改 Task 1 之外的架构
- 不顺便重构其他模块
- 如果发现 design 与现有代码冲突，停止并报告
- 如果发现需要修改其他 task 的设计，停止并报告
- 不要自行扩大任务范围

完成后：
1. 修改代码
2. 增加/修改测试
3. 执行相关测试
4. 检查 lint/typecheck
5. 总结修改的文件
6. 总结测试结果
7. 列出发现的问题
8. 判断 Task 1 是否可以标记完成

如果实施过程中发现以下情况，必须停止当前任务：
1. OpenSpec 与实际代码冲突
2. 发现新的权限模型
3. 发现未记录的权限入口
4. 需要修改架构设计
5. 需要改变数据库模型
6. 需要改变公共 API
7. 需要修改其他 Task 的职责
8. 无法确定兼容策略

不要自行做架构决策。

请输出：
- 当前任务
- 发现的问题
- 相关代码
- 对原设计的影响
- 可选解决方案
- 推荐方案
- 是否需要修改 OpenSpec

等待架构师决定后再继续。

不要执行 Task 2
```

最后的这个限制非常重要。目的是强制让 AI 在做完 Task 1下的任务后停下来，不要自动转入 Task 2执行。有的工具可能会在用户长时间没有交互响应的时候自行判断，选择后面的任务开始执行

## Step 8：每完成几个任务，做一次集成审查

例如：

```plain
Task 1～5 完成
        ↓
Integration Review
        ↓
Task 6～10
```

审查的目的是发现 AI 编写的代码与设计文档中的描述是否一致；是否有遗漏的功能没有实现；是否有只完成一半开发的功能；之前编写的单元测试是否能完全跑通。这一步不能完全依赖 AI 自己完成。比如需要集成测试的，需要带着真实数据库，在浏览器里完整操作的。

如果不差钱的话，可以随时做审查，多多益善

这一步实际使用的 Prompt 是这样的：

```plain
当前已经完成：Task 1-5

现在不要继续开发。请进行一次 OpenSpec 与代码实现的一致性审查。

检查：
1. 实际代码是否符合 design？
2. 是否符合 specs？
3. 是否出现设计漂移？
4. 是否出现重复权限逻辑？
5. 是否出现新的权限入口但没有进入 SDK？
6. Gateway 和 Service 验权职责是否清晰？
7. 缓存策略是否一致？
8. 权限变更是否正确失效缓存？
9. 旧 permissionCode 兼容逻辑是否符合设计？
10. 是否出现绕过 Authorization SDK 的实现？
11. 测试是否覆盖核心场景？

输出：PASS / FAIL
如果 FAIL：列出问题、影响和建议修复任务。

不要直接修改代码。
```

## Step 9：最终验证

所有业务模块迁移完成以后，再让 Agent 做一次全局验证。

Prompt可以这样写：

```plain
权限重构的所有 Implementation Task 已完成。现在进入最终验证阶段。不要新增功能，也不要主动修改架构。请基于 OpenSpec 对整个代码库进行验证。

重点检查：
1. 所有原权限入口是否已经迁移
2. 是否还有旧权限判断代码
3. 是否存在绕过 Authorization SDK 的权限判断
4. Gateway 验权是否正确
5. Service 验权是否正确
6. RPC 是否正确
7. MQ 是否正确
8. 定时任务是否正确
9. 前端权限是否兼容
10. Redis 缓存是否正确
11. 权限变更是否能够立即/最终生效
12. 多租户隔离是否正确
13. 新旧权限模型兼容是否正确
14. 数据迁移是否完整
15. 回滚方案是否可执行
16. 测试覆盖是否足够

请生成：
- Spec Compliance Report
- Migration Completeness Report
- Security Risk Report
- Remaining Technical Debt
- Cleanup Candidates

不要直接删除旧代码。
```

## 最终 Cleanup 单独做

这个非常重要。权限重构的时候，新实现/旧实现/兼容层，通常需要运行一段时间。所以不要：
新实现完成 -> 立即删除旧实现。应该：

新实现 -> 双读/双验权 -> 灰度 -> 观察 -> 确认 -> Cleanup

Cleanup 再单独建 Task：

```plain
Task Cleanup-1
删除旧 PermissionService

Task Cleanup-2
删除旧 permissionCode

Task Cleanup-3
删除旧数据库字段

Task Cleanup-4
删除旧缓存结构
```

以上是第一轮流程完整跑完需要的步骤，前面八步在做较大的功能变更或重构时必须要使用。最后一步根据实际情况决定是否需要。

## 任务执行中发现要变更设计怎么办

如果openspec 的change 里的任务已经执行几步了，发现需要调整需求，例如调整界面，改逻辑该怎么处理呢？关键是不要直接对正在执行的 Task 说“改一下需求”，而应该让模型先判断：

>这是“实现偏差”，还是“需求变更”`

如果确实是需求变更，就应该先暂停当前执行 → 分析影响 → 更新 Spec/Design/Tasks → 再继续 Apply。

假设已经执行：

```plain
Task 1 ✅
Task 2 ✅
Task 3 ✅
Task 4 🚧
Task 5 ⏳
Task 6 ⏳
```

现在发现以下几个问题需要调整原来的设计：权限管理界面需要调整，而且授权逻辑也要改变。此时不要直接告诉模型 `继续执行 Task 4，但是把界面改成 XXX`。因为这样会导致：Spec和Task对应着设计A，但是代码变成了实现B，最后导致 OpenSpec 失真。

正确流程是：

```plain
当前实现
   ↓
提出需求变更
   ↓
Impact Analysis
   ↓
修改 Proposal / Design / Spec
   ↓
重新评估 Tasks
   ↓
继续 Apply

```

### 提出需求变更 / Impact Analysis

实际中使用的 Prompt 如下：

```plain
当前 OpenSpec Change：refactor-permission-authorization

当前执行状态：Task 1-3 已完成, Task 4 进行中

现在发现新的需求变化：

1. 权限管理界面需要调整。
2. 原来的权限项平铺展示，改为按照 Resource 分组。
3. 授权操作从直接选择 PermissionItem，调整为选择 Resource 下的 Action。
4. 后端授权逻辑也需要适配这个变化。
5. 已经完成的 Task 不希望无必要地重新实现。

现在不要修改代码，也不要继续执行当前 Task。

请进入“需求变更影响分析”模式。

请分析：

1. 当前 OpenSpec 中哪些 proposal/design/spec 受到影响？
2. 哪些已经完成的 Task 受到影响？
3. 哪些正在执行的 Task 受到影响？
4. 哪些未执行 Task 受到影响？
5. 当前代码哪些部分已经与新需求冲突？
6. 哪些部分可以保持不变？
7. 是否需要修改数据模型？
8. 是否需要修改 API？
9. 是否需要修改前端？
10. 是否需要修改缓存或权限计算逻辑？
11. 是否需要重新设计后续 Task？
12. 是否需要回滚已经完成的 Task？

请区分：

A. 需求变更导致的影响
B. 当前实现错误
C. 不受影响的部分

不要修改任何代码。

最后给出一个“Change Impact Report”。
```

然后就等着模型自行判断“改 Spec 还是改代码”

### 修改 Proposal / Design / Spec

人工确认，或者提出新的意见再调整。这一步用实际用的 Prompt 是这样的：

```plain
刚才的 Change Impact Analysis 已确认。

架构师确认以下需求变化正式生效：

1. 权限管理界面改为 Resource → Action 两级展示。
2. 用户授权以 Resource + Action 为基本单位。
3. PermissionItem 仍然保留，但作为系统内部权限定义，不再作为主要的 UI 授权单位。
4. 后端 Authorization API 需要支持 Resource + Action。
5. 现有 permissionCode 兼容机制保持不变。
6. 已完成且不受影响的 Task 不重新执行。

现在请更新 OpenSpec。

要求：

1. 更新 proposal.md 中受影响的目标描述。
2. 更新 design.md 中受影响的设计。
3. 更新相关 specs。
4. 不要删除仍然有效的原始需求。
5. 对发生变化的需求明确体现新旧行为差异。
6. 检查不同 spec 之间是否出现矛盾。
7. 重新分析 tasks.md。
8. 标记：
   - 已完成且无需修改
   - 已完成但需要补充
   - 当前 Task 需要修改
   - 未执行但需要修改
   - 新增 Task
   - 可以删除的 Task
9. 不要执行任何代码修改。

最后输出：
- Spec Changes
- Design Changes
- Task Changes
- Completed Task Impact
- New Tasks
- Removed Tasks
```

### 重新规划 Tasks

更新完 Spec 后，不应该直接继续原来的 Task。应该重新规划新的tasks。有的时候模型在上一步确认完成后没有不确定的/冲突的地方了，可能会直接开启规划新的任务，如果不想这样，可以在上一步的 Prompt 里显示禁止。

实际使用的 Prompt 如下：

```plain
新的 Spec 已经确认。

请重新审查当前 Change 的任务计划。

当前状态：

Task 1 ✅
Task 2 ✅
Task 3 ✅
Task 4 🚧
Task 5 ⏳
Task 6 ⏳

请根据最新 Spec：

1. 判断已完成 Task 是否仍然符合 Spec。
2. 判断当前 Task 是否需要修改。
3. 判断未执行 Task 是否需要修改。
4. 创建必要的补偿 Task，而不是无条件重新执行已完成 Task。
5. 调整 Task 依赖关系。
6. 确保不存在重复实现。
7. 确保最终所有 Task 能覆盖最新 Spec。

输出新的执行计划。

不要修改代码。
```

生成新的 Tasks 后再回到前面执行任务的Step里继续往下执行

再次强调一下：尤其是已经 Apply 了一部分以后，千万不要把“最新需求”直接塞进当前 Task。**这是大型 SDD 项目最容易产生 Spec Drift 的地方。

正确的思想是：

> 需求变化不是“修改任务”，而是“修改目标状态”，然后根据新的目标状态重新计算剩余任务。

这样即使一个 OpenSpec Change 已经执行到 60%，需求发生变化，也不会把前面已经正确完成的 60% 全部推翻，而是形成一个清晰的 Delta → Re-plan → Continue 流程。

## 转化为命令

根据以上流程可以把这些 Prompt 转化为固定的几个命令：

/mctech-sdd-explore
/mctech-sdd-analyze
/mctech-sdd-decide
/mctech-sdd-spec
/mctech-sdd-plan
/mctech-sdd-implement
/mctech-sdd-change
/mctech-sdd-verify
/mctech-sdd-fix

详细的命令介绍参见 <commands/README.md>
