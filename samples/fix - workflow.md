# 典型场景说明

> 假如 Task3 执行完成后，发现有测试用例跑不通的，应该执行什么

这种情况不要直接执行 `/mctech-sdd-implement` {next task}，也不要直接用 `/mctech-sdd-change` 走变更流程。

如果 Task3 已经完成，但运行/测试发现问题，首先运行要判断这是：

Task3 实现错误 → 修复 Task3
测试/环境问题 → 修复测试或环境
Spec 本身有问题 → /mctech-sdd-change
发现了新的业务需求 → /mctech-sdd-change

默认应该先执行 `/mctech-sdd-verify`。

执行 Task3 的完整流程

```plain
    Task3 完成
      ↓
    运行测试
      ↓
    发现测试失败
      ↓
    /mctech-sdd-verify --task 3
      ↓
    判断失败原因
```

然后根据结果分流：

```plain
                           /mctech-sdd-verify
                                  │
          ┌───────────────────────┼──────────────────────┐
          ↓                       ↓                      ↓
       实现错误                  Spec问题             测试/环境问题
          │                       │                      │
          ↓                       ↓                      ↓
    /mctech-sdd-fix      /mctech-sdd-change         修复测试/环境
                                  │
                                  ↓
                          /mctech-sdd-plan
```

核心原则：测试失败本身不是需求变更。

只有 `/mctech-sdd-verify` 判断它涉及 Spec、Design、架构或业务规则变化 时，才进入 `/mctech-sdd-change`。这样可以避免一个非常常见的问题：AI 一看到测试失败，就擅自修改需求或扩大任务范围。

## 示例

运行 `/mctech-sdd-verify`  返回 以下信息

<div style="color: #e8f3fe">
[FAIL] 3.3 — org-management 换公司未校验 grants  .......
RISK  

| **项严重度说明**                                                        |        |                                                                                             |
| ----------------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------- |
| `auth-management/index.js` 仍注册 `UserDirectoryPermission`（UDP 模型） | 警告   | 授权读写已走 grants；模型残留属 §7 Cleanup 范围，但与 spec「MUST NOT 再读写 UDP」字面有漂移 |
| Controller body schema 仍声明 `directoryIds`                            | 小问题 | `auth-management/controller.js` L36/L52；运行时会拒，但 OpenAPI/路由 schema 易误导          |
| OpenAPI `disable()` 未刷新/清除权限缓存                                 | 警告   | `openapi-user-service.js#disable`；delta spec 要求禁用后刷新或清除缓存                      |
| `rejectDirectoryIdsContract` 仅拒 `Array.isArray(directoryIds)`         | 小问题 | 非数组形态（如字符串）不会触发拒绝                                                          |
| Sequelize 模型未声明复合唯一索引                                        | 小问题 | 依赖 `schema.sql` DDL；henhouse 纯 sync 可能缺约束                                          |
| 业务模块仍读 UDP                                                        | 预期   | `service-operations`、`image-sync` — 属 §5/§7，非本批范围                                   |

### NOT VERIFIED

- website e2e `AM.E01`、`AMP.*`（Playwright 浏览器缺失）
- `/me` 透传 `permissionSummary`（§4，非本批）
- 细粒度业务验权切换（§5）
</div>

这个结果不能简单理解成“FAIL，所以全部修复”。实际上混合了 **真正的功能缺陷、Spec Drift、Cleanup 遗留、测试环境问题和本批次明确不包含的内容**。

标准流程应该先做分类决策，再决定是 `/mctech-sdd-implement`、`/mctech-sdd-fix`、`/mctech-sdd-change` 还是继续 `/mctech-sdd-verify`。


```plain
当前 Verify Report
       │
       ├── ① 3.3 grants 校验
       │       ↓
       │   修代码 + 测试
       │
       ├── ② disable 缓存
       │       ↓
       │   修代码 + 测试
       │
       ├── ③ directoryIds schema
       │       ↓
       │   修代码 + 测试
       │
       ├── ④ rejectDirectoryIdsContract
       │       ↓
       │   修代码 + 边界测试
       │
       ├── ⑤ Sequelize unique index
       │       ↓
       │   判断是否需要补 Task
       │
       ├── ⑥ UDP 模型注册
       │       ↓
       │   判断 Cleanup 是否属于本批
       │
       ├── ⑦ service-operations/image-sync
       │       ↓
       │   明确 Out of Scope
       │
       └── ⑧ E2E
               ↓
           补环境后重新 Verify
```
