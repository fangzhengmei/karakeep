# Karakeep Feed Worker & Bookmark 创建流程分析

## 目录
1. [概述](#概述)
2. [Feed Worker 流程](#feed-worker-流程)
3. [Bookmark 创建流程](#bookmark-创建流程)
4. [抓取（Crawler）流程](#抓取crawler流程)
5. [AI 推理链路](#ai-推理链路)
6. [异常处理机制](#异常处理机制)
7. [并发控制与队列系统](#并发控制与队列系统)
8. [上下游接口配合](#上下游接口配合)

---

## 概述

Karakeep 使用分布式 worker 架构处理 RSS feed 导入和 bookmark 创建。整个流程涉及多个队列系统和 worker 进程，包括：

- **Feed Worker**: 定期拉取 RSS feed，解析并创建 bookmarks
- **Crawler Worker**: 抓取链接内容，生成截图、PDF 和网页存档
- **Inference Worker**: 执行 AI 任务（标签生成、摘要生成）
- **Webhook Worker**: 触发 webhook 通知
- **Search Index Worker**: 更新搜索索引

---

## Feed Worker 流程

### 1. Feed 调度与分发

**文件**: `packages/trpc/routers/bookmarks.ts:183-288`

```typescript
// 调度策略: 每小时执行一次，基于 feed ID 哈希计算分钟偏移量
function getFeedMinuteOffset(feedId: string): number {
  let hash = 0;
  for (let i = 0; i < feedId.length; i++) {
    hash = (hash << 5) - hash + feedId.charCodeAt(i);
    hash = hash & hash;
  }
  return Math.abs(hash) % 60;
}
```

**调度特点**:
- **均匀分发**: feed 任务按 ID 哈希分散到小时内的不同分钟，避免流量峰值
- **幂等性**: 使用 `idempotencyKey = ${feedId}-${hourlyWindow}` 防止重复调度
- **分组隔离**: 按用户 ID 分组，避免不同用户任务互相影响

### 2. Feed 拉取与解析

**文件**: `apps/workers/workers/feedWorker.ts:133-195`

**执行步骤**:

1. **配额检查**
```typescript
const quotaResult = await QuotaService.canCreateBookmark(db, feed.userId);
if (!quotaResult.result) {
  // 配额不足，跳过拉取
  return;
}
```

2. **HTTP 拉取**
   - 超时控制: 5秒
   - User-Agent 伪装
   - Content-Type 验证 (必须为 XML)

3. **解析 feed 条目**
   - 使用专门的 feed parser 解析 XML
   - 提取字段: guid、link、title、categories

### 3. 去重与创建

**文件**: `apps/workers/workers/feedWorker.ts:207-295`

**去重策略**:
```typescript
// 1. 数据库查询已导入的条目
const exitingEntries = await db.query.rssFeedImportsTable.findMany({
  where: and(
    eq(rssFeedImportsTable.rssFeedId, feed.id),
    inArray(rssFeedImportsTable.entryId, entryGuids)
  ),
});

// 2. 过滤新条目
const newEntries = feedItems.filter(
  (item) => !exitingEntries.some((entry) => entry.entryId === item.guid)
            && item.link && item.guid
);
```

**批量创建**:
```typescript
// 通过 TRPC 客户端批量创建 bookmarks
const createdBookmarks = await Promise.allSettled(
  newEntries.map((item) =>
    trpcClient.bookmarks.createBookmark({
      type: BookmarkTypes.LINK,
      url: item.link!,
      title: item.title,
      source: "rss",
    })
  )
);
```

**标签导入** (可选):
- 启用条件: `feed.importTags === true`
- 将 RSS 条目 categories 作为标签附加到 bookmark

### 4. 状态持久化

**成功状态**:
```typescript
await db.update(rssFeedsTable)
  .set({ 
    lastFetchedStatus: "success", 
    lastFetchedAt: new Date(),
    lastSuccessfulFetchAt: new Date()
  })
  .where(eq(rssFeedsTable.id, feed.id));
```

**失败状态** (在 onError 回调中):
```typescript
await db.update(rssFeedsTable)
  .set({ lastFetchedStatus: "failure", lastFetchedAt: new Date() })
  .where(eq(rssFeedsTable.id, feed.id));
```

---

## Bookmark 创建流程

### 1. 入口与去重

**文件**: `packages/trpc/routers/bookmarks.ts:201-227`

**前置检查**:
- **速率限制**: 每分钟最多 30 次创建
- **去重检查**: 对 LINK 类型的 URL 进行查重

```typescript
async function attemptToDedupLink(ctx: AuthedContext, url: string) {
  const result = await ctx.db
    .select({ id: bookmarkLinks.id })
    .from(bookmarkLinks)
    .leftJoin(bookmarks, eq(bookmarks.id, bookmarkLinks.id))
    .where(and(eq(bookmarkLinks.url, url), eq(bookmarks.userId, ctx.user.id)));
  // ...
}
```

### 2. 数据库事务创建

**文件**: `packages/trpc/routers/bookmarks.ts:229-350`

**事务步骤**:

1. **配额验证**
   - 检查用户 bookmark 数量配额
   - 检查存储配额

2. **主表插入** (`bookmarks` 表)
   ```typescript
   const bookmark = (await tx.insert(bookmarks).values({
     userId: ctx.user.id,
     title: input.title,
     type: input.type,
     archived: input.archived,
     favourited: input.favourited,
     note: input.note,
     summary: input.summary,
     createdAt: input.createdAt,
     source: input.source,
     summarizationStatus: input.type === BookmarkTypes.LINK ? "pending" : null,
   }).returning())[0];
   ```

3. **分表插入** (根据类型):
   - **LINK 类型**: `bookmarkLinks` 表，关联预抓取存档
   - **TEXT 类型**: `bookmarkTexts` 表
   - **ASSET 类型**: `bookmarkAssets` 表

4. **标签处理**
   - 规范化标签名: `normalizeTagName(tagName)`
   - 插入不存在的标签
   - 建立 `tagsOnBookmarks` 关联

### 3. 后续任务调度

**文件**: `packages/trpc/routers/bookmarks.ts:351-450`

创建完成后，异步触发多个下游任务:

| 任务类型 | 队列 | 触发条件 |
|---------|------|---------|
| 链接抓取 | `LinkCrawlerQueue` | LINK 类型且未提供 precrawled 存档 |
| 低优先级抓取 | `LowPriorityCrawlerQueue` | 高创建速率时自动降级 |
| AI 标签生成 | `OpenAIQueue` | 启用自动标签功能 |
| 摘要生成 | `OpenAIQueue` | LINK 类型且启用自动摘要 |
| 搜索索引 | `SearchIndexingQueue` | 所有新创建的 bookmark |
| Webhook 通知 | `WebhookWorker` | 配置了 webhook 的用户 |
| 资产预处理 | `AssetPreprocessingQueue` | ASSET 类型书签 |

**队列优先级动态调整**:
```typescript
// 当创建速率超过阈值时，自动使用低优先级队列
const useLowPriority = await shouldUseLowPriorityQueues(ctx);
const crawlerQueue = useLowPriority ? LowPriorityCrawlerQueue : LinkCrawlerQueue;
const priority = input.crawlPriority ?? 
  (useLowPriority ? QueuePriority.LOW : QueuePriority.MEDIUM);
```

---

## 抓取（Crawler）流程

### 1. Worker 配置

**文件**: `apps/workers/workers/crawlerWorker.ts:330-451`

**并发配置**:
```typescript
{
  concurrency: serverConfig.crawler.numWorkers,
  pollIntervalMs: 1000,
  timeoutSecs: serverConfig.crawler.jobTimeoutSec,
}
```

**浏览器资源管理**:
- **全局浏览器实例**: 复用 Chromium 实例
- **上下文清理**: 超时上下文自动收割 (10分钟超时)
- **互斥锁**: 保护浏览器实例访问

### 2. 页面抓取策略

**文件**: `apps/workers/workers/crawlerWorker.ts:519-1120`

**双模式抓取**:

| 模式 | 触发条件 | 特点 |
|-----|---------|-----|
| **浏览器模式** | 用户启用 browserCrawling | 完整渲染，支持 JavaScript，可生成截图和 PDF |
| **无浏览器模式** | 用户禁用或浏览器不可用 | 纯 HTTP 请求，资源消耗低，速度快 |

**浏览器模式步骤**:
1. **导航验证** - 验证 URL 合法性和访问权限
2. **页面加载** - 等待网络空闲或 5 秒超时
3. **内容提取** - 获取页面 HTML
4. **截图捕获** - 全屏或视口截图 (JPEG 格式)
5. **PDF 生成** - 完整页面 PDF 存档
6. **Banner 图片下载** - 提取并下载 Open Graph 图片

### 3. 内容解析子进程

**文件**: `apps/workers/workers/crawlerWorker.ts:1150-1242`

**设计原因**:
- 避免解析器内存泄漏影响主进程
- 独立内存限制控制
- 超时强制终止

**子进程配置**:
```typescript
{
  cmd: process.execPath, // Node.js 或 tsx (开发环境)
  args: [`--max-old-space-size=${serverConfig.crawler.parserMemLimitMb}`, scriptPath],
  timeout: serverConfig.crawler.parseTimeoutSec * 1000,
  stderr: "inherit",
}
```

**解析输出**:
- 元数据: title、description、publisher、author、published date
- 可读内容: 提取正文 (Readability 算法)
- 媒体信息: 图片、视频 URL

### 4. 资产存储与配额

**文件**: `apps/workers/workers/crawlerWorker.ts:1244-1606`

**资产类型**:
| 资产类型 | 存储路径 | 配额检查 |
|---------|---------|---------|
| 截图 (JPEG) | Asset 存储 | ✓ |
| PDF 存档 | Asset 存储 | ✓ |
| 网页归档 (Monolith) | Asset 存储 | ✓ |
| Banner 图片 | Asset 存储 | ✓ |

**配额控制流程**:
```typescript
// 1. 存储前检查配额
const { data: quotaApproved, error: quotaError } = await tryCatch(
  QuotaService.checkStorageQuota(db, userId, contentSize)
);

// 2. 配额不足时跳过存储
if (quotaError) {
  logger.warn(`Skipping asset storage due to quota exceeded: ${quotaError.message}`);
  return null;
}

// 3. 保存到 asset 存储
await saveAsset({ userId, assetId, metadata, asset, quotaApproved });
```

### 5. 抓取状态管理

**成功路径**:
```typescript
await db.transaction(async (tx) => {
  // 1. 更新 bookmarkLinks 元数据
  await tx.update(bookmarkLinks).set({
    title: metadata.title,
    description: metadata.description,
    publisher: metadata.publisher,
    author: metadata.author,
    imageUrl: metadata.imageUrl,
    favicon: metadata.favicon,
    contentLanguage: metadata.contentLanguage,
    datePublished: metadata.datePublished,
    dateModified: metadata.dateModified,
    htmlContent: truncatedHtml,
    contentAssetId: largeContentAssetId,
    crawlStatusCode: 200,
    crawlStatus: "success",
    crawledAt: new Date(),
  }).where(eq(bookmarkLinks.id, bookmarkId));

  // 2. 关联所有生成的资产
  await tx.insert(assets).values([
    { id: screenshotAssetId, bookmarkId, assetType: AssetTypes.LINK_SCREENSHOT, ... },
    { id: pdfAssetId, bookmarkId, assetType: AssetTypes.LINK_PDF, ... },
    { id: archiveAssetId, bookmarkId, assetType: AssetTypes.LINK_FULL_PAGE_ARCHIVE, ... },
    { id: bannerImageAssetId, bookmarkId, assetType: AssetTypes.LINK_BANNER_IMAGE, ... },
  ]);
});
```

**失败路径** (onError 回调):
```typescript
await db.transaction(async (tx) => {
  // 1. 标记抓取失败
  await tx.update(bookmarkLinks)
    .set({ crawlStatus: "failure" })
    .where(eq(bookmarkLinks.id, bookmarkId));
  
  // 2. 取消待处理的 AI 任务
  await tx.update(bookmarks)
    .set({ taggingStatus: null, summarizationStatus: null })
    .where(eq(bookmarks.id, bookmarkId));
});
```

---

## AI 推理链路

### 1. Inference Worker 配置

**文件**: `apps/workers/workers/inference/inferenceWorker.ts:44-81`

```typescript
{
  concurrency: serverConfig.inference.numWorkers,
  pollIntervalMs: 1000,
  timeoutSecs: serverConfig.inference.jobTimeoutSec,
}
```

### 2. 标签生成流程

**文件**: `apps/workers/workers/inference/tagging.ts`

**前置检查**:
1. 全局开关: `serverConfig.inference.enableAutoTagging`
2. 用户设置: `user.autoTaggingEnabled`
3. 内容可用性: 必须有可分析的内容

**提示词构建**:
```typescript
// 自定义提示词注入
const prompts = await db.query.customPrompts.findMany({
  where: and(
    eq(customPrompts.userId, bookmark.userId),
    inArray(customPrompts.appliesTo, ["all_tagging", "text"])
  ),
});

// 占位符替换: $tags, $aiTags, $userTags
const promptTexts = await replaceTagsPlaceholders(prompts.map(p => p.text), userId);
```

**内容类型适配**:
| Bookmark 类型 | 推理策略 |
|--------------|---------|
| LINK | 提取 HTML 纯文本，构建摘要提示 |
| TEXT | 直接使用文本内容 |
| ASSET (图片) | 多模态模型，图片转标签 |
| ASSET (PDF) | 提取 PDF 文本内容 |

**JSON 输出解析**:
```typescript
function parseJsonFromLLMResponse(response: string): unknown {
  // 1. 尝试直接解析 JSON
  // 2. 提取 Markdown 代码块中的 JSON
  // 3. 在文本中查找 { ... } 结构
  // 4. 最后尝试再次解析以获取原始错误
}
```

**标签规范化与去重**:
```typescript
// 1. 去除 hashtag 符号
tags = tags.map(t => t.startsWith('#') ? t.slice(1) : t.trim());

// 2. 数据库事务处理
await db.transaction(async (tx) => {
  // a. 匹配已有标签
  const matchedTags = await tx.query.bookmarkTags.findMany(...);
  
  // b. 创建新标签
  const newTagIds = (await tx.insert(bookmarkTags)
    .values(notFoundTagNames.map(name => ({ name, userId })))
    .onConflictDoNothing()
    .returning()).map(t => t.id);
  
  // c. 删除旧的 AI 标签
  await tx.delete(tagsOnBookmarks).where(
    and(eq(tagsOnBookmarks.attachedBy, "ai"), eq(tagsOnBookmarks.bookmarkId, bookmarkId))
  );
  
  // d. 附加新标签
  await tx.insert(tagsOnBookmarks).values(allTagIds.map(tagId => ({
    tagId, bookmarkId, attachedBy: "ai" as const
  }))).onConflictDoNothing();
});
```

### 3. 摘要生成流程

**文件**: `apps/workers/workers/inference/summarize.ts:47-191`

**内容截断策略**:
```typescript
// 构建提示词时考虑上下文窗口限制
const summaryPrompt = await buildSummaryPrompt(
  userSettings?.inferredTagLang ?? serverConfig.inference.inferredTagLang,
  customPrompts,
  textToSummarize,
  serverConfig.inference.contextLength // 模型上下文窗口限制
);
```

**输出与持久化**:
```typescript
const summaryResult = await inferenceClient.inferFromText(summaryPrompt, {
  schema: null, // 摘要是自由文本，无需 JSON Schema
  abortSignal: job.abortSignal,
});

await db.update(bookmarks)
  .set({ summary: summaryResult.response, modifiedAt: new Date() })
  .where(eq(bookmarks.id, bookmarkId));
```

---

## 异常处理机制

### 1. 队列级错误处理

**通用错误回调模式**:
```typescript
{
  onError: async (job) => {
    // 1. 指标统计
    workerStatsCounter.labels(workerType, "failed").inc();
    
    // 2. 重试耗尽时标记永久失败
    if (job.numRetriesLeft == 0) {
      workerStatsCounter.labels(workerType, "failed_permanent").inc();
      // 执行失败清理逻辑
    }
    
    // 3. 错误日志
    logger.error(`[${workerType}][${job.id}] Job failed: ${job.error}\n${job.error.stack}`);
  }
}
```

### 2. 超时与中止

**多级超时控制**:
```
┌─────────────────────────────────────────────────────────┐
│  Worker 级超时 (jobTimeoutSec)                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │  页面导航超时 (navigateTimeoutSec)                │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  页面加载等待 (5s)                         │  │  │
│  │  │  截图超时 (screenshotTimeoutSec)           │  │  │
│  │  │  解析子进程超时 (parseTimeoutSec)          │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**AbortSignal 传播**:
```typescript
// 1. Worker 级 abortSignal 传递到子操作
const response = await fetchWithProxy(url, {
  signal: AbortSignal.any([AbortSignal.timeout(5000), job.abortSignal]),
}, proxyConfig);

// 2. 中止时的资源清理
job.abortSignal.addEventListener('abort', () => {
  page.unrouteAll(); // 取消请求拦截
  cdpSession?.detach(); // 断开 CDP 会话
}, { once: true });
```

### 3. 部分失败处理

**Promise.allSettled 模式**:
```typescript
// Feed 批量导入时，单个失败不影响整体
const createdBookmarks = await Promise.allSettled(
  newEntries.map(item => trpcClient.bookmarks.createBookmark(...))
);

// 标签附加时容忍单个失败
await Promise.allSettled(
  newEntries.map(async (item, idx) => {
    const bookmark = createdBookmarks[idx];
    if (bookmark.status === "fulfilled" && item.categories) {
      try {
        await trpcClient.bookmarks.updateTags(...);
      } catch (error) {
        logger.warn(`Failed to attach tags: ${error}`);
        // 不抛出，继续执行
      }
    }
  })
);
```

### 4. 降级策略

**浏览器降级**:
```typescript
// 浏览器不可用时自动降级为 HTTP 模式
let browser: Browser | undefined;
try {
  browser = await getBrowserInstance();
} catch {
  return browserlessCrawlPage(jobId, url, abortSignal, proxyConfig);
}
```

**代理降级**:
```typescript
// 代理失败时尝试直连
const proxyConfig = selectRunProxies();
const response = await tryCatch(fetchWithProxy(url, {}, proxyConfig));
if (response.error && matchesNoProxy(url, proxyConfig.noProxy)) {
  return fetchDirect(url);
}
```

---

## 并发控制与队列系统

### 1. 队列配置

**核心队列列表**:
| 队列名称 | 用途 | 并发配置 |
|---------|-----|---------|
| `FeedQueue` | RSS feed 拉取 | concurrency: 1 |
| `LinkCrawlerQueue` | 链接抓取 | 可配置 numWorkers |
| `LowPriorityCrawlerQueue` | 低优先级抓取 | 独立并发 |
| `OpenAIQueue` | AI 推理 | 可配置 numWorkers |
| `SearchIndexingQueue` | 搜索索引 | 后台执行 |
| `WebhookWorker` | Webhook 通知 | 异步触发 |
| `AssetPreprocessingQueue` | 资产预处理 | 异步触发 |

### 2. 优先级与分组

**任务优先级**:
```typescript
enum QueuePriority {
  HIGH = 1,    // 用户手动触发
  MEDIUM = 2,  // 正常导入
  LOW = 3,     // 批量导入、后台任务
}
```

**用户级分组**:
```typescript
// 任务按用户 ID 分组，实现公平调度
FeedQueue.enqueue({ feedId }, {
  groupId: userId,  // 同一用户的任务串行执行
  delayMs: computedDelay,
  idempotencyKey,
});
```

### 3. 速率限制

**双层限流策略**:
```typescript
// 1. 常规速率限制 (每分钟 30 次)
createRateLimitMiddleware({
  name: "bookmarks.createBookmark",
  windowMs: 60 * 1000,
  maxRequests: 30,
});

// 2. 高容量检测 (5 分钟 30 次 → 自动降级)
const highVolumeResult = await rateLimitClient.checkRateLimit({
  name: "bookmarks.createBookmark.highVolume",
  windowMs: 5 * 60 * 1000,
  maxRequests: 30,
}, userId);
```

---

## 上下游接口配合

### 1. 上游调用链

**调用入口**:
```
Web 前端 (Next.js)
    ↓
TRPC API (packages/trpc/routers/bookmarks.ts)
    ↓
业务模型 (packages/trpc/models/bookmarks.ts)
    ↓
队列入队 (LinkCrawlerQueue / OpenAIQueue)
```

**TRPC  impersonation 模式**:
```typescript
// Feed Worker 使用 impersonation 调用用户 API
const trpcClient = await buildImpersonatingTRPCClient(feed.userId);
await trpcClient.bookmarks.createBookmark(...);
```

### 2. 下游通知链

**任务完成后的扇出**:
```
Crawler 完成
    ├─→ 触发 AI 标签任务 (OpenAIQueue)
    ├─→ 触发 AI 摘要任务 (OpenAIQueue)
    ├─→ 更新搜索索引 (SearchIndexingQueue)
    └─→ 触发 Webhook 通知 (WebhookWorker)
            └─→ 外部系统集成 (Zapier、自定义 Webhook)
```

### 3. 数据库一致性保证

**外键关联设计**:
```
bookmarks (主表)
    ├─→ bookmarkLinks (1:1)  - 链接元数据
    ├─→ bookmarkTexts (1:1)  - 文本内容
    ├─→ bookmarkAssets (1:1) - 资产元数据
    ├─→ assets (1:N)         - 关联资产文件
    └─→ tagsOnBookmarks (1:N) - 标签关联
           └─→ bookmarkTags (N:1)
```

**事务边界**:
- **Bookmark 创建**: 单个数据库事务，原子性保证
- **Crawler 更新**: 多表更新在同一事务中
- **AI 标签更新**: 标签替换是原子事务

### 4. 搜索索引同步

**触发时机**:
1. Bookmark 创建时
2. AI 标签/摘要生成完成后
3. Bookmark 更新时
4. Bookmark 删除时

**索引内容**:
- 标题、描述、URL
- 全文内容 (截断后)
- 标签列表
- AI 生成的摘要

---

## 总结

### 关键设计特点

1. **分布式异步架构**: 所有耗时操作通过队列异步执行，避免阻塞用户请求
2. **多层重试与降级**: 从网络请求到浏览器抓取都有完善的重试和降级策略
3. **配额与限流保护**: 用户级配额、速率限制、并发控制三重保护
4. **状态机管理**: 清晰的状态流转 (pending → success/failure)
5. **幂等性设计**: 通过 idempotencyKey 避免重复执行

### 潜在优化点

1. **任务优先级动态调整**: 根据用户行为实时调整队列优先级
2. **失败任务死信队列**: 永久失败任务的审计与重试机制
3. **工作窃取调度**: 跨 worker 的负载均衡
4. **渐进式内容抓取**: 先抓取元数据，后台异步抓取完整内容

---
*文档生成时间: 2025-05-13*
