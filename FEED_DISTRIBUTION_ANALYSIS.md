# Karakeep Feed 分发链路深度分析

## 目录
1. [分发总览](#分发总览)
2. [请求入口矩阵](#请求入口矩阵)
3. [数据读取接口](#数据读取接口)
4. [Web 端分发实现](#web-端分发实现)
5. [移动端分发实现](#移动端分发实现)
6. [状态同步与一致性](#状态同步与一致性)
7. [端到端时序图](#端到端时序图)
8. [跨端差异对比](#跨端差异对比)

---

## 分发总览

### 数据流向图

```
RSS Feed Source
     │
     ▼
┌─────────────────────────────────────────────────────┐
│             Feed Worker (Queue-based)               │
│  • 哈希分钟偏移分发 • 配额检查 • 去重导入         │
└─────────────────────────────────────────────────────┘
     │
     ▼ 写入
┌─────────────────────────────────────────────────────┐
│              PostgreSQL Database                    │
│  • bookmarks • bookmarkLinks • bookmarkTags • ...   │
└─────────────────────────────────────────────────────┘
     │
     │ ────────────────────────────────────────────────
     │ │                                           │
     ▼ ▼                                          ▼
┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ TRPC API 层  │  │ Search Indexer   │  │ 下游 Worker 触发  │
│ • 查询 • 搜索│  │ • MeiliSearch    │  │ • Crawler • AI    │
└──────────────┘  └──────────────────┘  └──────────────────┘
     │                    │                       │
     ▼                    ▼                       ▼
┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   Web 端    │  │   Web 端搜索     │  │   异步回调       │
│ (React +   │  │   (实时索引)     │  │   (Webhook 等)   │
│  Next.js)  │  └──────────────────┘  └──────────────────┘
└──────────────┘
     ▼
┌──────────────┐
│   移动端    │
│ (React Native │
│  + Expo)    │
└──────────────┘
```

### 关键特性

| 特性 | 说明 |
|-----|------|
| **读写分离** | 写入通过 Worker 异步执行，读取通过 TRPC 同步接口 |
| **索引分离** | 数据库用于结构化查询，MeiliSearch 用于全文搜索 |
| **多端一致** | Web 端和移动端共享相同的 TRPC 接口和业务逻辑 |
| **最终一致** | 搜索索引、AI 标签、抓取状态均异步更新 |

---

## 请求入口矩阵

### TRPC Router 接口定义

**文件**: `packages/trpc/routers/bookmarks.ts`

| 接口 | 类型 | 用途 | 核心参数 | 权限检查 |
|-----|-----|------|---------|---------|
| `getBookmark` | Query | 获取单个 Bookmark | `bookmarkId`, `includeContent` | `ensureBookmarkAccess` |
| `getBookmarks` | Query | 批量获取 Bookmark 列表 | `limit`, `cursor`, `sortOrder` | `createBookmarksQueriedMiddleware` |
| `searchBookmarks` | Query | 全文搜索 Bookmark | `text`, `limit`, `cursor`, `sortOrder` | `createEventLogMiddleware("search.query")` |
| `checkUrl` | Query | 检查 URL 是否已存在 | `url` | - |
| `getReadingProgress` | Query | 获取阅读进度 | `bookmarkId` | `ensureBookmarkAccess` |
| `createBookmark` | Mutation | 创建 Bookmark | `type`, `url`, `title`, `tags` | `createRateLimitMiddleware` |
| `updateBookmark` | Mutation | 更新 Bookmark | `bookmarkId`, 字段更新 | `ensureBookmarkOwnership` |
| `updateTags` | Mutation | 更新标签关联 | `bookmarkId`, `attach`, `detach` | `ensureBookmarkOwnership` |
| `updateReadingProgress` | Mutation | 更新阅读进度 | `bookmarkId`, 进度字段 | `ensureBookmarkAccess` |
| `deleteBookmark` | Mutation | 删除 Bookmark | `bookmarkId` | `ensureBookmarkOwnership` |
| `recrawlBookmark` | Mutation | 触发重新抓取 | `bookmarkId` | 速率限制 + 所有权检查 |

### 接口调用链

```
┌───────────────────────────────────────────────────────────────┐
│                      接口调用树                                │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  getBookmarks (批量列表)                                      │
│       ↓                                                       │
│  └──> Bookmark.loadMulti()                                    │
│            ↓                                                  │
│            ├──> 主键查询 (带分页 cursor)                       │
│            ├──> 关联预加载 (link/text/assets/tags)            │
│            └──> 权限过滤 (ensureOwnership)                    │
│                                                               │
│  getBookmark (单个详情)                                       │
│       ↓                                                       │
│  └──> Bookmark.fromId()                                       │
│            ↓                                                  │
│            ├──> 主键查询 + 关联预加载                         │
│            └──> 内容加载 (可选 includeContent)                │
│                                                               │
│  searchBookmarks (全文搜索)                                   │
│       ↓                                                       │
│  ├──> parseSearchQuery() → 构建 Matcher                       │
│  ├──> getBookmarkIdsFromMatcher() → 数据库过滤 ID             │
│  ├──> MeiliSearch.search() → 全文检索 + 排序                  │
│  └──> Bookmark.loadMulti() → 通过 ID 批量加载完整数据         │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## 数据读取接口

### Bookmark 模型读取核心

**文件**: `packages/trpc/models/bookmarks.ts`

#### 1. 单个读取 (`fromId`)

```typescript
static async fromId(ctx: AuthedContext, bookmarkId: string, includeContent: boolean) {
  // 1. 数据库查询 + 完整关联预加载
  const bookmark = await ctx.db.query.bookmarks.findFirst({
    where: eq(bookmarks.id, bookmarkId),
    with: {
      tagsOnBookmarks: { with: { tag: true } },
      link: true,
      text: true,
      asset: true,
      assets: true,  // 关联资产文件
    },
  });

  // 2. 访问权限检查
  if (!bookmark || !(await Bookmark.isAllowedToAccessBookmark(ctx, bookmark))) {
    throw new TRPCError({ code: "NOT_FOUND" });
  }

  // 3. 转换为标准化 Schema
  return new Bookmark(ctx, await this.toZodSchema(bookmark, includeContent));
}
```

#### 2. 批量读取 (`loadMulti`)

```typescript
static async loadMulti(ctx: AuthedContext, input: zGetBookmarksRequestSchema) {
  // 1. 构建分页条件
  const conditions = [eq(bookmarks.userId, ctx.user.id)];
  if (input.cursor) {
    conditions.push(input.sortOrder === "asc" 
      ? gt(bookmarks.createdAt, input.cursor.date)
      : lt(bookmarks.createdAt, input.cursor.date));
  }

  // 2. 主查询 + 关联加载
  const results = await ctx.db.query.bookmarks.findMany({
    where: and(...conditions),
    with: {
      tagsOnBookmarks: { with: { tag: true } },
      link: true,
      text: true,
      asset: true,
      assets: true,
    },
    orderBy: input.sortOrder === "asc" ? asc(bookmarks.createdAt) : desc(bookmarks.createdAt),
    limit: (input.limit ?? DEFAULT_NUM_BOOKMARKS_PER_PAGE) + 1,  // +1 判断下一页
  });

  // 3. 下一页 cursor 计算
  const nextCursor = results.length > limit 
    ? { date: results[limit - 1].createdAt, ver: 1 } 
    : null;

  // 4. 批量转换
  const bookmarks = await Promise.all(
    results.slice(0, limit).map(b => this.toZodSchema(b, input.includeContent ?? false))
  );

  return { bookmarks, nextCursor };
}
```

#### 3. 内容加载策略

| 内容类型 | 加载时机 | 说明 |
|---------|---------|------|
| **元数据** | 总是加载 | title, createdAt, source, favourited, archived |
| **标签** | 总是加载 | 关联查询 tagsOnBookmarks + bookmarkTags |
| **关联字段** | 总是加载 | link/text/asset 关联表元数据 |
| **HTML 内容** | `includeContent=true` | 按需从 asset 存储读取 |
| **资产文件** | 按需请求 | 截图、PDF 等通过独立接口下载 |

---

## Web 端分发实现

### 核心组件

**Web 端代码位置**: `apps/web/components/bookmarks/`

#### 1. 列表渲染组件

```typescript
// BookmarkList.tsx - 核心列表
<AnimatedList
  itemLayoutAnimation={LinearTransition}
  renderItem={(b) => <BookmarkCard bookmark={b.item} />}
  data={bookmarks}
  onEndReached={fetchNextPage}  // 无限滚动加载
/>

// BookmarkCard.tsx - 卡片组件
//  根据类型渲染不同预览:
//  ├──> BookmarkLinkPreview (LINK 类型)
//  ├──> BookmarkTextMarkdown (TEXT 类型)
//  └──> BookmarkAssetView (ASSET 类型)
```

#### 2. 查询 Hook

```typescript
// useBookmarkSearch.ts - 搜索状态管理
const { data, fetchNextPage, hasNextPage, isLoading } = useInfiniteQuery({
  queryKey: ["bookmarks", "search", searchParams],
  queryFn: async ({ pageParam }) => {
    return trpcClient.bookmarks.searchBookmarks({
      text: searchParams.text,
      limit: 50,
      cursor: pageParam,
      sortOrder: searchParams.sort,
      includeContent: false,  // 列表不加载内容
    });
  },
  getNextPageParam: (lastPage) => lastPage.nextCursor,
});
```

#### 3. 阅读进度同步

```typescript
// packages/trpc/routers/bookmarks.ts:728-768
updateReadingProgress: bookmarksProcedure
  .input({ bookmarkId, readingProgressOffset, ... })
  .mutation(async ({ input, ctx }) => {
    // UPSERT 模式: 不存在插入，存在则更新
    await ctx.db.insert(userReadingProgress)
      .values({ bookmarkId, userId: ctx.user.id, ... })
      .onConflictDoUpdate({
        target: [userReadingProgress.bookmarkId, userReadingProgress.userId],
        set: { ..., modifiedAt: new Date() },
      });
  })
```

---

## 移动端分发实现

### 核心组件

**移动端代码位置**: `apps/mobile/components/bookmarks/`

#### 1. 响应式列表

```typescript
// BookmarkList.tsx - 移动端优化
<Animated.FlatList
  ref={flatListRef}
  itemLayoutAnimation={LinearTransition}
  contentInsetAdjustmentBehavior="automatic"  // iOS 安全区适配
  contentContainerStyle={{ gap: 15, marginHorizontal: 15 }}
  renderItem={(b) => <BookmarkCard bookmark={b.item} />}
  data={bookmarks}
  onEndReached={fetchNextPage}
  keyExtractor={(b) => b.id}
/>
```

#### 2. 类型特化渲染

```typescript
// BookmarkCard.tsx - 类型分发
switch (bookmark.content.type) {
  case BookmarkTypes.LINK:
    return <BookmarkLinkPreview bookmark={bookmark} />;
  case BookmarkTypes.TEXT:
    return <BookmarkTextMarkdown content={bookmark.content.text} />;
  case BookmarkTypes.ASSET:
    return <BookmarkAssetView bookmark={bookmark} />;
  default:
    return <UnknownBookmarkType />;
}
```

#### 3. 离线阅读支持

```typescript
// ReaderPreview.tsx - 阅读器组件
//  内容预加载策略:
//  1. 列表页预加载相邻 Bookmark 元数据
//  2. 进入详情页时触发 HTML 内容下载
//  3. 本地缓存已加载内容，避免重复请求
const { data: bookmarkWithContent } = trpcClient.bookmarks.getBookmark.useQuery({
  bookmarkId,
  includeContent: true,  // 详情页加载完整内容
});
```

---

## 状态同步与一致性

### 1. 数据库一致性保障

**原子写入事务**:

```typescript
// 创建 Bookmark 时的事务边界
await ctx.db.transaction(async (tx) => {
  // 1. 写入主表
  const [bookmark] = await tx.insert(bookmarks).values({ ... }).returning();
  
  // 2. 写入类型分表
  await tx.insert(bookmarkLinks).values({ id: bookmark.id, url, ... });
  
  // 3. 写入标签关联 (已存在标签 + 新建标签)
  await tx.insert(tagsOnBookmarks).values(
    tagIds.map(tagId => ({ bookmarkId: bookmark.id, tagId, attachedBy: "human" }))
  ).onConflictDoNothing();
});
```

### 2. 搜索索引同步

**异步索引流程**:

```typescript
// SearchIndexingQueue 任务
// apps/workers/workers/searchWorker.ts
async function runIndex(searchClient, bookmarkId, batch) {
  // 1. 重新从数据库加载完整数据
  const bookmark = await db.query.bookmarks.findFirst({
    where: eq(bookmarks.id, bookmarkId),
    with: { link: true, text: true, asset: true, tagsOnBookmarks: ... },
  });

  // 2. 构建索引文档
  const document: BookmarkSearchDocument = {
    id: bookmark.id,
    userId: bookmark.userId,
    // LINK 类型字段
    ...(bookmark.link ? { url: bookmark.link.url, content: extractPlainText(...) } : {}),
    // TEXT 类型字段
    ...(bookmark.text ? { content: bookmark.text.text } : {}),
    // ASSET 类型字段
    ...(bookmark.asset ? { content: bookmark.asset.content } : {}),
    // 公共字段
    title: bookmark.title,
    note: bookmark.note,
    tags: bookmark.tagsOnBookmarks.map(t => t.tag.name),
    createdAt: bookmark.createdAt.toISOString(),
  };

  // 3. 写入 MeiliSearch
  await searchClient.addDocuments([document], { batch });
}
```

**索引触发时机**:

| 触发点 | 代码位置 | 说明 |
|-------|---------|------|
| 创建后 | `routers/bookmarks.ts:375` | `triggerSearchReindex` 异步调用 |
| 更新后 | `routers/bookmarks.ts:626` | 更新所有字段后重新索引 |
| 标签变更 | `routers/bookmarks.ts:1108` | 标签附着/分离后更新 |
| 删除时 | `models/bookmarks.ts:delete()` | 从索引中移除文档 |
| 抓取完成 | `crawlerWorker.ts:complete` | 抓取完成后更新内容索引 |

### 3. 最终一致性模型

```
时间轴 →
│
T0  用户导入 RSS Feed
│   ├──> 数据库写入成功 (立即可读)
│   └──> SearchIndexingQueue 入队
│
T1  ~0-30s
│   ├──> Search Worker 消费任务，写入 MeiliSearch
│   ├──> Crawler 抓取页面 (异步)
│   └──> AI 标签推理 (异步)
│
T2  ~几秒到几分钟
│   ├──> 抓取完成 → 更新 bookmarkLinks → 触发重新索引
│   ├──> 推理完成 → 更新 tagsOnBookmarks → 触发重新索引
│   └──> Webhook 触发 (如果配置)
│
T3  最终状态
    └──> 所有端通过相同接口查询到完整数据
```

### 4. 状态字段设计

| 字段 | 位置 | 状态值 | 说明 |
|-----|-----|-------|------|
| `crawlStatus` | `bookmarkLinks` | `pending`/`success`/`failure` | 抓取状态 |
| `taggingStatus` | `bookmarks` | `pending`/`success`/`failure` | AI 标签状态 |
| `summarizationStatus` | `bookmarks` | `pending`/`success`/`failure` | 摘要状态 |

**前端状态渲染**:

```typescript
// 基于 crawlStatus 渲染不同 UI
switch (bookmark.content.crawlStatus) {
  case "pending":
    return <LoadingSpinner />;
  case "failure":
    return <FailureMessage />;
  case "success":
    return <RenderContent />;
}
```

---

## 端到端时序图

### RSS Feed 导入完整流程

```
┌─────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  ┌──────────┐  ┌─────────┐
│ RSS Feed│  │ Feed Worker  │  │   Database   │  │ TRPC Router │  │ Search   │  │ Web/App │
│ Source  │  │              │  │              │  │              │  │ Worker   │  │ Client  │
└────┬────┘  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘  └────┬─────┘  └────┬────┘
     │              │                   │                   │               │             │
     │ 1. 定时拉取  │                   │                   │               │             │
     │ ────────────>│                   │                   │               │             │
     │              │                   │                   │               │             │
     │ 2. HTTP 响应 │                   │                   │               │             │
     │ <────────────│                   │                   │               │             │
     │              │                   │                   │               │             │
     │              │ 3. 解析 RSS XML   │                   │               │             │
     │              │── ── ── ── ── ── │                   │               │             │
     │              │                   │                   │               │             │
     │              │ 4. 去重查询       │                   │               │             │
     │              │── ── ── ── ── ──>│                   │               │             │
     │              │                   │  SELECT entries   │               │             │
     │              │ <────────── ── ──│                   │               │             │
     │              │                   │                   │               │             │
     │              │ 5. 批量创建       │                   │               │             │
     │              │── ── ── ── ── ──>│                   │               │             │
     │              │                   │  INSERT bookmarks │               │             │
     │              │                   │  INSERT tagsOnBm  │               │             │
     │              │                   │  TRANSACTION COMMIT              │             │
     │              │ <────────── ── ──│                   │               │             │
     │              │                   │                   │               │             │
     │              │ 6. 触发下游队列   │                   │               │             │
     │              │── ── ── ── ── ── │── ── ── ── ── ── │── ── ── ── ──│             │
     │              │                   │                   │ CrawlerQueue  │             │
     │              │                   │                   │ SearchQueue   │             │
     │              │                   │                   │ OpenAIQueue   │             │
     │              │                   │                   │               │             │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                      数据库写入成功，立即对查询可见                                        │
└─────────────────────────────────────────────────────────────────────────────────────────┘
     │              │                   │                   │               │             │
     │              │                   │ 7. 客户端轮询     │               │             │
     │              │                   │ <─────────────────│───────────────│─────────────│
     │              │                   │                   │               │             │
     │              │                   │                   │  getBookmarks │             │
     │              │                   │──────────────────>│               │             │
     │              │                   │                   │               │             │
     │              │                   │  SELECT bookmarks │               │             │
     │              │                   │ <──────────────────               │             │
     │              │                   │──────────────────>               │             │
     │              │                   │                   │ { bookmarks } │             │
     │              │                   │                   │──────────────>│             │
     │              │                   │                   │               │             │
     │              │                   │                   │               │ 8. 渲染列表 │
     │              │                   │                   │               │ <────────── │
     │              │                   │                   │               │             │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                      异步 Worker 后台处理 (与查询并行)                                    │
└─────────────────────────────────────────────────────────────────────────────────────────┘
     │              │                   │                   │               │             │
     │              │              9. Search Worker 索引    │               │             │
     │              │                   │                   │ <─────────────│             │
     │              │                   │  SELECT full data │               │             │
     │              │                   │ <─────────────────               │             │
     │              │                   │──────────────────>               │             │
     │              │                   │                   │  Index doc    │             │
     │              │                   │                   │─────────────> MeiliSearch   │
     │              │                   │                   │               │             │
     │              │              10. Crawler Worker 抓取  │               │             │
     │              │                   │                   │ <─────────────│             │
     │              │                   │  UPDATE link data │               │             │
     │              │                   │ <─────────────────               │             │
     │              │                   │──────────────────>               │             │
     │              │                   │                   │  Re-index    │             │
     │              │                   │                   │─────────────>│             │
     │              │                   │                   │               │             │
     │              │              11. AI Worker 推理       │               │             │
     │              │                   │                   │ <─────────────│             │
     │              │                   │  INSERT tags      │               │             │
     │              │                   │  UPDATE summary   │               │             │
     │              │                   │ <─────────────────               │             │
     │              │                   │──────────────────>               │             │
     │              │                   │                   │  Re-index    │             │
     │              │                   │                   │─────────────>│             │
     │              │                   │                   │               │             │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                      最终一致性达成 (所有数据完整可用)                                    │
└─────────────────────────────────────────────────────────────────────────────────────────┘
     │              │                   │                   │               │             │
     │              │                   │                   │               │ 12. 自动刷新│
     │              │                   │                   │               │ <────────── │
     │              │                   │                   │               │             │
     │              │                   │                   │               │ 显示抓取后 │
     │              │                   │                   │               │ 的完整内容 │
     │              │                   │                   │               │             │
```

### 关键时间节点说明

| 时间点 | 事件 | 用户感知 | 延迟范围 |
|-------|-----|---------|---------|
| **T0** | RSS Feed 拉取开始 | 无感知 | - |
| **T0 + ~1s** | 数据库事务提交完成 | Bookmark 出现在列表中（元数据阶段） | 1-2 秒 |
| **T0 + ~1-30s** | 搜索索引完成 | 可通过搜索找到该 Bookmark | 1-30 秒（取决于队列负载） |
| **T0 + ~5-60s** | 页面抓取完成 | 显示预览截图、HTML 内容 | 5-60 秒 |
| **T0 + ~10-120s** | AI 推理完成 | 显示自动标签、摘要 | 10-120 秒 |

---

## 跨端差异对比

### Web 端 vs 移动端对比矩阵

| 维度 | Web 端 (Next.js + React) | 移动端 (Expo + React Native) |
|-----|-------------------------|-----------------------------|
| **API 接口** | 完全相同 (TRPC HTTP) | 完全相同 (TRPC HTTP) |
| **查询逻辑** | `useInfiniteQuery` (React Query) | `useInfiniteQuery` (TanStack Query) |
| **分页策略** | 无限滚动 + Intersection Observer | 无限滚动 + FlatList onEndReached |
| **内容预加载** | 滚动预加载相邻项 | 进入详情页时加载完整内容 |
| **离线支持** | 浏览器缓存 + Service Worker | 本地 SQLite + 资源预下载 |
| **图片加载** | `<Image>` 组件 + 懒加载 | Expo Image + 优先加载机制 |
| **阅读进度** | 浏览器本地存储 + 服务器同步 | AsyncStorage + 服务器同步 |
| **动画效果** | CSS Transition + Framer Motion | Reanimated (原生驱动) |
| **搜索体验** | 实时输入 + 防抖 | 提交按钮触发搜索 |
| **分享集成** | Web Share API | 原生分享 Sheet |

### 性能优化策略差异

#### Web 端优化:

```typescript
// 虚拟滚动 (大数据量)
<Virtuoso
  totalCount={totalCount}
  itemContent={(index) => <BookmarkCard bookmark={data[index]} />}
  overscan={10}  // 预渲染视口外 10 项
/>

// 图片懒加载
<img
  loading="lazy"
  decoding="async"
  src={screenshotUrl}
/>
```

#### 移动端优化:

```typescript
// 原生 FlatList 优化
<FlatList
  removeClippedSubviews={true}  // 移出视口卸载视图
  maxToRenderPerBatch={10}       // 每批渲染数量
  windowSize={5}                 // 渲染窗口大小
  updateCellsBatchingPeriod={50} // 批量更新间隔
/>

// 图片内存优化
<Image
  contentFit="cover"
  transition={300}
  cachePolicy="memory-disk"  // 双层缓存
/>
```

---

## 总结

### 分发架构核心设计

1. **接口统一**：Web 端和移动端完全共享 TRPC 接口，确保数据一致性
2. **读写分离**：写入通过异步 Worker，读取通过同步数据库查询
3. **最终一致**：搜索索引、抓取内容、AI 标签均异步更新，不阻塞主流程
4. **渐进加载**：元数据先显示，内容、截图、AI 结果逐步到位
5. **乐观 UI**：前端基于状态字段渲染不同阶段的用户界面

### 潜在优化方向

1. **实时推送**：引入 WebSocket/Server-Sent Events 推送状态变更
2. **增量索引**：避免每次全量重新索引，仅更新变更字段
3. **预抓取策略**：基于用户行为预测，提前触发 Feed 拉取
4. **边缘缓存**：CDN 层缓存高频请求，减少数据库压力
5. **离线优先**：移动端先写本地，后台同步到服务器

---
*文档生成时间: 2025-05-13*
*分析基于: apps/workers, packages/trpc, apps/web, apps/mobile 代码库*
