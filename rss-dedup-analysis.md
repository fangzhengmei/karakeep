# Karakeep RSS Feed 解析与去重机制分析

## 1. 系统架构概览

Karakeep 的 RSS feed 处理系统采用 **定时调度 + 异步队列** 的架构，主要由以下组件构成：

- **调度器**：`FeedRefreshingWorker`（`apps/workers/workers/feedWorker.ts:33-84`）
- **解析器**：`parseFeedItems`（`apps/workers/workers/utils/feedParser.ts:34-41`）
- **去重引擎**：基于 `rssFeedImportsTable` 的 GUID 匹配机制
- **入库流程**：通过 tRPC 客户端调用书签创建 API

## 2. 抓取频率策略

### 2.1 调度机制

- **调度周期**：每小时整点触发（cron 表达式 `0 * * * *`），见 `feedWorker.ts:33`
- **流量平滑**：通过 `getFeedMinuteOffset(feedId)` 函数将 feed 分散到小时内不同分钟执行
- **幂等保证**：使用 `${feed.id}-${hourlyWindow}` 作为幂等键，确保同一 feed 每小时只调度一次

### 2.2 分钟偏移算法

```typescript
// apps/workers/workers/feedWorker.ts:22-31
function getFeedMinuteOffset(feedId: string): number {
  let hash = 0;
  for (let i = 0; i < feedId.length; i++) {
    hash = (hash << 5) - hash + feedId.charCodeAt(i);
    hash = hash & hash;
  }
  return Math.abs(hash) % 60;
}
```

**设计意图**：
- 基于 feed ID 确定性哈希，保证同一 feed 始终在固定分钟执行
- 避免所有 feed 在整点同时抓取造成的流量突刺
- 延迟计算考虑了当前分钟，若目标分钟已过则调度到下一小时

### 2.3 任务执行配置

- **并发数**：1（`feedWorker.ts:123`）
- **轮询间隔**：1000ms
- **超时时间**：30秒
- **抓取超时**：5秒（`AbortSignal.timeout(5000)`，`feedWorker.ts:166`）

## 3. 条目标准化流程

### 3.1 解析库与配置

使用 `rss-parser` 库，支持自定义字段 `id`：

```typescript
// apps/workers/workers/utils/feedParser.ts:4-8
const parser = new Parser({
  customFields: {
    item: ["id"],
  },
});
```

### 3.2 字段标准化规则

通过 Zod schema 实现严格的类型验证和转换：

| 字段 | 来源优先级 | 处理逻辑 |
|------|-----------|----------|
| `guid` | `item.guid` → `item.id` → `item.link` | 三级降级策略，确保每个条目都有唯一标识 |
| `link` | `item.link` | 可选，无链接的条目会被过滤 |
| `title` | `item.title` | 可选 |
| `categories` | `item.categories` | 支持字符串和 `{ _: string }` 对象两种格式，统一转换为字符串数组 |

```typescript
// apps/workers/workers/utils/feedParser.ts:19-30
const feedItemSchema = z.object({
    id: optionalStringSchema,
    link: z.string().optional(),
    guid: z.string().optional(),
    title: z.string().optional(),
    categories: z.array(categorySchema).optional(),
  }).transform((item) => ({
    ...item,
    guid: item.guid ?? item.id ?? item.link,
  }));
```

### 3.3 容错处理

- 使用 `safeParse` 进行解析，解析失败的条目被静默过滤（`flatMap` 模式）
- 不抛出异常，确保单个坏条目不影响整个 feed 的处理

## 4. 重复判断机制

### 4.1 去重策略

采用 **基于 GUID 的存在性检查** + **数据库唯一约束** 的双层保障：

**第一层：应用层过滤**（`feedWorker.ts:207-222`）

```typescript
// 查询已导入的条目
const exitingEntries = await db.query.rssFeedImportsTable.findMany({
  where: and(
    eq(rssFeedImportsTable.rssFeedId, feed.id),
    inArray(
      rssFeedImportsTable.entryId,
      feedItems.map((item) => item.guid).filter((id): id is string => !!id),
    ),
  ),
});

// 过滤新条目
const newEntries = feedItems.filter(
  (item) =>
    !exitingEntries.some((entry) => entry.entryId === item.guid) &&
    item.link &&
    item.guid,
);
```

**第二层：数据库约束**（`packages/db/schema.ts:682`）

```typescript
unique().on(bl.rssFeedId, bl.entryId),
```

### 4.2 去重范围

- 去重是 **per-feed** 的，不同 feed 之间的相同 GUID 不会被判定为重复
- 必须同时满足 `guid` 和 `link` 存在才会被考虑导入

### 4.3 导入记录表结构

| 字段 | 说明 |
|------|------|
| `entryId` | RSS 条目的 GUID，用于去重匹配 |
| `rssFeedId` | 关联的 feed ID |
| `bookmarkId` | 关联的书签 ID，可为 null（导入失败时） |

## 5. 入库衔接流程

### 5.1 书签创建

```typescript
// apps/workers/workers/feedWorker.ts:238-247
const createdBookmarks = await Promise.allSettled(
  newEntries.map((item) =>
    trpcClient.bookmarks.createBookmark({
      type: BookmarkTypes.LINK,
      url: item.link!,
      title: item.title,
      source: "rss",
    }),
  ),
);
```

**关键设计**：
- 使用 `buildImpersonatingTRPCClient(feed.userId)` 模拟用户身份调用
- 采用 `Promise.allSettled` 确保单个书签创建失败不影响其他条目
- `source: "rss"` 标记来源，便于后续追溯

### 5.2 标签导入（可选）

当 `feed.importTags` 为 true 时，将 RSS categories 作为标签附加：

```typescript
// apps/workers/workers/feedWorker.ts:250-273
if (feed.importTags) {
  await Promise.allSettled(
    newEntries.map(async (item, idx) => {
      const bookmark = createdBookmarks[idx];
      if (bookmark.status === "fulfilled" && item.categories?.length) {
        await trpcClient.bookmarks.updateTags({
          bookmarkId: bookmark.value.id,
          attach: item.categories.map((tagName) => ({ tagName })),
          detach: [],
        });
      }
    }),
  );
}
```

### 5.3 导入记录写入

```typescript
// apps/workers/workers/feedWorker.ts:276-288
await db.insert(rssFeedImportsTable)
  .values(
    newEntries.map((item, idx) => {
      const b = createdBookmarks[idx];
      return {
        entryId: item.guid!,
        bookmarkId: b.status === "fulfilled" ? b.value.id : null,
        rssFeedId: feed.id,
      };
    }),
  )
  .onConflictDoNothing();
```

**设计特点**：
- 非事务性设计，即使导入记录写入失败，书签已创建的事实不会回滚
- `onConflictDoNothing()` 处理并发场景下的唯一约束冲突
- 书签创建失败时 `bookmarkId` 设为 null，该 GUID 仍会被标记为已导入，避免重试

### 5.4 状态更新

- 成功时更新 `lastFetchedStatus: "success"` 和 `lastFetchedAt`
- 成功解析后立即更新 `lastSuccessfulFetchAt`（`feedWorker.ts:193-196`）
- 失败时更新 `lastFetchedStatus: "failure"`，但保留 `lastSuccessfulFetchAt`

## 6. 容错与边界处理

### 6.1 配额检查

在抓取前检查用户书签配额，避免无效抓取：

```typescript
// apps/workers/workers/feedWorker.ts:150-159
const quotaResult = await QuotaService.canCreateBookmark(db, feed.userId);
if (!quotaResult.result) {
  logger.debug(`User ${feed.userId} doesn't have enough quota. Skipping.`);
  return;
}
```

### 6.2 内容类型校验

```typescript
// apps/workers/workers/feedWorker.ts:179-184
const contentType = response.headers.get("content-type");
if (!contentType || !contentType.includes("xml")) {
  throw new Error(`Feed is not a valid RSS feed`);
}
```

### 6.3 空条目处理

当 feed 中没有新条目时，直接返回不执行后续操作。

## 7. 设计权衡分析

### 7.1 优点

1. **流量平滑**：分钟偏移算法避免整点流量突刺
2. **容错性强**：`allSettled` 模式确保单个失败不影响整体
3. **去重可靠**：应用层 + 数据库双层去重保障
4. **可追溯**：`rssFeedImportsTable` 记录完整导入历史
5. **身份隔离**：通过模拟 tRPC 客户端确保权限控制一致

### 7.2 潜在改进点

1. **非事务性**：书签创建与导入记录写入非原子，极端情况下可能出现 GUID 未标记但书签已创建
2. **无重试机制**：单次抓取失败后需等待下一小时调度
3. **全量查询**：每次抓取都查询所有已导入的 GUID，feed 条目多时可能有性能问题
4. **guid 冲突**：不同 feed 间相同 GUID 不会去重，可能导致重复书签

## 8. 测试验证

端到端测试覆盖了主要场景（`packages/e2e_tests/tests/workers/feed.test.ts`）：

- ✅ 基本抓取与书签创建
- ✅ 标签导入功能
- ✅ 重复抓取不创建重复书签
- ✅ 失败状态追踪（保留上次成功时间）
