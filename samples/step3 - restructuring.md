# Permission model restructuring

## User

现在进入“重构方案分析阶段”。
基于前面已经完成的系统探索结果，对现有权限项授权和验权体系进行重构方案分析。
本阶段仍然禁止修改代码。

permission-exploration/new-permission-requirements.md 为本次重构新增/变更的需求

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

至少提出 2～3 个可行架构方案。每个方案说明：
- 架构
- 优点
- 缺点
- 性能
- 安全性

## Assistant

进入重构方案分析阶段。先加载 OpenSpec 探索技能，并读取需求与既有探索结论。

## Assistant

完整分析已落盘：[`permission-exploration/analysis/refactor-architecture-options.md`](permission-exploration/analysis/refactor-architecture-options.md)。本阶段未改业务代码。

---

## 现状与新需求的核心矛盾

现行模型 = **角色门禁** + **目录可见范围**；新需求要的是 **目录 × 权限项（操作能力包）**。第三维缺失，且写路径基本未验权。

```plain
需求目标
User ──role──▶ admin ⇒ ALL
  │
  └── Grant(directoryPath, items[]) ──▶ 可见性 / 菜单 / 写操作 assert
```

---

## 16 问结论

| #               | 结论                                                                                |
| --------------- | ----------------------------------------------------------------------------------- |
| **1 根本问题**  | 无 Action；授权≠验权闭环；权威在 website；角色与能力耦合；列表过滤≠写授权           |
| **2 解耦**      | **应解耦**：共享 Grant/Redis，分 AuthorizationService 与 PermissionEvaluator        |
| **3 概念**      | Resource=目录/服务；Action=技术操作；**PermissionItem**=三项能力包；Grant=Item×目录 |
| **4 关系**      | 保留 `user.role`；developer 直接持有 Grant；admin 短路；**不建完整 RBAC 表**        |
| **5 权限项**    | 目录子树上的运维能力包，不是单 API/单按钮                                           |
| **6 验权标识**  | **对内** `action + resource`；**对外/授权** `permissionItem code`；可见性=任一 item |
| **7 Gateway**   | 认证 + 粗拦；**非权威**                                                             |
| **8 Service**   | **必须权威验权**（统一 Checker）                                                    |
| **9 内部 RPC**  | 传身份；**server 再验**；勿信「网关已验」                                           |
| **10 MQ/定时**  | 系统任务=`system`；用户异步消费时 assert（现状无 MQ）                               |
| **11 前后端**   | `/me` 下发 `permissionSummary`；前端只消费；后端必验                                |
| **12 Redis**    | `perm:user:{id}` = grants 快照（+可选目录展开集）                                   |
| **13 失效**     | 改授权：DB 事务 → 重算 → **DEL 再 SET**；目录变更批量重算；禁用踢 session           |
| **14 多租户**   | 一期维持弱租户（`company_id` 组织属性）；强隔离非目标                               |
| **15 API 兼容** | 扩展 permissions shape；旧 `directoryIds` 过渡；写操作收紧允许 breaking             |
| **16 DB 迁移**  | 加 items → backfill（产品定）→ 双写 → 强制 → 重建 Redis → 开 assert                 |

**三项 PermissionItem 草案**：`svc.lifecycle` / `svc.edit_publish` / `svc.deploy`（命名待确认）。

---

## 三个架构方案

### 方案 A — 目录 ACL + 权限项（推荐一期）

website：session + 粗拦 + `/me` summary → server：`PermissionChecker` 读 Redis，list/assert 统一执行。

|          |                                                          |
| -------- | -------------------------------------------------------- |
| **优点** | 贴合需求；复用 UDP/授权 UI；改动可控                     |
| **缺点** | 环境维度弱；item 增多需再演进                            |
| **性能** | 缓存命中后列表 IN + assert 路径匹配；可接受              |
| **安全** | 写路径强制 assert 后明显强于现状；仍依赖 server 网络隔离 |

### 方案 B — RBAC-lite（Role 默认 + 用户覆盖）

在 A 上增加角色默认 Grant，为「发布员」等预留。

|          |                             |
| -------- | --------------------------- |
| **优点** | 扩展性好                    |
| **缺点** | 当前仅 2 角色，一期成本偏高 |
| **性能** | 与 A 相近                   |
| **安全** | 与 A 同级；配置更易出错     |

### 方案 C — Gateway 集中鉴权 + Server 信任

|          |                           |
| -------- | ------------------------- |
| **优点** | 业务代码干净              |
| **缺点** | **重蹈覆辙**（直连绕过）  |
| **性能** | Gateway 多一次 I/O        |
| **安全** | **弱** — 不推荐作权威模型 |

---

## 建议锁定方向

**一期采用方案 A**：权威在 server；Item×目录；admin 短路；Redis 按人缓存；Gateway 只做认证/粗拦。

仍需产品确认：

1. 旧 UDP 行 backfill 成哪些 items？
2. 「特定环境」是否进 Grant？
3. 页面功能 × 权限项矩阵（需求已要求）
4. OpenAPI 旧客户端默认策略
5. server 是否仅 website 可达

收敛后可用 `/opsx:propose` 立变更（如 `directory-permission-items`）写入 proposal/design。
