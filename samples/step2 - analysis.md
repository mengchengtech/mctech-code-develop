# Permission validation analysis

## User

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

## Assistant

按「权限验权入口完整性」做只读扫描：先对齐探索技能与既有上下文，再按你列的 16 类定位实际执行权限判断的位置。

## Assistant

## 验权入口完整性清单

系统验权分三层：**登录态**（session）→ **角色**（`userRole=admin|developer`）→ **目录数据范围**（`user_directory_permissions`）。无 permission code / ACL 注解体系。

---

### 1. HTTP Controller / API

| 模块                                 | 文件                                          | 类/函数                                                               | 验权方式                                         | 权限标识                      | 权限数据来源           | 用户身份来源                   | 缓存               | 绕过路径                                                |
| ------------------------------------ | --------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------ | ----------------------------- | ---------------------- | ------------------------------ | ------------------ | ------------------------------------------------------- |
| auth                                 | `website/.../auth/controller.js`              | `AuthController.me`                                                   | 无 `ctx.user` → 401                              | 登录态                        | Redis session          | Cookie→中间件注入              | Redis session      | 无（依赖中间件先跑）                                    |
| auth                                 | 同上                                          | `AuthController.login`                                                | LDAP/本地密码；`status!==active`→403；未注册→403 | 账号状态 `active`             | DB `users` + LDAP      | 请求 body `loginName`          | 登录失败计数 Redis | admin 账号走配置密码，不查 DB                           |
| service-operations                   | `server/.../service-operations/controller.js` | `listDirectories` / `listServices` / `searchServices` / `deployTasks` | 透传 `ctx.user` 到 Service 做目录过滤            | `userRole` + `userId`         | 见 §6                  | `ctx.user`（data-scope）       | 否                 | 写操作接口无目录断言（见下）                            |
| service-operations                   | 同上                                          | `triggerDeploy` / `syncImage` / k8s 运维等                            | **无资源验权**，仅取 `operatorId`                | —                             | —                      | `ctx.user.userId`              | 否                 | **知悉 serviceId 可越权写**                             |
| image-sync                           | `server/.../image-sync/controller.js`         | `listSyncTasks`                                                       | 透传 role/userId 过滤                            | 同目录授权                    | 见 §6                  | `ctx.user`                     | 否                 | `getSyncTaskLog` / `stopSyncTask` **无目录过滤**        |
| openapi                              | `server/.../openapi/controller.js`            | 全部 `/openapi/v1/*`                                                  | Controller **自身无角色验权**                    | AccessKey（期望由签名中间件） | 配置 `openapi.clients` | `ctx.authInfo`（若中间件生效） | nonce Redis        | website 白名单放行 `/openapi/`；依赖签名中间件          |
| org / auth-management / lifecycle 等 | server Controllers                            | CRUD                                                                  | **无 `#ensureAdmin`**                            | —                             | —                      | 仅依赖 website 页守卫          | —                  | **直连 server 可绕过 admin 页面守卫**（待网络隔离确认） |
| org                                  | `server/.../org-management/controller.js`     | `createUser`/`updateUser` body schema                                 | `role` enum 仅 `developer`                       | 角色白名单                    | 请求 schema            | —                              | 否                 | OpenAPI/`createUser` 服务层另强制 developer             |

---

### 2. Gateway Filter

**未发现**独立 API Gateway Filter。验权在 Koa henhouse 插件链完成。

---

### 3. Middleware（实际执行）

| 模块    | 文件                                       | 类/函数                      | 验权方式                                       | 权限标识              | 权限数据来源           | 用户身份来源                                                               | 缓存                      | 绕过路径                                                                                                                   |
| ------- | ------------------------------------------ | ---------------------------- | ---------------------------------------------- | --------------------- | ---------------------- | -------------------------------------------------------------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| website | `website/.../plugins/session-auth.js`      | `sessionAuthMiddleware`      | Cookie→Redis；无 session→401/302               | 登录态                | Redis `session:sid:*`  | Cookie `session`                                                           | **是** Redis              | `WHITELIST`：`/api/auth/login\|logout`、`/static/`、`/assets/`、`/login`、`/openapi/`、`/openapi-spec.json`、`/libraries/` |
| website | `website/.../plugins/page-role-guard.js`   | `pageRoleGuardMiddleware`    | 非 admin 且 path∈`ADMIN_ONLY_PAGES`→302        | `userRole==='admin'`  | session 内 `userRole`  | `ctx.user`                                                                 | 否（角色在 session）      | **不拦** `/services/service-operations/`、`/task-management*`；无 user 时 `next()`                                         |
| website | `website/.../plugins/openapi-signature.js` | `openapiSignatureMiddleware` | HMAC-SHA1（`@mctech/api-authenticate`）+ nonce | `accessId` / OPS 签名 | 配置 `openapi.clients` | Header/Query 签名                                                          | nonce Redis；clients 内存 | 已注册于 `index.js`；源码引用未定义 `WebError`/`AUTH_ERRORS`，nonce 失败被吞并**降级放行**；非 `/openapi/v1/` 跳过         |
| server  | `server/.../plugins/data-scope.js`         | `dataScopeMiddleware`        | **不拒绝**，只组装 `ctx.user`                  | —                     | 请求头                 | `x-user-id` / `x-user-role`（默认 `developer`）/ `x-user-product-line-ids` | 否                        | **可伪造头**（若直连 server）；`productLineIds` 已无业务消费                                                               |
| server  | `server/.../plugins/role-guard.js`         | `roleGuardMiddleware`        | **恒 `next()`**                                | 名义 `userRole`       | —                      | `ctx.user`                                                                 | 否                        | **完全空实现，等同无守卫**                                                                                                 |
| website | `root-redirect.js`                         | —                            | 非验权                                         | —                     | —                      | —                                                                          | —                         | —                                                                                                                          |

注册：website `henhouse.js` 顺序 root-redirect → openapi-signature → session-auth → page-role-guard；`index.js` components 含上述四项。

---

### 4. Interceptor

**未发现** HTTP/RPC Interceptor 类验权。

---

### 5. Annotation / Decorator

| 发现                                  | 说明                        |
| ------------------------------------- | --------------------------- |
| `@Controller` / `@route` / `@Service` | 仅路由/DI，**无权限元数据** |
| `@RequirePermission` 等               | **不存在**                  |

---

### 6. Service 方法（真正做权限判断）

| 模块               | 文件                                | 类/函数                                       | 验权方式                                   | 权限标识           | 权限数据来源                                                      | 用户身份来源    | 缓存               | 绕过路径                                             |
| ------------------ | ----------------------------------- | --------------------------------------------- | ------------------------------------------ | ------------------ | ----------------------------------------------------------------- | --------------- | ------------------ | ---------------------------------------------------- |
| service-operations | `.../service-operations/service.js` | `listDirectories`                             | `role!=='admin'` → UDP 目录+子孙过滤       | `userId` + 目录 ID | DB `user_directory_permissions` + `service_directories.full_path` | 调用参数        | **否**（每次查库） | 仅列表；写路径不调用                                 |
| 同上               | 同上                                | `listOnlineServices`                          | 同上，过滤 `services.directoryId`          | 同上               | 同上                                                              | 同上            | 否                 | 同上                                                 |
| 同上               | 同上                                | `searchServices`                              | 同上                                       | 同上               | 同上                                                              | 同上            | 否                 | 同上                                                 |
| 同上               | 同上                                | `listDeployTasks`                             | 同上，限制 `jenkins_executions.service_id` | 同上               | 同上                                                              | 同上            | 否                 | 停止/触发部署无此过滤                                |
| image-sync         | `.../image-sync/service.js`         | `#resolveAllowedServiceIds` + `listSyncTasks` | 非 admin 限制 `service_id IN (...)`        | 同上               | 同上                                                              | 同上            | 否                 | `triggerSync`/`stopSyncTask`/`getSyncTaskLog` 未调用 |
| org-management     | `.../org-management/service.js`     | `createUser`                                  | `CREATABLE_ROLES` 白名单                   | role∈可创建集合    | 代码常量                                                          | 请求参数        | 否                 | UI/schema 再限 `developer`                           |
| openapi-user       | `.../openapi-user-service.js`       | `create`                                      | 强制 `role:'developer'`                    | 硬编码             | —                                                                 | OpenAPI 调用方  | 否                 | 非调用方身份验权，是数据约束                         |
| auth-management    | `.../auth-management/service.js`    | CRUD 授权                                     | **管理授权数据，不验调用方权限**           | directoryIds       | DB                                                                | 路径参数 userId | 否                 | 依赖 page-role-guard / OpenAPI 签名                  |

---

### 7. RPC Client / Server

| 模块                      | 位置                                                         | 行为                             | 是否验权             |
| ------------------------- | ------------------------------------------------------------ | -------------------------------- | -------------------- |
| website→server            | `$asynctx.webContext.extras` + infra-discovery `_addHeaders` | `userId`/`userRole` → `x-user-*` | **传递身份，不校验** |
| server                    | `data-scope`                                                 | 读头写入 `ctx.user`              | **信任调用方头**     |
| website Proxy Controllers | `*/controller.js` rpc.*                                      | 纯转发                           | 依赖 website 中间件  |

绕过：绕过 website 直连 server + 伪造 `x-user-role: admin`。

---

### 8. MQ Producer / Consumer

**未发现** MQ 及其中的权限判断。

---

### 9. Scheduled Job

| 模块              | 文件                   | 行为            | 验权           |
| ----------------- | ---------------------- | --------------- | -------------- |
| image-sync        | `setInterval` 状态同步 | 系统轮询 Harbor | **无用户验权** |
| build-pipeline    | 状态同步 timer         | 系统轮询        | **无**         |
| history-retention | 清理 timer             | 系统清理        | **无**         |

---

### 10. WebSocket

**无 WebSocket。** 存在 SSE（`deploy/sse`、`sync-tasks/:id/sse`）：仅走 session（website）+ 存在性校验，**无目录权限断言**。

---

### 11. GraphQL

**不存在。**

---

### 12. 文件/对象存储访问

**无**应用内 OSS/S3 ACL 验权。镜像走 Harbor API，无用户级权限门闸。

---

### 13. 数据库查询层

| 位置                                        | 方式                          | 说明                         |
| ------------------------------------------- | ----------------------------- | ---------------------------- |
| Sequelize `UserDirectoryPermission.findAll` | Service 内显式查询            | **非** DB RLS / 行级安全策略 |
| 裸 SQL JOIN `user_directory_permissions`    | auth-management / openapi get | 读授权数据，非调用方验权     |

无统一 DAO 拦截器。

---

### 14. 前端路由

| 模块 | 文件              | 类/函数     | 验权方式                     | 权限标识 | 来源               | 缓存       | 绕过                                          |
| ---- | ----------------- | ----------- | ---------------------------- | -------- | ------------------ | ---------- | --------------------------------------------- |
| 布局 | `page-layout.vue` | `onMounted` | 无 user → `/login.html`      | 登录态   | `GET /api/auth/me` | Pinia 内存 | 直输 URL；**服务端 page-role-guard 才是硬拦** |
| —    | —                 | —           | **无 Vue Router beforeEach** | —        | 多页 `.html` 入口  | —          | 每页独立加载                                  |

---

### 15. 前端按钮/组件权限

| 模块 | 文件              | 类/函数               | 验权方式                                       | 权限标识       | 来源                             | 缓存  | 绕过                             |
| ---- | ----------------- | --------------------- | ---------------------------------------------- | -------------- | -------------------------------- | ----- | -------------------------------- |
| 菜单 | `sidebar-nav.vue` | `visibleNavItems`     | `item.role===null` 或 `===userRole`            | `admin` / null | `userStore.currentUser.userRole` | Pinia | **仅 UI 隐藏**；URL 直访靠服务端 |
| 菜单 | 同上              | `visibleOpenApiEntry` | `userRole==='admin'`                           | admin          | 同上                             | 同上  | 同上                             |
| —    | —                 | —                     | **无** `v-permission` / 按钮级 permission code | —              | —                                | —     | 页面内按钮对已进页用户全开       |

---

### 16. 其他自定义权限判断

| 入口               | 文件                               | 方式                  | 标识     | 来源     | 身份      | 缓存  | 绕过                                               |
| ------------------ | ---------------------------------- | --------------------- | -------- | -------- | --------- | ----- | -------------------------------------------------- |
| 登录失败限流       | `redis-client.recordLoginFailure`  | IP 计数 >5 拦         | IP       | Redis    | `ctx.ip`  | Redis | 换 IP                                              |
| LDAP 认证          | `ldap-client.authenticateWithLdap` | 密码校验              | 账号     | LDAP     | loginName | 否    | admin 旁路 LDAP                                    |
| 账号禁用           | `AuthController.login`             | `status!=='active'`   | status   | DB users | loginName | 否    | 已有 session 不重查 status（session 过期前仍有效） |
| OpenAPI 改公司约束 | `OpenApiUserService.update`        | 有 UDP 则禁改 company | 权限条数 | UDP 表   | —         | 否    | 业务完整性约束，非调用方验权                       |

---

## 结构总览

```plain
Browser
  │
  ▼
website Koa
  ├─ openapi-signature  (/openapi/v1/* HMAC；nonce 降级风险)
  ├─ session-auth       (Cookie→Redis；白名单含 /openapi/)
  └─ page-role-guard    (admin-only 路径前缀)
        │
        ▼ RPC x-user-*
server Koa
  ├─ data-scope         (信任头 → ctx.user；不拒绝)
  └─ role-guard         (空实现)
        │
        ▼
Service 层目录过滤（仅 list*）
  UDP → 子孙目录 → serviceId 过滤
```

---

## 扫描结论（缺口）

**实际执行拒绝/过滤的入口共约 12 处有效点**（session、page-role、openapi 签名、login 状态、5 个 list 过滤、前端菜单/登录跳转）。

**明确无验权或空壳：** server `role-guard`；大多数写操作 Controller；直连 server；MQ/WS/GraphQL/存储/注解/拦截器；按钮级权限。

相关基线文档：`permission-exploration/exploration/H-verification-entrypoints.md`、`D-verification-flow.md`（本次复核：`openapi-signature` **已**列入 `website/src/server/index.js` components，与 J 文档旧结论不一致）。

## User

把输出保存到permission-exploration/analysis目录下

## Assistant

已保存至 `permission-exploration/analysis/verification-entrypoints-integrity.md`。
