# Service list access bug

## User

/ai-fix BUG: 服务运营管理里，非管理员登录后进入任意的目录里，返回的都所有有权限目录下的服务列表

## Assistant

先定位「服务运营管理」目录权限与服务列表的 Spec/实现差异，再按 `/ai-fix` 流程修。

## Assistant

根因已确认：非 admin 时 `directoryId` 被 `Op.in: allDirIds` 覆盖。接着补失败用例并修复。

## Assistant

判定为 `IMPLEMENTATION_BUG`：非 admin 时权限目录集覆盖了传入的 `directoryId`。先写失败用例，再修实现。

## Assistant

### Fixed

**IMPLEMENTATION_BUG**：`listOnlineServices` 非 admin 时用全部可见目录 `Op.in` **覆盖**了传入的 `directoryId`，导致下钻任意目录都返回全部有权限目录的服务。

修复：有 `directoryId` 时校验在可见集内后只过滤该目录；无 `directoryId` 时才用全部可见目录；顺带去掉调试 `console.log`，无权限空结果统一返回 `[]`。

### Tests

`listOnlineServices` 全量 **29 passing**（含 SOS.115–117）

### Spec Compliance

符合「下钻到子目录 → 展示该目录直属服务」；权限集仅作可见性约束，不再覆盖目录作用域。

### Remaining Issues

无
