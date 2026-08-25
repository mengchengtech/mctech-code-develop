# Architectural decision constraints

## User

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

## Assistant

基于已确认的架构决策整理最终技术方案。先读取现有探索文档与项目知识。

## Assistant

终稿：`permission-exploration/analysis/final-architecture.md`

### 决策冲突（未擅自消解，需你确认）

| ID    | 冲突                                                              | 可选解读                                                                                         |
| ----- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **A** | C3「Service 负责细粒度验权」vs C4「Service 不许自行实现判断算法」 | **A1** Service 只做调用点，算法在统一 Permission 组件；**A2** 业务 Service 自写判断 → 与 C4 矛盾 |
| **B** | C5「Resource + Action」vs C6「PermissionItem 可授权」             | **B1** Item 授权、R+A 验权，固定映射；**B2** 只能二选一                                          |
| **C** | C8「必须缓存失效」vs 产品要求「重算后先删后写」                   | **C8-1** 仅 DEL/懒加载；**C8-2** 同步重算 → DEL → SET                                            |

编号缺 **#10**，已按原文保留，未补造。

### 方案骨架（暂按 A1/B1 展开，冲突确认后可改）

```plain
Website：认证 + API 粗拦 + /me summary
    │
Service：业务资源 assert/过滤 ──调用──► PermissionChecker（唯一算法）
                                          ▲
AuthManagement：Grant(Item) 写库 ──► Redis 用户权限集合失效/重算
```

- Permission = Resource + Action；PermissionItem = 可授权能力包（四项 + 环境作用域）
- 未上线：直接新 grants 表/契约，丢弃 UDP，无迁移/灰度/回滚
- 完整矩阵、Item code、Redis schema 仍待定

请先拍板冲突 **A / B / C**。确认后可走 `/opsx:propose`。

## User

PermissionChecker是什么，独立部署还是公共库

## Assistant

**PermissionChecker** = 方案里的**逻辑组件名**：唯一实现「用户对某 Resource 是否有某 Action」的判断算法。业务 Service 只调用它（对应冲突 A 的 A1 读法）。**部署形态架构师尚未锁定。**

### 两种落法

|            | 公共库（进程内）                                            | 独立服务                                    |
| ---------- | ----------------------------------------------------------- | ------------------------------------------- |
| 形态       | `server` 内模块，或 monorepo 抽 `@mctech/...-permission` 包 | 单独进程/RPC，业务 Service 远程问「能不能」 |
| 谁执行算法 | 与业务同进程                                                | 权限服务进程                                |
| 延迟       | 读 Redis / 本地判定，低                                     | 多一跳 RPC                                  |
| 与现状     | 现仅有 `server` + `website`，贴合                           | 多一套部署与可用性                          |
| 满足 C4    | 是（算法不进业务 Service）                                  | 是                                          |

### 建议（非已确认决策）

**本期用公共库 / server 内 Permission 模块**，不独立部署。

理由：server 仅 website 可达；细粒度验权本就在 Service 侧；权威数据在 Redis 用户权限集合；独立服务本期收益小、运维成本高。

Website 粗拦若需同一套 Item↔(R,A) 映射，可复用同一库做只读判断；**资源级权威仍在 server 侧 Checker**。

若你确认「公共库」或「独立服务」，可写回 `final-architecture.md`。

## User

本期改为server内模块

## Assistant

已写入终稿 **C12**：

- PermissionChecker = **`server` 进程内模块**
- 非独立部署、本期不抽跨仓公共库
- 冲突 A 按 **A1** 关闭

仍待确认：冲突 **B**、**C**，以及 Item code / Redis schema / 完整矩阵。

## User

选B1,C8-2

## Assistant

将冲突 B、C 分别锁定为 B1 与 C8-2，并更新终稿。

## Assistant

已写入终稿：

| 约束           | 结论                                                      |
| -------------- | --------------------------------------------------------- |
| **C13 / B1**   | 授权 = PermissionItem；验权 = Resource + Action；固定映射 |
| **C14 / C8-2** | 写授权后 **同步重算 → DEL → SET**                         |

架构冲突 A/B/C 均已关闭。仍待：Item/Action 正式 code、Redis schema、完整 UI 矩阵。

下一步：`/opsx:propose`（建议 `directory-permission-items`）。

## User

重新归纳下还有什么需要决策的。以及决策的内容

## Assistant

架构冲突已全部关闭。还剩这些决策：

### 必须先拍（阻塞契约）

| ID     | 决策内容                      | 要拍什么                                                                                          |
| ------ | ----------------------------- | ------------------------------------------------------------------------------------------------- |
| **P1** | Item 正式 code + 显示名       | 四项稳定标识；草案 `svc.manage` / `svc.edit_publish` / `svc.deploy` / `svc.k8s_ops`；k8s 文案未定 |
| **P2** | Action 枚举 + Item→(R,A) 映射 | 每个 Item 展开成哪些 Action；Resource 怎么标识；映射单一事实源                                    |
| **P3** | Redis value schema            | key/字段/索引/版本；草案 `atom-ops:perm:user:{userId}` + `grants` + `indexes`                     |
| **P4** | 完整权限矩阵                  | 各页面/按钮/dialog 归属哪个 Item（±环境）；§7 仅骨架，需产品确认                                  |

### 建议 propose 时一并定

| ID     | 决策内容                                      | 选项                                          |
| ------ | --------------------------------------------- | --------------------------------------------- |
| **P5** | Website 是否做「无 Grant 则拦服务 API」粗门槛 | 做 / 不做（只靠角色门禁 + server）            |
| **P6** | `/me` summary 谁组装                          | website 读 Redis / 调 server / 仅 server 返回 |
| **P7** | 系统定时任务身份                              | `system` 旁路 / 专用账号走 Checker            |

### 可在 design 默认选定

| ID     | 决策内容                         |
| ------ | -------------------------------- |
| **P8** | Grants 正式表名与 DDL 细节       |
| **P9** | Redis miss：回源回填 vs 直接失败 |

---

**已锁定、无需再议**：双层验权、server 内 Checker、Item 授权 + R+A 验权、重算→DEL→SET、无兼容/迁移/灰度、矩阵 UI α、四项能力与环境规则骨架。

详情已写入 `final-architecture.md` §10。优先关 **P1–P4**，再 `/opsx:propose`。

## User

P1-P4，先自主生成，人工决策；P5 「无 Grant 则拦服务 API」是什么意思；P6 调server/仅server返回 有什么区别；P7 system 旁路是什么意思

## Assistant

P1–P4 自主草案已写入 `final-architecture.md` §10.1，摘要如下。

### P1–P4 草案（待你确认）

| ID     | 自主结论                                                                                                                                        |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **P1** | code：`svc.manage` / `svc.edit_publish` / `svc.deploy` / `svc.k8s_ops`；显示名「服务增删改 / 特定环境下编辑+发布 / 部署 / Kubernetes 运维」     |
| **P2** | Resource=`directory \| service \| menu`；Action 按 manage/edit_publish/deploy/k8s 拆细；Item→Action 固定映射；`list.visible`=任一 Item          |
| **P3** | key=`atom-ops:perm:user:{userId}`；value=`v/role/grants/indexes`；写路径重算→DEL→SET                                                            |
| **P4** | 已按页面扫齐：CRUD/公共配置→manage；环境配置/发布→edit_publish；部署/镜像同步→deploy；重启扩缩 CronJob→k8s_ops；只读列表→任一可见；组织等→admin |

可直接回：「P1–P4 按草案」或逐条改。

---

### P5「无 Grant 则拦服务 API」

某 developer **四个服务权限项一条都没有**时：

- **不做**：website 仍放行服务运营等 API → 打到 server → 空列表/403  
- **做**：website 直接拦（403/重定向），请求不到 server  

只判「有没有任一服务 Grant」，**不**按具体 serviceId 细判。

---

### P6「调 server」vs「仅 server 返回」

都是给前端 `permissionSummary`，差别在谁拼装：

|              | **调 server（B）**                               | **仅 server 返回（C）**                |
| ------------ | ------------------------------------------------ | -------------------------------------- |
| 前端入口     | 通常仍是 website `/me`                           | server 权限接口（website 透传）        |
| website 职责 | `/me` 内 RPC 问 server，再与用户信息**合并**返回 | **不合并**；权限体只来自 server        |
| 共同点       | 都不直读权限 Redis；算法/缓存在 server           | 都不直读权限 Redis；算法/缓存在 server |

另有方案 A：website 直读 Redis（与 B/C 不同）。

---

### P7「system 旁路」

定时任务等**没有登录用户**时：

- **旁路**：Permission 模块认出「系统身份」→ **跳过** R+A 校验直接放行（集中定义，禁止业务里私自跳过）  
- **对照**：建服务账号并授 Grant，任务带该身份照常 `assert`

---

详表见终稿 §10。确认 P1–P4（及可选 P5–P7）后可 `/opsx:propose`。

## User

P1 同意。svc.k8s_ops 显示名用「Kubernetes 运维」
P2 映射边界（restore/purge收紧为 admin-only）
P3 shape是什么
P4 整表归属是什么？只读是否一律「任一可见」是什么意思？
P5 做
P6 选C
P7 选system旁路
P8 同意
P9 权限读取redis是在website还是server？

## Assistant

先澄清 P3/P4/P9，再把已确认项写入终稿。

## Assistant

### 已写入终稿

P1、P2（restore/purge→admin）、P5、P6=C、P7 system 旁路、P8 → C15–C21。

---

### 你问的三点

**P3 shape**  
= Redis **value 的 JSON 长什么样**（字段结构），不是算法。  
草案：`{ v, role, grants[], indexes{...} }`。  
是否采用该 shape，**还待你点头**。

**P4 整表归属**  
= §10.3 矩阵里每一行「操作 → 哪个 Item」是否都同意。

**只读一律「任一可见」**  
= 列表/详情/日志/YAML/监控等**只读**操作：只要对该目录有**任意一个**服务 Item，就能看；不必再有 deploy/k8s。  
例：只有 deploy、没有 k8s，也能看 Pod 日志。  
若不同意，需另定「哪些只读要绑写权限」。

**P9 权限 Redis 在哪读**  
| 数据                   | 位置             |
| ---------------------- | ---------------- |
| `perm:user:*` 权限集合 | **server** 读/写 |
| Session                | website          |

与 P6=C 一致：website **不**读权限 Redis。P5「有无 Grant」通过 **调 server / 透传 summary** 得知。  
P9 还剩 miss 策略未选：回源回填 / 直接失败。

---

### 仍待你确认

1. **P3**：是否采用该 Redis JSON shape？  
2. **P4**：整表是否同意？只读是否一律「任一可见」？配置组是否仍归 `svc.manage`？  
3. **P9 miss**：server 读 miss 时回源回填，还是直接失败？

## User

P3 同意
P4 同意
P9 回源重算回填

## Assistant

将 P3、P4、P9 落盘为已确认约束。

## Assistant

已落盘：

| ID  | 约束                            |
| --- | ------------------------------- |
| P3  | **C22** Redis shape 定稿        |
| P4  | **C23** 整表 + 只读一律任一可见 |
| P9  | **C24** miss → 回源重算回填     |

**P1–P9 全部关闭。** 终稿约束 **C1–C24**。

下一步：`/opsx:propose`（建议 `directory-permission-items`）。
