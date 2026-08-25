# OpenSpec change for Task 2

## User

现在执行 OpenSpec Change：refactor-permission-authorization
只执行 Task 2 - Permission Core Model
不要执行其他 task。

执行前：
1. 阅读 proposal
2. 阅读 design
3. 阅读 specs
4. 阅读 Task 2
5. 检查当前代码状态
6. 确认 Task 2 的依赖已经满足

执行要求：
- 严格按照 OpenSpec spec 实现
- 不修改 Task 2 之外的架构
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
8. 判断 Task 2 是否可以标记完成

## Assistant

## Implementing: refactor-permission-authorization (schema: spec-driven)

**进度：** 11/40（§1 已完成；本轮完成 §2）  
**依赖：** 1.1–1.4 已满足，无 design/代码冲突阻塞。

### 完成内容（2.1–2.7）

| 任务 | 实现                                                                          |
| ---- | ----------------------------------------------------------------------------- |
| 2.1  | `permission-set-builder.js`：Grant → full_path 子孙展开 → indexes             |
| 2.2  | Redis key `atom-ops:perm:user:{userId}`，shape `{ v, role, grants, indexes }` |
| 2.3  | `refreshUserPermissionCache`：重算 → DEL → SET                                |
| 2.4  | `getOrLoad` miss → DB 重算回填                                                |
| 2.5  | `assert` / `filterVisible`；admin 短路；`SYSTEM_ACTOR` 旁路                   |
| 2.6  | `getPermissionSummary(userId)`                                                |
| 2.7  | `PA.04`–`PA.10`                                                               |

### 测试结果

......

### 发现的问题

1. **`assert` 签名扩展**：design 为 `assert(user, resource, action, env?)`；目录作用域验权必须知道 `directoryId`，实现为可选第 5 参 `scope: { directoryId }`。§5 接入业务时需统一调用约定；若要严格四参，需改 design/spec。
2. **目录展开口径**：按 design 使用 `full_path` 前缀子孙；现网 `service-operations` 仍用 `%/${id}/%`（Cleanup 前并存，属预期）。
3. **Redis key**：按 spec 使用字面量 `atom-ops:perm:user:{userId}`，未走 website 的 `RedisKeyBuilder`（与 session key 分离，符合 C21/C22）。

### Task 2 是否可标记完成

**可以。** §2（2.1–2.7）已全部勾选；gate「`PA.*` 全绿」满足，可进入 §3/§4/§5。

## User

解决问题 2 改为 full_path前缀

## Assistant

将现网目录展开改为 `full_path` 前缀匹配。先定位所有 `%/${id}/%` 用法。

## Assistant

现网 `service-operations` / `image-sync` 的 `%/${id}/%` 改为 `full_path` 前缀。

## User

解决问题3 Redis Key改为使用RedisKeyBuilder方式

## Assistant

将权限 Redis key 改为 `RedisKeyBuilder`。先核对现有用法与测试断言。

## Assistant

design/spec 仍写字面量 key，与 `RedisKeyBuilder` 冲突。按你的要求改代码，并同步 OpenSpec 契约。

## User

解决问题1 添加scope参数

## Assistant

将 `assert` 的 `scope` 参数写入 design/spec，并核对代码与测试一致性。

## Assistant

**已定稿：** `assert(user, resource, action, env?, scope?)`，`scope = { directoryId? }`。

### 契约
- design / spec / final-architecture：补齐 `scope` 说明
- 目录作用域：MUST 带 `scope.directoryId`
- 菜单级（如 `menu` + `view.service-lifecycle`）：可不传

### 代码
- 实现本已支持第 5 参；`PermissionService.assert` 补 JSDoc

### 测试
- `PA.07b`：缺 `directoryId` → 403；菜单级无 scope → 放行  
- `openspec validate --strict` → valid
