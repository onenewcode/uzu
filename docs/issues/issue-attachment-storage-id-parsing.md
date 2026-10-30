# P2: GUI 附件清理依赖 ID 字符串前缀推断创建时间

## 元信息

| 项 | 内容 |
| --- | --- |
| Severity | P2 |
| 组件 | `mirai-gui` / attachment storage |
| 主要文件 | `apps/mirai-gui/client/src/features/chat/services/attachment-storage.ts` |
| 关联文件 | `apps/mirai-gui/client/src/features/chat/services/attachment-files.ts`、`apps/mirai-gui/client/src/types/files.ts` |
| 状态 | 待修复 |

## 概述

附件 ID 目前由时间戳和随机数拼接生成：

```ts
const generateFileId = (): string => {
  return `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
};
```

清理逻辑从 ID 反解析时间：

```ts
const timestamp = Number.parseInt(file.id.split("-")[0] ?? "", 10);
if (Number.isFinite(timestamp) && timestamp < sevenDaysAgo) {
  fileIdsToDelete.push(file.id);
}
```

`AttachedFile` 数据结构本身没有 `createdAt`，这属于把业务语义编码在 identifier 格式中的设计问题。

## 影响

1. **数据迁移风险**
   - 一旦 ID 生成规则改变，旧附件无法推断保留时间。
   - 使用 `crypto.randomUUID()` 等新格式时，清理逻辑会失效。

2. **无法解析的记录永久保留**
   - 如果 `file.id` 不含可解析时间戳，`cleanup()` 不会删除。
   - localStorage 可能持续增长。

3. **持久化 JSON 没有 schema 校验**
   - 旧版本或测试写入的数据可能不是预期对象。
   - 目前 `JSON.parse` 后直接赋给 `storage`，类型上是信任的。

4. **测试与多环境隔离风险**
   - singleton 在导入时读取 localStorage，测试间共享状态，除非显式重置。

## 建议修复

### 1. 增加显式字段

```ts
export type AttachedFile = {
  id: string;
  createdAt: number;
  ...
};
```

### 2. ID 使用随机 UUID

```ts
const generateFileId = (): string => {
  if (typeof crypto !== "undefined" && "randomUUID" in crypto) {
    return crypto.randomUUID();
  }
  return `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`;
};
```

清理不再依赖 fallback 格式。

### 3. 数据迁移

加载时执行：

```ts
const createdAt = file.createdAt ?? parseLegacyTimestamp(file.id);
```

若无法解析，选择明确策略：保留并标记待处理，或一次性删除无法识别的记录。建议保守保留，但避免永久无主数据。

### 4. 校验 persisted shape

解析后过滤非对象、缺 id、缺 content 的记录，避免运行时访问 undefined。

### 5. 增加存储类测试

目前 `attachmentStorage` 没有直接单元测试。应导出可测试类或提供 `resetForTests()`。

## 建议测试

1. **cleanup deletes old files**
   - `createdAt` 早于 7 天。
   - 断 localStorage 中删除。

2. **cleanup keeps recent files**
   - `createdAt` 在 7 天内。
   - 断保留。

3. **uuid ids cleanup**
   - ID 是 UUID。
   - 时间只看 `createdAt`。

4. **legacy migration**
   - 旧数据没有 `createdAt`，ID 为 `Date.now()-random`。
   - 断补齐 createdAt。

5. **invalid persisted data**
   - JSON 包含数组、字符串或 null。
   - 断不会崩溃，并清理为空映射。

## 验证命令

```bash
cd apps/mirai-gui
pnpm typecheck
pnpm test
pnpm lint
```

