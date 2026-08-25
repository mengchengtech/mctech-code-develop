# Permission architecture exploration

## User

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
2. 授权流程
   - 谁负责给用户/角色授权
   - 授权入口有哪些
   - 授权数据保存在哪里
   - 授权后如何刷新缓存
   - 是否存在多套授权逻辑
3. 验权流程
   - HTTP API 验权
   - Service 内部验权
   - RPC 调用
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

## Assistant

当前权限体系是「二元角色 + 目录级数据范围」，不是完整 RBAC/ACL。以下按你要求的 A–K 整理；标注 **[事实]** / **[推测]** / **待确认**。

---

## A. 当前权限架构概览

```plain
┌───────────── website ─────────────────┐     RPC(x-* headers)     ┌────────── server ──────────┐
│ session-auth (Cookie→Redis→ctx.user)  │ ────────────────────────► │ data-scope → ctx.user      │
│ page-role-guard (admin 页面/API 前缀) │                           │ role-guard (空操作)        │
│ openapi-signature (签名, 见问题)      │                           │ 业务 Service 目录过滤      │
│ 前端 sidebar 按 userRole 隐藏菜单     │                           │ AuthManagement CRUD 授权   │
└───────────────────────────────────────┘                           └───────────────────────────┘
```

**两层控制 [事实]**

| 层       | 机制                                | 作用                                                      |
| -------- | ----------------------------------- | --------------------------------------------------------- |
| 角色     | `users.role` ∈ `{admin, developer}` | 页面/菜单是否可进                                         |
| 数据范围 | `user_directory_permissions`        | developer 在服务运营/任务列表里能看到哪些目录子树下的服务 |

**不存在 [事实]**：独立权限项表、资源/操作 ACL、用户组表、permission code、角色-权限多对多。

---

## B. 权限数据模型

| 概念        | 本项目对应                   | 位置                                                                       | 说明                                                                 |
| ----------- | ---------------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| 用户        | `users` / `User`             | `docs/resources/db/schema.sql:42-59`；`server/.../org-management/index.js` | `login_name, username, role, company_id, status, is_deleted`；无密码 |
| 用户组/角色 | **仅字段 `role`**            | 同上；`VALID_ROLES` in `org-management/service.js:6`                       | 无 groups/roles 表；admin 本地账号 `userId=0`（登录时，非 DB 行）    |
| 权限/权限项 | **无独立实体**               | —                                                                          | “权限”= 目录授权记录                                                 |
| 资源        | `service_directories`        | schema `service_directories`；运营里当数据边界                             | 授权对象是目录节点                                                   |
| 操作        | **无 action 表**             | —                                                                          | 操作许可靠角色 + 页面守卫，非 action code                            |
| 菜单        | 前端硬编码                   | `website/.../sidebar-nav.vue:59-67`                                        | `role: 'admin' \| null`                                              |
| API 权限    | 路径前缀列表                 | `page-role-guard.js:8-14`                                                  | admin-only 前缀拦截                                                  |
| 数据权限    | `user_directory_permissions` | schema:466-476；model `auth-management/index.js`                           | `(user_id, directory_id)` 唯一                                       |

子孙目录展开口径 **[事实]**：授权目录 id + `fullPath LIKE '%/<id>/%'`（如 `service-operations/service.js:334-340`）。

---

## C. 授权流程

**谁授权 [事实]**

1. **UI**：管理员在「授权管理」页面对用户追加/移除目录（`auth-management` 页面 + store）。
2. **OpenAPI**：外部系统 HMAC 签名后调 `/openapi/v1/users/:id/permissions`（`server/.../openapi/controller.js:53-88`）。
3. **组织管理**：可改用户 `role`（`updateUser` 仍允许 `admin|developer`）；**新建**仅允许 `developer`（`CREATABLE_ROLES`）。

**入口 [事实]**

| 入口       | 路径                                                      |
| ---------- | --------------------------------------------------------- |
| 页面       | `/auth-management.html` → `/services/auth-management/...` |
| Server API | `GET/POST/DEL /auth-management/users/:userId/permissions` |
| OpenAPI    | `GET/PUT/POST/DEL /openapi/v1/users/:userId/permissions`  |
| Service    | `AuthManagementService.{get,set,add,remove}Permissions`   |

UI 只用 **add / remove**；`setPermissions`（全量替换）主要给 OpenAPI PUT。

**存储 [事实]**：TiDB/MySQL 表 `user_directory_permissions`，无 Redis 权限缓存。

**缓存刷新 [事实]**：目录授权改完即下次查库生效。Session 里的 `userRole` 在登录时写入 Redis，**改角色后需重新登录**才会变（`auth/controller.js` 写 session；`session-auth` 只读 Redis）。

**多套授权逻辑 [事实]**

- 目录授权：一套（`AuthManagementService`，OpenAPI 复用）。
- 角色：`users.role` + 本地 admin 配置（`admin.email/password`）。
- 历史产品线授权 `user_product_lines`：**已废弃**（openspec tasks 注明移除）。

---

## D. 验权流程

### 1) HTTP（website）

1. `session-auth`：白名单外必须 Cookie `session` → Redis → `ctx.user`；API 无 session → 401；页面 → 302 `/login.html`。  
   文件：`website/src/server/plugins/session-auth.js`
2. `page-role-guard`：非 admin 访问 `ADMIN_ONLY_PAGES` → 302 `/dashboard.html`。  
   含 `/services/org-management/`、`/services/auth-management/`、`/services/service-lifecycle/`、`/services/operation-logs/`、`/api/openapi/` 等。  
   **不含** `/services/service-operations/`、`/task-management`。

### 2) Service 内部

- **列表/搜索/任务列表**：`role !== 'admin'` 时用 `UserDirectoryPermission` 过滤（`service-operations/service.js`、`image-sync/service.js:#resolveAllowedServiceIds`）。
- **部署 / k8s / 镜像同步触发**：使用 `ctx.user.userId` 作操作人，**未见**对目标 service 的目录权限校验（`build-pipeline`、deploy/k8s 路由）。**[事实：无校验代码]**

### 3) RPC

`session-auth` 把用户放进 `$asynctx.webContext.extras` → `infra-discovery` 序列化为 `x-user-id`、`x-user-role` 等头 → server `data-scope` 再组装 `ctx.user`。  
框架：`infra-discovery/.../request-context.js:_addHeaders`；项目：`server/.../plugins/data-scope.js`。

### 4) 前端

- `sidebar-nav.vue`：按 `userRole` 过滤菜单。
- `page-layout.vue`：无 user → 跳登录。
- **无**按钮级 v-permission / permission code。

### 5) OpenAPI

设计为 HMAC（`openapi-signature.js`）+ 配置 `openapi.clients`；**与登录 session 独立**。实现状态见 J。

---

## E. 权限相关模块清单

| 模块                           | 路径                                       | 职责                          |
| ------------------------------ | ------------------------------------------ | ----------------------------- |
| auth                           | `website/.../modules/auth`                 | 登录/登出/me、LDAP、session   |
| session-auth / page-role-guard | `website/.../plugins/`                     | 会话鉴权、页面角色守卫        |
| openapi-signature              | `website/.../plugins/openapi-signature.js` | OpenAPI 签名（见问题）        |
| auth-management                | server + website                           | 目录授权 CRUD + UI            |
| org-management                 | server + website                           | 用户/公司/产品线；角色字段    |
| openapi                        | server + website                           | 外部用户与权限 API            |
| data-scope / role-guard        | `server/.../plugins/`                      | 注入 ctx.user；角色守卫空实现 |
| service-operations             | server                                     | 目录数据权限过滤              |
| image-sync                     | server                                     | 同步任务列表的目录过滤        |
| 前端 user/auth store           | `user-store.js`、`auth-management/stores`  | me / 授权 UI                  |

---

## F. 权限相关 API 清单

**认证**

- `POST /api/auth/login|logout`，`GET /api/auth/me`

**授权管理（经 website 代理）**

- `GET /services/auth-management/users`
- `GET|POST /services/auth-management/users/:userId/permissions`
- `DEL .../permissions/:directoryId`  
  → server `/auth-management/...`

**OpenAPI**

- `/openapi/v1/users` CRUD/disable/enable  
- `/openapi/v1/users/:userId/permissions` GET/PUT/POST/DEL  
- `/openapi/v1/companies`

**带数据权限的读接口（server，经 `/services/service-operations`）**

- `GET .../directories`、`/services`、`/services/search`、`/deploy/tasks`  
- image-sync：`listSyncTasks`（非 admin 过滤）

---

## G. 数据库 / 缓存 / 配置

**表 [事实]**

- `users`、`companies`、`product_lines`、`service_directories`、`user_directory_permissions`  
  DDL：`docs/resources/db/schema.sql`

**Redis [事实]**（`website/.../libs/redis-client.js`，前缀 `atom-ops`）

- `session/sid/<uuid>`：登录态  
- `login_fail/ip/<ip>`：登录限流  
- `openapi/nonce/<nonce>`：防重放  

**配置 [事实]**

- `website/.../individual.yml`：`admin`、`ldap`、`session.ttl`、`openapi.clients`、`redis`  
- `server/.../individual.yml`：datasource，无权限专用项  

**无 [事实]**：权限结果 Redis 缓存、permission 配置中心条目。

---

## H. 已发现的验权入口

| #   | 入口              | 文件                                              | 行为                                                                    |
| --- | ----------------- | ------------------------------------------------- | ----------------------------------------------------------------------- |
| 1   | session-auth      | `website/.../session-auth.js`                     | 登录态                                                                  |
| 2   | page-role-guard   | `website/.../page-role-guard.js`                  | admin 页面/API 前缀                                                     |
| 3   | sidebar 菜单过滤  | `sidebar-nav.vue`                                 | 前端隐藏                                                                |
| 4   | data-scope        | `server/.../data-scope.js`                        | 从头注入 `ctx.user`                                                     |
| 5   | role-guard        | `server/.../role-guard.js`                        | **恒 next()**                                                           |
| 6   | 目录过滤×4        | `service-operations/service.js`                   | listDirectories / listOnlineServices / searchServices / listDeployTasks |
| 7   | 目录过滤          | `image-sync/service.js:#resolveAllowedServiceIds` | listSyncTasks                                                           |
| 8   | openapi-signature | `openapi-signature.js`                            | 设计为签名验权（见 J）                                                  |
| 9   | 登录态校验        | `AuthController.me`                               | 无 ctx.user → 401                                                       |
| 10  | 角色白名单        | `createUser` / OpenAPI create                     | 强制/限制 developer                                                     |

**未发现**：PermissionService、`@RequirePermission`、统一 ACL middleware、deploy/k8s 的目录鉴权。

历史 `#ensureAdmin`：`server/src` **已无实现**；仅残留测试 `service-lifecycle/permission.test.js`（复制逻辑自测）。

---

## I. 模块依赖关系

```plain
[Browser]
   │ Cookie session
   ▼
website: session-auth → page-role-guard
   │ this.rpc + webContext.extras → x-user-*
   ▼
server: eureka parse-x-header(ctx.extras) + data-scope(ctx.user)
   │
   ├─ auth-management ──► UserDirectoryPermission / ServiceDirectory
   ├─ org-management ───► User / Company / ProductLine
   ├─ service-operations ► 读权限表做列表过滤；写操作多半不验目录
   ├─ image-sync ───────► 任务列表过滤
   └─ openapi ──────────► 复用 AuthManagementService + OpenApiUserService
```

前端约定：`userRole`（session/me）↔ DB `users.role`；菜单 `role:'admin'` ↔ page-role-guard 前缀列表（需人工对齐）。

---

## J. 问题与技术债（区分事实与推测）

1. **server `role-guard` 空实现** [事实]：始终 `next()`；测试仍按“会拦路径”写，与实现脱节（`role-guard.test.js` 用 `user.role`，中间件读 `user.userRole`）。
2. **admin 能力主要靠 website 页面守卫** [事实]：server 上 service-lifecycle / org-management **无** `#ensureAdmin`。直连 server 端口则 **待确认** 是否可绕过。
3. **数据权限只覆盖“列表类”** [事实]：deploy / k8s / sync 触发未见 `UserDirectoryPermission` 校验 → developer 若知 serviceId **可能**越权操作（**推测**需用实测验证）。
4. **`productLineIds` 残留** [事实]：`data-scope` 仍解析 `x-user-product-line-ids`；业务已改为目录授权；session 不再写该字段。
5. **openapi-signature 可疑** [事实]：  
   - `website/index.js` components **未包含** `openapi-signature`；  
   - 源码引用未定义的 `SIGNATURE_WHITELIST` / `WebError` / `AUTH_ERRORS`。  
   → 是否真正启用、签名是否生效 **待确认**。
6. **`/openapi/` 在 session 白名单** [事实]：`session-auth` WHITELIST 含 `/openapi/`，依赖签名插件；若签名未生效则风险高。
7. **E2E 与菜单不一致** [事实]：`role-based-access` 期望 developer 2 个导航项，但现有 `task-management` 为 `role:null`，应为 ≥3。
8. **permission 测试陈旧** [事实]：`SLP.*` 测的是已删除的 `#ensureAdmin` 副本，非生产代码。
9. **schema 注释过时** [事实]：`users.role` 注释仍写 `admin`，代码默认 `developer`。
10. **授权无审计（UI 路径）** [事实]：OpenAPI 写 operation_logs；`AuthManagementService` 自身不写审计。

---

## K. 尚未确认的信息

| 项                                                         | 原因                                                                                 |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| 生产是否启用 `openapi-signature` 组件                      | components 列表与 require 不一致                                                     |
| 直连 `atom-ops-server` 时 admin API 是否暴露               | 依赖网络隔离，代码层未拦                                                             |
| developer 对未授权 serviceId 调用 deploy/k8s 的实际结果    | 无对应鉴权代码，缺集成/e2e 证明                                                      |
| henhouse `ctx.user` vs `ctx.extras` 中间件顺序是否覆盖字段 | data-scope 重写 `ctx.user`；`username` 等是否完整传到 server 依赖顺序                |
| Redis key 是否严格等于 spec 的 `session:<uuid>`            | 代码用 `RedisKeyBuilder('atom-ops').build('session','sid',id)`，与文档字符串可能不同 |
| `permission.json`（import-jobs）与运行时权限关系           | 像迁移/导入映射，非鉴权引擎                                                          |
| 是否还有 http-proxy 注入 X-User（非 rpc）路径              | 业务多为 `this.rpc`；openspec 提过 http-proxy，本仓库 website 业务模块未见显式设头   |

---

### 小结（代码事实）

平台权限 = **`admin|developer` 角色门禁** + **`user_directory_permissions` 目录数据范围**；授权落库、验权分散在 middleware 与少数 Service 查询过滤中，**不是**统一 PermissionService / RBAC 权限码体系。

若要继续深入，可只盯一条线：例如「developer 调用 deploy/trigger 是否越权」做只读追踪或 spike（仍不改代码）。需要立项实现时，用 `/opsx:propose` 再落提案。

## User

把输出信息保存到permission-exploration目录下，按类别分别保存到不同的文件里

## Assistant

已写入 `permission-exploration/`，按 A–K 分文件：

| 文件                            | 内容         |
| ------------------------------- | ------------ |
| `README.md`                     | 索引与总览   |
| `A-architecture-overview.md`    | 架构概览     |
| `B-data-model.md`               | 数据模型     |
| `C-authorization-flow.md`       | 授权流程     |
| `D-verification-flow.md`        | 验权流程     |
| `E-module-inventory.md`         | 模块清单     |
| `F-api-inventory.md`            | API 清单     |
| `G-db-cache-config.md`          | DB/缓存/配置 |
| `H-verification-entrypoints.md` | 验权入口     |
| `I-module-dependencies.md`      | 模块依赖     |
| `J-tech-debt.md`                | 问题与技术债 |
| `K-unconfirmed.md`              | 待确认项     |
