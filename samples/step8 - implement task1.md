# OpenSpec Change for Task 1

## User

现在执行 OpenSpec Change：

refactor-permission-authorization

只执行：

Task 1 - Permission Core Model

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

不要执行 Task 2。

## Implementing: refactor-permission-authorization (schema: spec-driven)

**Progress:** 4/40 → Task 1 Foundation 完成。未执行 Task 2。

### 依赖确认
- 1.1 `depends-on: —`：可直接开做
- design / 现码：无冲突（旧 UDP 保留至 Cleanup）

### 测试结果

......

### 发现的问题
1. MySQL/TiDB 对 UNIQUE 中 `NULL` 可多行；`svc.manage` 唯一性靠应用层 `validateGrantEnvConstraint`（表约束对非 NULL env 仍有效）
2. 仅加载 permission 模型时，`createTestDb` 会打无害 SQLITE 索引错误日志（`.catch` 吞掉，不影响结果）

### Task 1 可标记完成？
**可以。** 1.1–1.4 已完成；可进入 §2 Infrastructure。

## User

问题 1 有什么解决方法

## Assistant

**现象：** `UNIQUE(user_id, directory_id, item_code, env_id)` 在 MySQL/TiDB 里，`env_id = NULL` 时不互斥，同一用户同一目录可插入多行 `svc.manage`。

**可选解法（按推荐序）：**

### 1. 生成列 + 唯一索引（推荐，不改业务语义）
保留 `env_id` 对 `svc.manage` 为 `NULL`（符合 design），再加一列把 NULL 归一成哨兵，只用于唯一约束：

```sql
env_id BIGINT NULL,
env_key BIGINT AS (IFNULL(env_id, 0)) STORED,
UNIQUE KEY uk_user_dir_item_env (user_id, directory_id, item_code, env_key)
```

- `svc.manage`：`env_id` 仍 NULL，`env_key=0`
- 环境类 Item：`env_id` 为真实环境 ID；需约定 **env_id 永不为 0**（或哨兵用 `-1`）

TiDB / MySQL 8+ 均支持 generated column。

### 2. 哨兵值替代 NULL（实现简单，改契约）
`svc.manage` 存 `env_id = -1`（或 0），不再用 NULL。  
**代价：** 与当前 design/spec「`env_id` MUST 为 NULL」冲突，要改 OpenSpec 后再做。

### 3. 仅应用层防重（现状）
`validateGrantEnvConstraint` + 写入前 `SELECT` / 事务内 upsert。  
**弱点：** 并发双写仍可能重复；不适合单独当唯一保障。

### 4. 部分唯一索引
仅对 `env_id IS NOT NULL` 建唯一；`manage` 另建  
`UNIQUE(user_id, directory_id, item_code) WHERE env_id IS NULL`。  
**TiDB/MySQL 不支持** PostgreSQL 式 partial unique，基本不可用。

---

**建议：** 走 **方案 1**，spec 仍写「manage ⇒ `env_id` IS NULL」，DB 层用 `env_key` 保证唯一。若要落地，需先改 `proposal/design/DDL`（属 Task 1 补强或单独小变更），不要在 Task 2 里静默改表结构。

## User

暂停往后执行，先解决发现的问题

## Assistant

暂停 Task 2。按方案 1（生成列 `env_key`）修 UNIQUE/NULL 问题，并同步 design / DDL / 测试。

## Assistant

已按方案 1 落地，**未进入 Task 2**。

### 改动
- DDL：`env_key BIGINT AS (IFNULL(env_id, -1)) STORED`，唯一键改为 `(user_id, directory_id, item_code, env_key)`
- `env_id` 语义不变：`svc.manage` 仍为 NULL
- 应用层：环境类 Item 要求 `envId` 为正整数（禁止 `-1`/`0`）
- SQLite 测试：`ifnull(env_id, -1)` 表达式唯一索引
- 同步：`design.md` / `spec.md` / `final-architecture.md`

### 测试
`6 passing`（含 PA.01c 重复 manage 拒绝）

可继续 Task 2 时再说。

## User

方案一不好，切换为方案二 "哨兵值替代 NULL"

## Assistant

切换为方案二：`svc.manage` 用哨兵 `env_id = -1`，去掉生成列。同步改 DDL、design/spec、校验与测试。

## Assistant

已切到方案二，去掉 `env_key`。

### 定稿
| 层          | 约定                                                                         |
| ----------- | ---------------------------------------------------------------------------- |
| DB          | `env_id BIGINT NOT NULL`；`UNIQUE(user_id, directory_id, item_code, env_id)` |
| manage      | 落库 `env_id = -1`（`NO_ENV_SENTINEL`）                                      |
| 环境类 Item | `env_id` 正整数                                                              |
| API         | manage 仍省略 `envId`；`normalizeEnvIdForStorage` / `envIdForApi` 互转       |

### 测试
`6 passing`；`openspec validate` 通过。

未执行 Task 2。

## User

解决问题2，模型上使用unique标记唯一索引

## Assistant

已处理。

### 改动
1. **模型** `options.uniques: [['userId', 'directoryId', 'itemCode', 'envId']]`（henhouse 唯一约束标记）
2. **`createTestDb`**：把 `uniques` 转成 Sequelize `indexes` 并 `sync`；去掉手写 Grant UNIQUE SQL
3. **遗留索引**：仅当 `services`/`users` 表已加载才建，消除 SQLITE 噪音

### 测试
`6 passing`，无 `SQLITE_ERROR` 日志
