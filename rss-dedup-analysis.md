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

## 2.5 调度链路对照

Karakeep 存在两条独立的 RSS 抓取触发链路，它们在频率控制、入队参数和行为语义上有显著差异。

### 2.5.1 链路概述

| 维度 | 定时调度链路 | 手动触发链路 |
|------|-------------|-------------|
| 触发入口 | `FeedRefreshingWorker` cron 调度器 | tRPC `feeds.fetchNow` mutation |
| 代码位置 | `apps/workers/workers/feedWorker.ts:33-84` | `packages/trpc/routers/feeds.ts:81-93` |
| 触发时机 | 每小时整点自动触发 | 用户主动调用 API |
| 频率控制 | 强约束（每小时 1 次） | 无内置约束，可被频繁调用 |

### 2.5.2 入队参数差异

两条链路最终都调用 `FeedQueue.enqueue()`，但传递的参数截然不同：

**定时调度链路入队参数**（`feedWorker.ts:67-76`）：
```typescript
FeedQueue.enqueue(
  { feedId: feed.id },
  {
    idempotencyKey: `${feed.id}-${hourlyWindow}`,  // 每小时唯一
    groupId: feed.userId,
    delayMs: delayMinutes * 60 * 1000,              // 分钟偏移延迟
  },
);
```

**手动触发链路入队参数**（`feeds.ts:85-92`）：
```typescript
FeedQueue.enqueue(
  { feedId: ctx.feed.id },
  {
    groupId: ctx.user.id,
    // 无 idempotencyKey —— 每次调用创建独立任务
    // 无 delayMs —— 立即执行
  },
);
```

| 参数 | 定时调度 | 手动触发 | 影响 |
|------|---------|---------|------|
| `idempotencyKey` | `${feedId}-${YYYY-MM-DDTHH:00:00}` | 未提供 | 定时任务每小时去重；手动调用可重复入队 |
| `delayMs` | 0-3540000ms（0-59分钟） | 未提供（0ms） | 定时任务分散执行；手动任务立即执行 |
| `groupId` | `feed.userId` | `ctx.user.id` | 两者一致，用于同一用户内任务保序，全局仍串行 |

### 2.5.3 频率控制机制对比

**定时调度链路**：
1. cron 每小时触发一次调度器
2. 调度器遍历所有启用的 feed，每个 feed 生成一个任务
3. 通过 `idempotencyKey` 确保同一 feed 每小时只入队一次
4. 即使调度器被重复触发，队列的幂等机制会拒绝重复任务

**手动触发链路**：
1. 每次 API 调用直接入队一个任务
2. 无幂等保护，连续调用会产生多个排队任务
3. 依赖 `groupId` 按用户串行执行，但任务会累积
4. 无频率限制，理论上可无限触发

### 2.5.4 对去重前置的影响

两条链路共享同一套去重逻辑，但触发时机差异导致不同的去重行为：

**场景 1：正常定时调度**
- T=00:00 — 调度器运行，为 feed A 生成任务，延迟 15 分钟执行
- T=00:15 — 任务执行，抓取并去重，写入导入记录
- T=01:00 — 下一轮调度，生成新的幂等键，重复流程
- 结果：去重窗口稳定为 1 小时

**场景 2：定时 + 手动混合触发**
- T=00:00 — 定时任务入队，延迟 15 分钟
- T=00:05 — 用户点击"立即刷新"，手动任务入队，立即执行
- T=00:05 — 手动任务执行，发现并导入新条目，写入导入记录
- T=00:15 — 定时任务执行，查询导入记录时发现所有条目已存在，无新内容
- 结果：手动任务"抢占"了定时任务的工作，定时任务成为空跑

**场景 3：连续手动触发**
- T=00:00 — 用户第一次点击，任务 1 入队并执行
- T=00:01 — 用户第二次点击，任务 2 入队（因 `groupId` 串行，排队等待）
- T=00:02 — 任务 1 完成，任务 2 开始执行
- T=00:02 — 任务 2 执行，发现无新条目（RSS 源 1 分钟内无更新）
- 结果：任务 2 完全无效，浪费资源

### 2.5.5 对入库时序的影响

**延迟执行 vs 立即执行**：
- 定时任务的 `delayMs` 确保抓取操作在小时内均匀分布，避免 RSS 源服务器和自身数据库的流量突刺
- 手动任务无延迟，立即执行，可能与定时任务或其他用户的手动任务产生并发

**并发限制与 groupId 作用**：

Feed Worker 的全局并发数配置为 1（`feedWorker.ts:123`），这意味着**同一时间整个系统只能有一个 feed 抓取任务在执行**，所有任务（无论来自哪个用户）都是全局串行的。

`groupId` 的作用边界：
- `groupId` 用于确保**同一用户**的任务按入队顺序执行，避免同一用户的多个 feed 任务乱序
- 在全局并发数为 1 的限制下，`groupId` 不会带来并行性，仅用于组内保序
- 不同用户的任务虽然分属不同 group，但由于全局并发限制，仍然是串行执行

```
队列顺序：[用户A任务1, 用户B任务1, 用户A任务2, 用户C任务1]
执行顺序：用户A任务1 → 用户B任务1 → 用户A任务2 → 用户C任务1  （全局串行）
组内保序：用户A任务1 始终在 用户A任务2 之前执行
```

**实际并发行为**：
- 无并行执行，数据库连接池压力可控
- 手动任务可能被排在已有定时任务之后等待执行
- 同一用户连续点击"立即刷新"产生的多个任务会按顺序依次空跑

**状态字段的时序**：
- `lastFetchedAt`：每次任务执行完成（无论成败）都会更新
- `lastSuccessfulFetchAt`：仅在成功解析 feed 后更新（`feedWorker.ts:193-196`）
- `lastFetchedStatus`：任务完成后更新为 `success` 或 `failure`

当手动任务在定时任务之前执行成功时，定时任务执行时会看到：
- `lastSuccessfulFetchAt` 已被手动任务更新
- `lastFetchedStatus` 为 `success`
- 但定时任务仍会完整执行抓取流程，只是去重阶段发现无新条目

### 2.5.6 队列重试机制

FeedQueue 默认配置了 **1 次重试**（`shared-server/src/queues.ts:230`）：
```typescript
export const FeedQueue = createDeferredQueue<ZFeedRequestSchema>("feed_queue", {
  defaultJobArgs: {
    numRetries: 1,
  },
  keepFailedJobs: false,
});
```

**重试与调度的关系**：
- 定时任务和手动任务共享此重试配置，无差异
- 任务失败后会**立即重试 1 次**（无延迟），若再次失败则标记为永久失败
- 队列层面的重试是**同一次调度内的重试**，不会触发新的调度
- 即使任务最终失败（重试耗尽），**下一小时的定时调度仍然会正常执行**，两者互相独立
- 失败任务不会被保留（`keepFailedJobs: false`），无法手动重试，只能等待下一次调度或用户手动触发

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
2. **重试策略单一**：队列内置 1 次立即重试，无指数退避、无根据失败类型（网络超时 vs 解析错误）的差异化重试，重试耗尽后需等待下一小时调度
3. **全量查询**：每次抓取都查询所有已导入的 GUID，feed 条目多时可能有性能问题
4. **guid 冲突**：不同 feed 间相同 GUID 不会去重，可能导致重复书签
5. **手动触发无频率限制**：`fetchNow` API 无幂等保护和速率限制，可被滥用导致无效任务堆积
6. **定时任务空跑**：手动触发后定时任务仍会执行完整流程，造成不必要的资源浪费
7. **任务取消缺失**：已入队的延迟任务无法取消，用户手动触发后定时任务仍会执行

### 7.3 调度链路设计权衡

**双链路设计的合理性**：
- 定时链路保证系统自动化运行，无需用户干预
- 手动链路提供用户控制权，满足"立即看到新内容"的心理预期
- 两者共享同一套执行逻辑，避免代码重复

**可优化方向**：
1. 为 `fetchNow` 增加最短间隔限制（如 5 分钟内不重复入队）
2. 手动触发时取消同一 feed 已入队的定时延迟任务
3. 任务执行前检查 `lastSuccessfulFetchAt`，若距现在不足 N 分钟则直接跳过
4. 为手动任务设置独立的幂等键，防止用户快速重复点击

## 8. 测试验证

端到端测试覆盖了主要场景（`packages/e2e_tests/tests/workers/feed.test.ts`）：

- ✅ 基本抓取与书签创建
- ✅ 标签导入功能
- ✅ 重复抓取不创建重复书签
- ✅ 失败状态追踪（保留上次成功时间）
