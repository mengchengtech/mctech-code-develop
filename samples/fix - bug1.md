# Service management access bug

## User

/ai-fix BUG: 非管理员登录进入服务管理列表，能看到所有产品线信息，应该只能看到自己有权限的产品线目录

## Assistant

定位非管理员在服务管理列表的产品线权限过滤问题，先查 OpenSpec 变更与相关实现。

## Assistant

根因：前端根层用未过滤的 `/services/common/product-lines` 拉全量产品线。将新增按 `manageDirIds` 过滤的后端接口，并改前端调用。

## Assistant

### Fixed

**现象**：非 admin 进入服务管理根层，通过 `/services/common/product-lines` 拉取全量产品线。

**根因**：子目录/服务列表已按 `manageDirIds` 过滤，根层产品线仍走未过滤的 common 接口。

**修复**：
- 新增 `GET /service-lifecycle/product-lines`，`listVisibleProductLines` 按 `manageDirIds` 关联的 `productLineId` 去重返回；admin 全量，无 Grant 返回空
- 前端 `fetchProductLines` 改调上述接口

### Tests

```bash
cd server && npm test -- --grep "service-lifecycle list visibility"
```

结果：**10 passing**（含新增 SL.L09）

`openspec validate refactor-permission-authorization --strict`：**通过**

### Spec Compliance

符合 `permission-authorization/spec.md`「服务管理目录/服务列表 MUST 仅使用 `manageDirIds`」及 Scenario「服务管理仅展示 manage 目录」——根层产品线现为 manage 授权目录的产品线子集。

### Remaining Issues

无。服务运营根层仍用 `/services/common/product-lines`（按 env 类 Grant 过滤，属独立范围）。

## User

服务运营和任务管理的列表也一起检查下

## Assistant

检查服务运营与任务管理的产品线列表权限过滤。

## Assistant

## 检查结果

| 模块 | 根层产品线 | 子目录/服务/任务 | 结论 |
|------|-----------|----------------|------|
| **服务管理** | 已修复（上轮） | `manageDirIds` 过滤 | ✅ |
| **服务运营** | 存在相同问题 → **已修复** | `listDirectories` / `listServices` 已按 env 过滤 | ✅ |
| **任务管理** | 无产品线列表 | `listDeployTasks` / `listSyncTasks` 已按 env + 目录 Grant 过滤 | ✅ 无需改动 |

## 服务运营修复

**现象**：根层仍走 `/services/common/product-lines`，非 admin 能看到全部产品线。

**修复**：
- 新增 `GET /service-operations/product-lines?envId=`，`listVisibleProductLines` 按当前 env 的环境类 Grant 目录关联 `productLineId` 去重
- 前端 `fetchProductLines({ envId })` 改调新接口，未选环境时清空列表

## 任务管理

页面结构为「选环境 → 部署任务 / 镜像同步任务」两个 Tab，不涉及产品线导航。后端：
- `listDeployTasks` — 非 admin 按 `#resolveVisibleDirectoryIds(userId, role, envId)` 过滤
- `listSyncTasks` — 同上（TM.IS.02 已覆盖）

## 测试

```
SO.P02 / SO.P03 — 2 passing
TM.IS.02 — passing
```

建议手动验证：非 admin 登录 → 服务运营选 E1 → 根层仅见 E1 有 deploy/edit_publish/k8s_ops Grant 的产品线。
