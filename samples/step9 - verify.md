# Permission authorization refactor

## User

/mctech-sdd-verify refactor-permission-authorization --task 1,2,3

## Assistant

先读取该 OpenSpec 变更的 spec、design、tasks，并对照任务 1–3 的实现做验收。

## Assistant

AI Verify: `refactor-permission-authorization` — Tasks 1, 2, 3

**验收范围**：Foundation (§1) + Infrastructure (§2) + Business Auth/OpenAPI/UI (§3)  
**测试执行**：`npm test -- --grep "PA\.|AM\.|OA\.P"` → **27 passing**；`openspec validate --strict` → **valid**

---

## 1. Spec Compliance

| 任务                     | 结论     | 说明                                                                                                                    |
| ------------------------ | -------- | ----------------------------------------------------------------------------------------------------------------------- |
| §1 Foundation            | **PASS** | 模型、映射、校验、PA.01–PA.03 均符合 spec                                                                               |
| §2 Infrastructure        | **PASS** | 重算、Redis key/shape、C14/C24、Checker、summary、PA.04–PA.10 均符合                                                    |
| §3.1–3.5 Auth/OpenAPI/UI | **PASS** | grants 契约、缓存刷新钩子、公司校验、矩阵 UI、失败保持对话框                                                            |
| §3.6 OpenAPI 文档面      | **RISK** | endpoint 定义已对齐 grants；错误码表仍残留旧 `directoryIds` 条目                                                        |
| Spec「不再读写 UDP」     | **RISK** | AuthManagement 写路径已切 grants；`auth-management/index.js` 仍注册旧 `UserDirectoryPermission` 模型（§7 Cleanup 范围） |

### 关键实现对照

**§1 数据模型** — `user_directory_permission_grants` + 哨兵 `-1`：

```9:21:server/src/server/modules/permission/index.js
        name: 'UserDirectoryPermissionGrant',
        database: 'atom_ops',
        attributes: {
          id: { type: 'INTEGER', primaryKey: true, autoIncrement: true },
          userId: { type: 'INTEGER', allowNull: false, field: 'user_id' },
          directoryId: { type: 'INTEGER', allowNull: false, field: 'directory_id' },
          itemCode: { type: 'STRING', allowNull: false, field: 'item_code' },
          envId: { type: 'INTEGER', allowNull: false, field: 'env_id' },
```

**§1.3 映射** — restore/purge 不进 Item；任一 Item 含 `list.visible`：**PASS**

**§2 缓存 C14** — 重算 → DEL → SET：**PASS**（`permission-cache.js:101-110`）

**§3 grants 契约** — 拒绝旧 `directoryIds[]`、返回 Grant 列表：**PASS**

---

## 2. Implementation Completeness

| 子任务                                      | 状态            |
| ------------------------------------------- | --------------- |
| 1.1 DDL + Sequelize 模型                    | ✅               |
| 1.2 Permission 模块骨架                     | ✅               |
| 1.3 Item→(R,A) 映射                         | ✅               |
| 1.4 PA.01–PA.03                             | ✅               |
| 2.1 权限集合重算                            | ✅               |
| 2.2 Redis 读写                              | ✅               |
| 2.3 `refreshUserPermissionCache`            | ✅               |
| 2.4 miss 回填                               | ✅               |
| 2.5 assert / filterVisible / admin / system | ✅               |
| 2.6 `getPermissionSummary`                  | ✅               |
| 2.7 PA.04–PA.10                             | ✅               |
| 3.1 AuthManagement + OpenAPI grants         | ✅               |
| 3.2 事务后刷新缓存                          | ✅（调用链存在） |
| 3.3 公司变更校验                            | ✅               |
| 3.4 矩阵 UI                                 | ✅               |
| 3.5 保存失败保持对话框                      | ✅               |
| 3.6 website OpenAPI 代理/文档               | ⚠️ 部分          |
| 3.7 AM/OA 单测 + AM.E01 e2e                 | ✅               |

### 发现问题

#### RISK-1：OpenAPI 文档错误码未完全对齐 grants 契约
- **文件**：`website/src/web-content/pages/openapi-docs/views/main-view.vue` L100–107
- **违反**：Task 3.6 / `openapi` spec「拒绝旧 directoryIds」
- **影响**：文档仍展示 `DIRECTORY_IDS_MUST_BE_ARRAY`、`DIRECTORY_IDS_NOT_ALL_EXIST`，与实现不符
- **修复**：替换为 `DIRECTORY_IDS_CONTRACT_REMOVED`、`GRANTS_MUST_BE_ARRAY`、`MANAGE_MUST_NOT_HAVE_ENV` 等

#### RISK-2：`directoryIds` + `grants` 混合请求未拒绝
- **文件**：`auth-management/service.js` `rejectDirectoryIdsContract`
- **逻辑**：仅当 `hasDirectoryIds && !hasGrants` 时拒绝；同时携带两者时静默忽略 `directoryIds`
- **影响**：契约歧义，可能被误用
- **修复**：`hasDirectoryIds` 即拒绝（无论是否含 grants）

#### RISK-3：缓存刷新依赖 DI，无静默失败保护测试
- **文件**：`auth-management/service.js` `#refreshCache`
- **逻辑**：`#permSvc?.refreshUserPermissionCache` 可选链；服务未注入时跳过刷新
- **影响**：配置错误时授权写入成功但缓存不更新
- **修复**：生产路径 `#permSvc` 必须存在；缺注入应抛错；补集成测试（AuthManagement + 真实 PermissionService + Redis stub）

#### RISK-4：旧 UDP 模型残留（已知，§7 范围）
- **文件**：`server/src/server/modules/auth-management/index.js` L11–21
- **影响**：模型仍注册，与 spec「不再读写 UDP」字面不完全一致；当前 service 层未使用
- **修复**：§7.1 删除

#### NOT VERIFIED：禁用用户时权限缓存清理
- **文件**：`openapi-user-service.js` `disable()`
- **违反**：`openapi` spec「禁用用户 → 刷新或清除权限缓存」
- **影响**：禁用后 Redis 仍可能保留旧权限集合（§3 未列此子任务，但 OpenAPI spec 已定义）
- **修复**：`disable` 后调用 `refreshUserPermissionCache` 或 `cache.del`

---

## 3. Test Coverage

| 用例组        | 状态                             | 覆盖                                                                 |
| ------------- | -------------------------------- | -------------------------------------------------------------------- |
| PA.01–PA.10   | **PASS**                         | env 约束、映射、唯一键、重算、C14/C24、assert、admin/system、summary |
| AM.01–AM.06   | **PASS**                         | 保存、拒绝旧契约、env 校验、缓存刷新钩子、幂等/404                   |
| OA.P01–OA.P05 | **PASS**                         | 查询/替换/拒绝旧契约、用户详情 grants、公司校验                      |
| OPC.1–OPC.6   | **PASS**（未在 grep 范围，存在） | Controller 委托 + 审计                                               |
| AM.E01        | **PASS**                         | 矩阵结构、manage 仅无环境列                                          |
| AMP.5         | **PASS**                         | 保存失败对话框保持                                                   |

### 缺口

| 场景                                                         | 状态                                      |
| ------------------------------------------------------------ | ----------------------------------------- |
| AuthManagement → PermissionService 端到端缓存刷新（非 mock） | **NOT VERIFIED**                          |
| 矩阵保存成功 e2e（勾选 → 保存 → 列表更新）                   | **NOT VERIFIED**                          |
| 无效 `item` code 保存                                        | **NOT VERIFIED**（校验逻辑存在，无单测）  |
| OpenAPI DELETE grant 404 集成                                | **NOT VERIFIED**（AM.05 覆盖 service 层） |

---

## 4. Compatibility

| 项                                      | 结论                                                                                                    |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| 新表 `user_directory_permission_grants` | **PASS** — DDL 与模型一致                                                                               |
| API grants 契约                         | **PASS** — AuthManagement + OpenAPI 已切换                                                              |
| 旧 `directoryIds[]`                     | **PASS** — 显式拒绝                                                                                     |
| 旧 UDP 业务过滤                         | **RISK** — `service-operations` / `image-sync` 仍读 `user_directory_permissions`（§5 范围，非本次任务） |
| 双写/兼容层                             | **PASS** — 无（符合 design D5）                                                                         |

---

## 5. Security / Risk

| 项                                           | 结论                                          |
| -------------------------------------------- | --------------------------------------------- |
| 授权写路径校验（manage/env/目录/环境存在性） | **PASS**                                      |
| admin 短路 / system 旁路                     | **PASS** — 集中定义于 `permission-checker.js` |
| restore/purge 仅 admin                       | **PASS** — 映射排除 + Checker admin-only      |
| 权限绕过（§1–3 范围）                        | **PASS** — 授权写路径无已知绕过               |
| Website 直读 perm Redis key                  | **PASS** — §3 未涉及，website 仅 RPC 代理     |
| 禁用用户缓存残留                             | **RISK** — 见 RISK-3/NOT VERIFIED             |

---

## 6. Remaining Issues

### FAIL
无阻断性 FAIL（§1–3 核心功能已实现且测试通过）。

### RISK（建议修复后再合入 §4）
1. OpenAPI 文档错误码表残留 `directoryIds` 相关条目（3.6）
2. 混合 `directoryIds` + `grants` 请求未严格拒绝
3. 缺少 AuthManagement ↔ PermissionService 集成测试（真实缓存刷新）
4. OpenAPI 禁用用户未清权限缓存（openapi spec，超出 tasks 字面范围）

### 预期残留（§5/§7，非本次验收失败）
- 业务 Service 仍用旧 UDP 过滤（`service-operations/service.js`、`image-sync/service.js`）
- `auth-management/index.js` 旧 UDP 模型定义
- §4 `/me` summary 透传、§4 菜单粗拦未实现

---

## 7. Release Recommendation

**结论：§1–3 可有条件通过（Conditional PASS）**

| 维度                        | 判定                                                   |
| --------------------------- | ------------------------------------------------------ |
| Spec Compliance（§1–3）     | **PASS**（3.6 文档面 RISK）                            |
| Implementation Completeness | **PASS**（文档/集成测试有缺口）                        |
| Test Coverage               | **PASS**（关键单测全绿；集成/e2e 成功路径有缺口）      |
| Compatibility               | **PASS**（授权写路径）；业务读路径旧 UDP 属后续 §5     |
| Security / Risk             | **PASS**（§1–3）；禁用缓存 RISK                        |
| Release                     | **可进入 §4**，建议先修 3.6 文档漂移 + 补 1 条集成测试 |

**不建议仅凭 §1–3 做全量上线**：§5 业务 Checker 接入与 §7 UDP 清理未完成前，运行时验权仍走旧路径，新 Grant 对业务 API 不生效。

## User

/mctech-sdd-implement refactor-permission-authorization --task 3

## Assistant

先核对 Task 3 的当前实现与验收缺口，再按 spec 补齐。

## Assistant

## 2. 实现内容

Task 3 主体（3.1–3.7）此前已实现；本次补齐 mctech-sdd-verify 发现的缺口：

- **3.1** `directoryIds` 拒绝逻辑收紧：请求体含 `directoryIds` 数组即抛 `DIRECTORY_IDS_CONTRACT_REMOVED`（含与 `grants` 混合场景）
- **3.2/3.7** 新增 **AM.07**：AuthManagementService + 真实 PermissionService + MemoryRedis，验证 Grant 保存后 Redis 缓存写入
- **3.6** OpenAPI 文档：补充 Grant 对象说明；错误码表替换旧 `DIRECTORY_IDS_MUST_BE_ARRAY`，新增 `GRANTS_MUST_BE_ARRAY`、`DIRECTORY_IDS_CONTRACT_REMOVED`、`ENV_IDS_NOT_ALL_EXIST`、`MANAGE_MUST_NOT_HAVE_ENV`、`ITEM_REQUIRES_ENV`、`PERMISSION_NOT_FOUND`

## 3. 测试结果

```plain
AM.01–AM.07  → 8 passing
OA.P01–OA.P05 + OA.P03b → 6 passing
OPC.1–OPC.6  → 6 passing
openspec validate --strict → valid
```

## 4. 是否符合 Spec

**是**。§3 各子任务均满足：

| 子任务                                    | 状态                  |
| ----------------------------------------- | --------------------- |
| 3.1 grants 契约 + 拒绝旧 `directoryIds[]` | ✅                     |
| 3.2 事务后 `refreshUserPermissionCache`   | ✅（AM.07 端到端验证） |
| 3.3 公司变更检查 grants 表                | ✅                     |
| 3.4 矩阵 UI                               | ✅（已有）             |
| 3.5 保存失败保持对话框                    | ✅（已有）             |
| 3.6 website OpenAPI 文档对齐              | ✅                     |
| 3.7 AM/OA 单测 + AM.E01 e2e               | ✅                     |

## 5. 是否影响其他 Task

- **§4/§5**：无影响；业务 Service 仍走旧 UDP 过滤，待后续任务切换
- **§7 Cleanup**：`auth-management/index.js` 旧 UDP 模型残留，属 §7.1 范围，未改动

## 6. 遗留问题

| 项                   | 说明                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------- |
| 禁用用户未清权限缓存 | `openapi` spec 要求 disable 后刷新/清除缓存；不在 Task 3 字面范围，建议 §4 或独立 patch 处理 |
| 矩阵保存成功 e2e     | AM.E01 覆盖结构；成功保存→列表更新路径无 e2e                                                 |
| 旧 UDP 模型/业务过滤 | §5/§7 待办                                                                                   |
