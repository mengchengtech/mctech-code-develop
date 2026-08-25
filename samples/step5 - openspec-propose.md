# Permission system refactor

## User

现在基于以下已经确认的架构决策，为权限系统重构创建 OpenSpec Change。
Change 名称：refactor-permission-authorization
目标：重构现有权限项授权和验权体系，使系统形成统一的权限模型、授权模型和验权模型，并支持 Gateway + Service 双层验权、权限缓存、灰度迁移和旧权限模型兼容。
请创建：
1. proposal
2. design
3. specs
4. tasks

要求：
proposal：
说明为什么重构、当前问题、目标和非目标。
design：
详细描述：
- Permission Model
- PermissionItem Model
- Resource / Action
- Authorization Model
- Verification Model
- Gateway
- Service
- Authorization SDK
- Cache
- Permission Change Event
- Compatibility
- Migration

以上列出的项在前几轮讨论中存在就添加描述，不存在或已否决就忽略

## Assistant

架构文档已锁定：Website（非 Gateway）、无兼容/迁移/灰度。按 C1–C24 创建 Change；已否决项不写入 design。

## Assistant

Change `refactor-permission-authorization` 已就绪：`openspec validate --strict` 通过。

**路径**：`openspec/changes/refactor-permission-authorization/`

| Artifact                                 | 说明                                                                                                   |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `proposal.md`                            | Why / What / Capabilities / Impact / 非目标                                                            |
| `design.md`                              | Permission、PermissionItem、R+A、Authorization、Verification、Website、Service、Permission 模块、Cache |
| `specs/permission-authorization/spec.md` | 新能力：Grant、映射、Checker、缓存、summary、矩阵 UI                                                   |
| `specs/auth/spec.md`                     | 角色/菜单/粗拦 + `/me.permissionSummary`                                                               |
| `specs/openapi/spec.md`                  | grants 契约替换 `directoryIds[]`                                                                       |
| `tasks.md`                               | 7 组可勾选实现任务                                                                                     |

**相对你初始表述的对齐（按已锁定架构）**：
- 双层 = **Website + Service**（非 Gateway）
- **无**兼容 / 迁移 / 灰度（C9–C11）
- **无**独立 Authorization SDK → server 内 Permission 模块（C12）
- **无**消息队列式 Permission Change Event → 写路径同步重算→DEL→SET

实现：`/opsx-apply` 或直接让我按 tasks 开工。
