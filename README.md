# 命令说明

项目开发环境中推荐 **9 个 `/mctech-sdd-*` 命令**：

## 最终推荐的命令速查表

以后实际上只需要记住下面这些：

| 命令                    | 职责         | 一句话记忆                 | 改代码 |
| ----------------------- | ------------ | -------------------------- | ------ |
| `/mctech-sdd-explore`   | 负责事实     | **现在是什么？**           | ❌      |
| `/mctech-sdd-analyze`   | 负责方案     | **问题是什么？怎么解决？** | ❌      |
| `/mctech-sdd-decide`    | 负责决策     | **最终选哪个？**           | ❌      |
| `/mctech-sdd-spec`      | 负责契约     | **最终必须是什么？**       | ❌      |
| `/mctech-sdd-plan`      | 负责拆解     | **怎么拆着做？**           | ❌      |
| `/mctech-sdd-implement` | 负责执行     | **按规范做一个 Task**      | ✅      |
| `/mctech-sdd-change`    | 负责需求变更 | **需求变了怎么办？**       | ❌      |
| `/mctech-sdd-verify`    | 负责验收     | **做出来符合规范吗？**     | 默认 ❌ |
| `/mctech-sdd-fix`       | 负责修复     | **验证结果修复实现问题**   | ✅      |

无论数据库重构、微服务拆分、前端大改、API 重构，都可以复用同一套 Agent 工作流。

它们对应：

```plain
Explore → Analyze → Decide → Spec → Plan → Implement → Verify
                        ↑            │
                        │            │
                        └── Change ──┘
```

下面这些 Prompt 可以直接作为你的 Agent 命令模板使用。

---

## 1\. `/mctech-sdd-explore`

### 作用

**只探索，不设计，不修改代码。**

适合：

- 接手老项目
- 大型重构前
- 不熟悉某个功能
- 需要建立 Current State
- 权限、订单、租户等跨模块功能分析

### 使用场景

例如：

用户输入：

> `/mctech-sdd-explore` 探索当前项目的权限授权和验权实现。

或者：

> `/mctech-sdd-explore` 探索订单创建流程，重点分析订单状态变化和权限检查。

---

## 2\. `/mctech-sdd-analyze`

### 作用

**基于 Exploration 分析问题和方案。**

不修改代码，不修改最终 Spec。

适合：

- 大型重构
- 多方案比较
- 技术方案设计
- 判断是否需要拆模块
- 分析兼容性、性能、安全性

### 使用场景

例如：

> `/mctech-sdd-analyze` 分析权限系统应该采用 Gateway 验权还是 Gateway + Service 双层验权。

或者：

> `/mctech-sdd-analyze` 分析当前 Redis 缓存重构有哪些方案。

---

## 3\. `/mctech-sdd-decide`

### 作用

**把架构师的决定正式记录下来。**

它不是让 AI 替架构师做决定。

适合：

- 方案已经讨论完成
- 架构师确定最终方案
- 需要把口头决策固化
- 防止后续 Agent 自己改变架构

### 使用场景

例如你确定：

```plain
Gateway 做 API 粗粒度验权，Service 做资源级验权，统一通过 Authorization SDK。
```

就执行：

> `/mctech-sdd-decide` Gateway 做 API 粗粒度验权，Service 做资源级验权，统一通过 Authorization SDK。

把这些决策交给 Agent 固化。

---

## 4\. `/mctech-sdd-spec`

### 作用

**把 Exploration + Decision 转换成 OpenSpec 的正式目标规范。**

这是 SDD 最核心的命令之一。

适合：

- 创建新的 OpenSpec Change
- 根据设计生成 Proposal / Design / Spec
- 将需求正式化
- 在 Apply 前建立目标状态

### 使用场景

例如：

```plain
/mctech-sdd-spec [spec name]
```

可显示指定 Spec 名称也可由 AI 自动生成

创建：

```plain
openspec/changes/refactor-permission/
```

这是**Apply 之前必须完成**的步骤。

---

## 5\. `/mctech-sdd-plan`

### 作用

**把 Spec 拆成安全、可执行、可验证的 Tasks。**

适合：

- 大型功能
- 跨模块重构
- 多 Agent 并行开发
- OpenSpec Change 已有 Spec
- 需要确定实施顺序

### 使用场景

例如：

```plain
/mctech-sdd-plan <spec name>
```

执行后会拆成类似下面的几组：

```plain
Task 1 Permission Model
Task 2 Authorization Context
Task 3 Authorization Engine
Task 4 Permission Cache
......
Task 7 Cleanup
```

---

## 6\. `/mctech-sdd-implement`

### 作用

**真正修改代码。**

它是唯一主要负责 Coding 的命令。

核心原则：

> **一次只执行一个 Task。**

适合：

- OpenSpec 已经完成
- Task 已明确
- 开始实际开发

### 使用场景

例如：

```plain
/mctech-sdd-implement task 3
```

执行：

> Task 3：Authorization Engine。

千万不要执行不带参数的 `/mctech-sdd-implement` 命令，然后让 Agent 自己把整个 tasks.md 全做完。这样一次使用的上下文特别大，AI 执行起来也很慢。中间过程有没有问题，有没有偏离完全是未知的

---

## 7\. `/mctech-sdd-change`

### 作用

处理：

> **OpenSpec 已经 Apply 一部分之后，需求发生变化。**

例如：

```plain
Task 1 ✅
Task 2 ✅
Task 3 🚧
Task 4 ⏳
```

这时候会发现：

> UI 要调整，而且授权逻辑也发生变化。

不要直接修改 Task 3。

使用：

```plain
/mctech-sdd-change <变更内容>
```

### 使用场景

```plain
Apply
 ↓
完成几个 Task
 ↓
需求变化
 ↓
/mctech-sdd-change
 ↓
更新 Spec
 ↓
重新 Plan
 ↓
/mctech-sdd-implement
```

---

## 8\. `/mctech-sdd-verify`

### 作用

验证代码是不是符合 Spec，而不是单纯检查“代码能不能运行”。

适合：

- Task 完成后
- 阶段性集成
- 全部开发完成
- 发布前
- Archive 前

### 使用场景

```plain
/mctech-sdd-verify --task 3
```

也可以：

```plain
/mctech-sdd-verify --task 3-6
```

## 9\. `/mctech-sdd-fix`

### 使用场景

#### 1\. 直接修改 `/mctech-sdd-verify` 发现的错误

比如现在执行 `/mctech-sdd-verify`后得到下面的错语信息：

```plain
FAIL
org-management 换公司未校验 grants

Classification:
IMPLEMENTATION_BUG

Action:
FIX_REQUIRED
```

可以直接调用 `/mctech-sdd-fix`

```plain
Agent执行流程：

读取 Verify Report
       ↓
找到 Task3
       ↓
读取 Spec
       ↓
检查代码
       ↓
修改代码
       ↓
补测试
       ↓
执行测试
```

然后再次执行 `/mctech-sdd-verify`

如果 PASS 继续下一个 Task。

#### 2\. 如果 `/mctech-sdd-fix` 发现其实不是实现问题

这也是这个命令最有价值的地方。

例如 `/mctech-sdd-verify` 返回下面的错误：

```plain
FAIL
Expected: 403
Actual: 401
```

`/mctech-sdd-fix` 分析后发现:

```plain
Spec:
内部 API 应返回 401
测试要求 403
```

那么它**不能修改代码**。应该返回：

```plain
AI_CHANGE_REQUIRED = false
DECISION_REQUIRED = false
Classification: TEST_DEFECT
Recommendation: 修改测试
```

如果发现以下信息：

```plain
Spec: 管理员只能通过 ROLE_ADMIN 修改用户
New Requirement: 拥有 USER_MANAGER 也可以修改用户
```

那么会返回：

```plain
Classification: NEW_REQUIREMENT
Next Action: AI_CHANGE_REQUIRED
```

然后再执行 `/mctech-sdd-change` 走变更流程

#### 3\. 修改外部测出的bug

如果bug只是正软的软件bug，不是功能变更。可直接用 `/mctech-sdd-fix` 查找原因修复

`/mctech-sdd-fix` 命令后可添加对bug的简单描述

```plain
/mctech-sdd-fix Bug: 用户切换公司后仍然可以访问原公司的订单。

严重程度：高

复现步骤：
1. 用户登录公司 A
2. 切换到公司 B
3. 请求公司 A 的订单 API
4. 返回订单数据

期望：
切换公司后不能访问公司 A 的数据。

实际：
仍然可以访问。
```

严重程度 ...... 后面的内容可写可不写。要是bug简单描述能说清楚，就不用写。如果有复杂的触发机制，需要写清楚复现步骤。期望，实际的信息

#### 处理失败的测试

`/mctech-sdd-fix` 的定位是 **Verify → Remediation**，而不是简单的 Fix Code。

它可以处理以下的事件：

```plain
IMPLEMENTATION_BUG
TEST_DEFECT
CONTRACT_DRIFT
CONFIGURATION_DEFECT
```

但不能处理这些事件：

```
SPEC_DEFECT
DESIGN_CHANGE
ARCHITECTURE_PROBLEM
NEW_REQUIREMENT
```

---

## 推荐的生产工作流

现在把 9 个命令串起来：

```plain
          新功能 / 大重构
               │
               ▼
           /mctech-sdd-explore
               │
               ▼
           /mctech-sdd-analyze
               │
               ▼
           /mctech-sdd-decide
               │
               ▼
           /mctech-sdd-spec
               │
               ▼
           /mctech-sdd-plan
               │
               ▼
    ┌─────────────────────┐
    │ /mctech-sdd-implement Task1 │
    │ /mctech-sdd-verify Task1    │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │ /mctech-sdd-implement Task2 │
    │ /mctech-sdd-verify Task2    │
    └──────────┬──────────┘
               │
               ▼
              ...
        ┌──────────────┐
        │  /mctech-sdd-verify  │
        └──────┬───────┘
               │
               ├──────────────────────────┐
               │                          │
       ┌───────┴────────┐                 │
       │                │                 │
       ▼                ▼                 │
  实现有问题          Spec/需求变化          │
       │                │                 │
       ▼                ▼                 │
   /mctech-sdd-fix          /mctech-sdd-change            │
       │                │                 │
       │                ▼                 │
       │            /mctech-sdd-plan              │
       │                │                 │
       │                ▼                 │
       │            /mctech-sdd-implement         │
       │                │                 │
       └────────┬───────┘                 │
                │                         │
                ▼                         │
                │                         │
                ▼                         │
           /mctech-sdd-verify ────────────────────┘
                │
                ▼
            Final Verify
                │
                ▼
             Archive
```

---

## 小需求不要走完整流程

这套流程不能变成“任何改一行代码都要 Explore”。

我建议生产环境用一个非常简单的判断：

| 变化                     | 使用命令                              |
| ------------------------ | ------------------------------------- |
| 不知道代码在哪里         | `/mctech-sdd-explore`                 |
| 不知道怎么设计           | `/mctech-sdd-analyze`                 |
| 有多个方案需要架构决策   | `/mctech-sdd-decide`                  |
| 新功能/重大需求          | `/mctech-sdd-spec`                    |
| Spec 已确定，需要拆任务  | `/mctech-sdd-plan`                    |
| 已确定怎么做，开始编码   | `/mctech-sdd-implement`               |
| 已实施部分，需求发生变化 | `/mctech-sdd-change`                  |
| 检查是否符合 Spec        | `/mctech-sdd-verify`                  |
| 简单 Bug 修复            | 通常直接 Coding，不必走完整 SDD       |
| 简单 UI 文案/样式        | 通常直接 Coding                       |
| 小范围业务逻辑修改       | 根据影响决定是否 `/mctech-sdd-change` |

---

## 最重要的几个“禁止事项”

- Explore：不能改代码。
- Analyze：不能把推荐方案当成最终方案。
- Decide：不能替架构师做最终决策。
- Spec：不能偷偷改变已确认的架构决策。
- Plan：不能通过拆 Task 改变业务需求。
- Implement：不能自行扩大任务范围。
- Change：不能直接修改代码，必须先分析影响。
- Verify：不能以“测试通过”代替“符合 Spec”。

## 与 OpenSpec 命令之间的关系

OpenSpec 标准命令负责“规范驱动开发流程”，/mctech-sdd-* 负责把这个流程进一步拆成适合 AI Agent 执行的细粒度工作流。

| OpenSpec 标准命令 | `/mctech-sdd-*`                                                                           | 关系                                                    |
| ----------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `propose`         | `/mctech-sdd-explore` + `/mctech-sdd-analyze` + `/mctech-sdd-decide` + `/mctech-sdd-spec` | `/mctech-sdd-*` 把 propose 前后的分析、决策、规范化拆开 |
| `explore`         | `/mctech-sdd-explore`                                                                     | 基本对应                                                |
| `apply`           | `/mctech-sdd-implement`                                                                   | 核心对应                                                |
| `verify`          | `/mctech-sdd-verify`                                                                      | 基本对应                                                |
| `archive`         | 暂无专门命令                                                                              | OpenSpec 生命周期管理                                   |
| —                 | `/mctech-sdd-plan`                                                                        | 对应 OpenSpec tasks 的细化                              |
| —                 | `/mctech-sdd-fix`                                                                         | Verify 后的缺陷修复闭环                                 |
| —                 | `/mctech-sdd-change`                                                                      | 实施过程中的需求/Spec 变更管理                          |

> OpenSpec 关注的是：Change 的生命周期。
> /mctech-sdd-* 关注的是：AI Agent 每一步应该思考什么、能修改什么、什么时候必须停下来。

它们不是竞争关系，而是上下层关系，就好像基础设施和上层应用的关系。OpenSpec 可作为底层规范和事实来源，/mctech-sdd-* 作为团队的 Agent 操作层。
