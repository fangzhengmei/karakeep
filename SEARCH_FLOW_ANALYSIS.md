# Karakeep 搜索流程深度分析

本文档详细梳理了 Karakeep 项目中从收藏内容写入、索引构建到搜索结果返回的完整技术实现流程。

---

## 一、系统架构概览

Karakeep 采用**插件化架构**，搜索引擎通过 `PluginManager` 动态加载，当前默认实现为 **Meilisearch**。

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  内容写入层     │────▶│  队列调度层     │────▶│  索引构建层     │
│  (tRPC API)    │     │  (BullMQ)      │     │  (Meilisearch)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                          │
                                                          ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  结果返回层     │◀────│  结果处理层     │◀────│  查询执行层     │
│  (UI 渲染)      │     │  (排序/权限)    │     │  (搜索查询)     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

**核心模块位置：**
- 搜索接口定义：`packages/shared/search.ts`
- Meilisearch 插件：`packages/plugins/search-meilisearch/src/index.ts`
- 搜索过滤逻辑：`packages/trpc/lib/search.ts`
- 搜索 Worker：`apps/workers/workers/searchWorker.ts`
- 查询解析器：`packages/shared/searchQueryParser.ts`
- 队列定义：`packages/shared-server/src/queues.ts`

---

## 二、收藏内容写入流程

### 2.1 写入入口

书签创建的主入口是 `bookmarksAppRouter.createBookmark` 过程 (`packages/trpc/routers/bookmarks.ts:184-452`)。

**关键步骤：**

```typescript
// 1. 事务写入数据库
const bookmark = await ctx.db.transaction(async (tx) => {
  // 检查配额
  const quotaResult = await QuotaService.canCreateBookmark(tx, ctx.user.id);
  // 写入 bookmarks 主表
  const bookmark = (await tx.insert(bookmarks).values({...}).returning())[0];
  // 根据类型写入子表 (bookmarkLinks / bookmarkTexts / bookmarkAssets)
  // ...
});

// 2. 触发后续异步任务
await Promise.all([
  RuleEngine.triggerOnEvent(...),
  triggerSearchReindex(bookmark.id, enqueueOpts),  // 🔍 触发索引
  new WebhooksService(ctx.db).triggerWebhook(...),
]);
```

### 2.2 触发重新索引的事件

索引更新通过 `triggerSearchReindex` 函数统一触发 (`packages/shared-server/src/queues.ts:189-203`)，该函数将索引任务加入 `SearchIndexingQueue` 队列。

**触发索引的完整事件列表：**

| 事件类型 | 触发位置 | 说明 |
|---------|---------|------|
| **创建书签** | `bookmarks.ts:443` | 创建后立即触发 |
| **更新书签** | `bookmarks.ts:626` | 更新元数据、内容后触发 |
| **更新书签文本** | `bookmarks.ts:677` | 废弃 API，仍兼容 |
| **删除书签** | `bookmarks.ts:934` | 触发 `type: "delete"` 操作 |
| **更新标签** | `bookmarks.ts:1148` | 标签增删后触发 |
| **生成摘要** | `bookmarks.ts:1322` | AI 摘要生成后触发 |
| **爬虫完成** | `crawlerWorker.ts:2317` | 页面抓取完成后触发 |
| **AI 打标完成** | `tagging.ts:596` | 自动标签生成后触发 |
| **资产预处理完成** | `assetPreprocessingWorker.ts:480` | OCR/PDF 解析后触发 |
| **摘要生成完成** | `summarize.ts:188` | 摘要任务完成后触发 |
| **管理员单条重索引** | `admin.ts:700` | 手动触发 |
| **管理员全量重索引** | `admin.ts:248-263` | 先清空再逐条重建 |

**幂等性保证：**
```typescript
// triggerSearchReindex 使用 idempotencyKey 防止重复入队
await SearchIndexingQueue.enqueue(
  { bookmarkId, type: "index" },
  {
    ...opts,
    idempotencyKey: `index:${bookmarkId}`,  // 同一书签的索引任务会去重
  }
);
```

---

## 三、索引构建机制

### 3.1 增量索引（默认模式）

**增量索引是系统的默认工作模式**，每次仅处理单个书签的索引更新。

**流程：**
1. `triggerSearchReindex(bookmarkId)` 入队
2. `SearchIndexingWorker` 消费队列 (`searchWorker.ts:23-176`)
3. 从数据库读取完整书签数据，构建 `BookmarkSearchDocument`
4. 调用 `searchClient.addDocuments([document])`

**文档结构定义** (`packages/shared/search.ts:5-23`)：
```typescript
interface BookmarkSearchDocument {
  id: string;
  userId: string;                    // 用于权限过滤
  url?: string;
  title?: string;
  linkTitle?: string;
  description?: string;
  content?: string;                  // 纯文本内容（搜索主要字段）
  metadata?: string;
  fileName?: string;
  createdAt?: string;                // 用于排序
  note?: string;
  summary?: string;
  tags: string[];
  publisher?: string;
  author?: string;
  datePublished?: Date;
  dateModified?: Date;
}
```

**文档构建逻辑** (`searchWorker.ts:89-119`)：
```typescript
const document: BookmarkSearchDocument = {
  id: bookmark.id,
  userId: bookmark.userId,
  ...(bookmark.link ? {
    url: bookmark.link.url,
    linkTitle: bookmark.link.title,
    description: bookmark.link.description,
    content: await Bookmark.getBookmarkPlainTextContent(  // HTML → 纯文本
      bookmark.link, bookmark.userId
    ),
    publisher: bookmark.link.publisher,
    author: bookmark.link.author,
  } : {}),
  ...(bookmark.asset ? {
    content: bookmark.asset.content,    // OCR 提取的文本
    metadata: bookmark.asset.metadata,
  } : {}),
  ...(bookmark.text ? { content: bookmark.text.text } : {}),
  tags: bookmark.tagsOnBookmarks.map((t) => t.tag.name),
};
```

### 3.2 增量与全量索引的界限

| 维度 | 增量索引 | 全量索引 |
|------|---------|---------|
| **触发方式** | 事件驱动（内容变更时自动触发） | 管理员手动调用 `reindexAllBookmarks` |
| **触发入口** | `triggerSearchReindex(bookmarkId)` | `admin.ts:248-263` |
| **数据范围** | 单条书签 | 所有书签 |
| **前置操作** | 无 | 先调用 `clearIndex()` 清空整个索引 |
| **队列优先级** | `QueuePriority.Default` (0) | `QueuePriority.Low` (50) |
| **并发控制** | 正常并发 | 低优先级，不影响正常业务 |

**全量索引实现：**
```typescript
// admin.ts:248-263
reindexAllBookmarks: adminBookmarksProcedure.mutation(async ({ ctx }) => {
  const searchIdx = await getSearchClient();
  await searchIdx?.clearIndex();                    // 先清空索引
  const bookmarkIds = await ctx.db.query.bookmarks.findMany({
    columns: { id: true },
  });
  await Promise.all(
    bookmarkIds.map((b) =>
      triggerSearchReindex(b.id, {
        priority: QueuePriority.Low,                // 低优先级
      }),
    ),
  );
});
```

### 3.3 批量处理机制（Meilisearch 插件内部）

Meilisearch 插件实现了**客户端级别的批量队列** (`BatchingDocumentQueue`)，但这是**请求合并**而非全量索引。

**设计要点** (`search-meilisearch/src/index.ts:46-207`)：

1. **自动批处理**：
   - 积累到 `MEILI_BATCH_SIZE` 条或超时 `MEILI_BATCH_TIMEOUT_MS` 后批量发送
   - 使用 Mutex 保证线程安全

2. **操作去重**：
   ```typescript
   // 同一文档的多次操作只保留最后一次
   const lastOpIndexByDocId = new Map<string, number>();
   for (let i = 0; i < this.pendingOperations.length; i++) {
     const docId = op.type === "add" ? op.document.id : op.id;
     lastOpIndexByDocId.set(docId, i);  // 记录每个文档的最后操作
   }
   ```

3. **重试策略**：
   ```typescript
   // searchWorker.ts:160
   const batch = job.runNumber === 0;  // 首次执行启用批量，重试时禁用
   // 重试时直接发送，不经过批量队列，提高可靠性
   ```

---

## 四、搜索查询流程

### 4.1 查询入口

搜索主入口是 `bookmarksAppRouter.searchBookmarks` (`bookmarks.ts:804-902`)。

### 4.2 高级查询语法解析

用户输入的查询字符串首先经过 `parseSearchQuery` 解析器 (`packages/shared/searchQueryParser.ts`)，分离出**全文搜索文本**和**结构化过滤条件**。

**支持的查询语法：**
```
#tag:javascript           # 按标签过滤
list:reading              # 按列表过滤
is:archived               # 按状态过滤 (archived/fav/tagged/inlist/link/text/media/broken)
url:github.com            # 按 URL 过滤
title:react               # 按标题过滤
source:extension          # 按来源过滤
after:2024-01-01          # 按日期过滤
age:7d                    # 按相对时间过滤
feed:tech-news            # 按 RSS 订阅过滤
-tag:deprecated           # 反向过滤（排除）
"exact phrase"            # 精确短语
(term1 OR term2)          # 逻辑组合
```

**解析器实现原理：**
- 使用 `typescript-parsec` 解析器组合子库
- 词法分析器 (Lexer) 将输入转换为 Token 流
- 语法分析器 (Parser) 构建表达式树
- 支持 `AND`（默认，空格分隔）和 `OR` 逻辑

**解析结果结构：**
```typescript
interface TextAndMatcher {
  text: string;              // 剩余的全文搜索文本
  matcher?: Matcher;         // 结构化过滤条件 AST
}

// Matcher 类型示例
type Matcher =
  | { type: "tagName"; tagName: string; inverse?: boolean }
  | { type: "and"; matchers: Matcher[] }
  | { type: "or"; matchers: Matcher[] }
  // ... 更多类型
```

### 4.3 过滤条件执行

结构化过滤条件通过 `getBookmarkIdsFromMatcher` 函数在**数据库层面**执行 (`packages/trpc/lib/search.ts:441-448`)。

**执行策略：**

1. **每种过滤条件独立查询**，返回符合条件的书签 ID 列表
2. **组合操作**：`AND` 取交集 (`intersect`)，`OR` 取并集 (`union`)
3. **最终结果**是一个书签 ID 数组，传给搜索引擎作为过滤条件

**关键代码示例：**
```typescript
// search.ts:40-69 - 交集实现
function intersect(vals: BookmarkQueryReturnType[][]): BookmarkQueryReturnType[] {
  const countMap = new Map<string, number>();
  for (const arr of vals) {
    for (const item of arr) {
      countMap.set(item.id, (countMap.get(item.id) ?? 0) + 1);
    }
  }
  // 出现在所有子查询中的 ID 才保留
  return Array.from(countMap.entries())
    .filter(([_, count]) => count === vals.length)
    .map(([id]) => map.get(id)!);
}
```

### 4.4 搜索引擎查询

数据库过滤得到的 ID 列表和用户 ID 一起传给 Meilisearch 执行全文搜索。

**搜索调用：**
```typescript
// bookmarks.ts:831-860
let filter: FilterQuery[];
if (parsedQuery.matcher) {
  const bookmarkIds = await getBookmarkIdsFromMatcher(ctx, parsedQuery.matcher);
  filter = [
    { type: "in", field: "id", values: bookmarkIds },  // 数据库过滤结果
    { type: "eq", field: "userId", value: ctx.user.id },  // 🔒 权限过滤
  ];
} else {
  filter = [{ type: "eq", field: "userId", value: ctx.user.id }];
}

const resp = await client.search({
  query: parsedQuery.text,
  filter,
  sort: [{ field: "createdAt", order: createdAtSortOrder }],
  limit: input.limit,
  ...(input.cursor ? { offset: input.cursor.offset } : {}),
});
```

**Meilisearch 查询参数：**
```typescript
// search-meilisearch/src/index.ts:262-281
const result = await this.index.search(options.query, {
  filter: options.filter?.map((f) => filterToMeiliSearchFilter(f)),
  limit: options.limit,
  offset: options.offset,
  sort: options.sort?.map((s) => `${s.field}:${s.order}`),
  attributesToRetrieve: ["id"],                    // 只取回 ID
  showRankingScore: true,                           // 返回相关性分数
  matchingStrategy: "all",                          // 所有词必须匹配
});
```

### 4.5 结果聚合与排序

**两阶段查询架构：**

```
阶段1: Meilisearch 返回 → [ {id: "b1", score: 0.95}, {id: "b2", score: 0.87} ]
                           (仅包含 ID 和相关性分数)
                                    │
                                    ▼
阶段2: 数据库加载     →  SELECT * FROM bookmarks WHERE id IN (...)
                                    │
                                    ▼
阶段3: 应用层排序     →  按用户选择的排序策略重新排序
```

**排序策略** (`bookmarks.ts:880-890`)：
```typescript
switch (true) {
  case sortOrder === "relevance":
    // 按搜索引擎返回的相关性分数降序
    results.sort((a, b) => idToRank[b.id] - idToRank[a.id]);
    break;
  case sortOrder === "desc":
    // 按创建时间降序（最新优先）
    results.sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime());
    break;
  case sortOrder === "asc":
    // 按创建时间升序（最早优先）
    results.sort((a, b) => a.createdAt.getTime() - b.createdAt.getTime());
    break;
}
```

**关键决策点：为什么分两阶段查询？**
1. Meilisearch 索引只存储搜索字段，不存储完整业务数据
2. 减少搜索引擎的数据冗余和同步成本
3. 数据库是数据的可信来源，确保最新状态
4. 方便应用层进行权限校验和数据转换

---

## 五、高亮显示实现

### 5.1 当前状态

**当前代码库中没有实现搜索结果关键词高亮功能。**

证据：
```typescript
// Meilisearch 查询时只请求 id 字段
attributesToRetrieve: ["id"],  // search-meilisearch/src/index.ts:268

// 没有请求高亮参数
// 缺失: attributesToHighlight, highlightPreTag, highlightPostTag

// 搜索返回结果定义中也没有高亮字段
interface SearchResult {
  id: string;
  score?: number;  // 只有 ID 和分数
}
```

### 5.2 可扩展点

如果需要实现高亮，需要修改以下位置：

1. **Meilisearch 查询参数** (`search-meilisearch/src/index.ts:262-271`)：
   ```typescript
   // 需要添加
   attributesToHighlight: ["title", "content", "description"],
   highlightPreTag: "<mark>",
   highlightPostTag: "</mark>",
   showMatchesPosition: true,
   ```

2. **SearchResult 接口** (`packages/shared/search.ts:43-46`)：
   ```typescript
   interface SearchResult {
     id: string;
     score?: number;
     highlights?: Record<string, string>;  // 新增高亮字段
     matchesPosition?: Record<string, Array<{start: number, length: number}>>;
   }
   ```

3. **搜索响应处理** (`search-meilisearch/src/index.ts:273-280`)：
   ```typescript
   return {
     hits: result.hits.map((hit) => ({
       id: hit.id,
       score: hit._rankingScore,
       highlights: hit._formatted,  // 传递高亮结果
       matchesPosition: hit._matchesPosition,
     })),
     // ...
   };
   ```

---

## 六、权限过滤层级

系统采用**三层权限过滤机制**，确保数据安全：

### 第一层：搜索引擎级过滤（查询时）

**位置：** `bookmarks.ts:831-843`

在搜索请求中强制加入 `userId` 过滤条件，在搜索引擎层面就排除其他用户的数据。

```typescript
filter = [
  { type: "eq", field: "userId", value: ctx.user.id },  // 🔒 强制过滤
];
```

**Meilisearch 配置：**
```typescript
// search-meilisearch/src/index.ts:364
const desiredFilterableAttributes = ["id", "userId"].sort();
// userId 被配置为可过滤字段，确保可以在查询时过滤
```

### 第二层：数据加载时权限校验

**位置：** `packages/trpc/models/bookmarks.ts:122-131`

从数据库加载书签数据时，`BareBookmark.isAllowedToAccessBookmark` 方法进行二次校验。

```typescript
protected static async isAllowedToAccessBookmark(
  ctx: AuthedContext,
  { id: bookmarkId, userId: bookmarkOwnerId }: { id: string; userId: string },
): Promise<boolean> {
  // 所有者直接通过
  if (bookmarkOwnerId == ctx.user.id) {
    return true;
  }
  // 协作者：检查是否在有访问权限的共享列表中
  const bookmarkLists = await List.forBookmark(ctx, bookmarkId);
  return bookmarkLists.some((l) => l.canUserView());
}
```

### 第三层：数据返回时字段过滤

**位置：** `packages/trpc/models/bookmarks.ts:753-766`

返回数据前，根据访问者身份进行字段脱敏。

```typescript
asZBookmark(): ZBookmark {
  if (this.bookmark.userId === this.ctx.user.id) {
    return this.bookmark;  // 所有者看到完整数据
  }
  // 协作者看不到敏感字段
  return {
    ...this.bookmark,
    archived: false,        // 屏蔽归档状态
    favourited: false,      // 屏蔽收藏状态
    note: null,             // 屏蔽个人笔记
  };
}
```

### 权限过滤总结

| 层级 | 位置 | 过滤逻辑 | 粒度 |
|------|------|---------|------|
| **L1 搜索引擎** | `bookmarks.ts:842` | `userId = currentUser` | 行级（排除其他用户数据） |
| **L2 数据库加载** | `bookmarks.ts:122-131` | 所有权 + 共享列表权限 | 行级（校验单条访问权限） |
| **L3 字段脱敏** | `bookmarks.ts:753-766` | 非所有者隐藏敏感字段 | 字段级（数据脱敏） |

---

## 七、关键设计决策分析

### 7.1 为什么采用异步索引而不是同步更新？

**设计：** 索引更新通过队列异步处理，不阻塞用户请求

**理由：**
1. **响应速度**：文档构建可能涉及 HTML → 纯文本转换、OCR 内容提取等耗时操作
2. **可靠性**：队列支持重试（最多 5 次），失败任务不会影响主流程
3. **弹性**：批量索引或高并发时可通过队列削峰填谷
4. **解耦**：搜索引擎故障不影响核心业务流程

**权衡：**
- ✅ 写入延迟低（~几十毫秒）
- ❌ 索引存在短暂延迟（通常几秒到几十秒）
- ❌ 用户可能看到"已创建但搜索不到"的短暂不一致

### 7.2 为什么数据库过滤 + 搜索引擎过滤的混合架构？

**设计：** 结构化过滤（标签、列表、日期等）在数据库执行，全文搜索在搜索引擎执行

**理由：**
1. **表达能力**：数据库可以处理复杂的关联查询（如智能列表、标签权限）
2. **数据一致性**：数据库是事实来源，过滤结果准确
3. **搜索引擎能力边界**：Meilisearch 不擅长处理复杂的关联逻辑
4. **性能**：ID 列表过滤在搜索引擎中是高效的数值操作

**关键代码：**
```typescript
// 先在数据库过滤得到 ID 列表
const bookmarkIds = await getBookmarkIdsFromMatcher(ctx, parsedQuery.matcher);

// 再传给搜索引擎作为过滤条件
filter = [
  { type: "in", field: "id", values: bookmarkIds },
  { type: "eq", field: "userId", value: ctx.user.id },
];
```

### 7.3 为什么索引文档中要冗余存储 tags 而不是只存 ID？

**设计：** `BookmarkSearchDocument.tags` 存储标签名称数组，而不是标签 ID

**理由：**
1. **搜索友好**：用户搜索的是标签名称（如 `#javascript`），不是数据库内部 ID
2. **性能**：避免搜索时的表关联，搜索引擎可以直接匹配标签名
3. **更新成本**：标签名称很少变化，更新成本可接受

**同步机制：** 标签更新时 (`bookmarks.ts:1148`) 会触发重新索引，确保数据一致。

### 7.4 为什么批量处理仅在首次执行时启用？

**设计：** `batch = job.runNumber === 0`，重试时禁用批量

**理由：**
1. **首次执行**：批量可显著减少 Meilisearch 的任务数，提高吞吐量
2. **重试场景**：批量中的单个文档失败会导致整个批次重试，浪费资源
3. **可靠性优先**：重试时应该直接发送单个任务，避免被其他文档牵连

---

## 八、数据流时序图

### 8.1 写入 → 索引流程

```
用户创建书签
    │
    ▼
tRPC API (createBookmark)
    ├─ 写入数据库（事务）
    ├─ 触发爬虫/推理等异步任务
    └─ triggerSearchReindex(bookmarkId)
            │
            ▼
    SearchIndexingQueue ────────────┐
            │                       │ 队列持久化
            ▼                       │
SearchIndexingWorker.run()          │
    ├─ 从 DB 读取完整书签数据        │
    ├─ 构建 SearchDocument          │
    └─ searchClient.addDocuments()  │
            │                       │
            ▼                       │
  Meilisearch BatchingQueue         │
    ├─ 积累到 batchSize 或超时      │
    ├─ 去重（同一文档只留最后操作） │
    └─ 批量发送到 Meilisearch        │
            │                       │
            ▼                       │
  Meilisearch 内部索引更新 ◀─────────┘
```

### 8.2 搜索查询流程

```
用户输入查询 "react #tutorial after:2024-01-01"
    │
    ▼
parseSearchQuery()
    ├─ text: "react"
    └─ matcher: AND(
          tagName: "tutorial",
          dateAfter: "2024-01-01"
       )
    │
    ▼
getBookmarkIdsFromMatcher()
    ├─ 查询 tag = tutorial 的书签 → [id1, id2, id3]
    ├─ 查询 date > 2024-01-01 的书签 → [id2, id3, id4]
    └─ 取交集 → [id2, id3]
    │
    ▼
Meilisearch.search()
    query: "react"
    filter: [id IN [id2, id3], userId = currentUser]
    sort: createdAt:desc
    │
    ▼
返回结果: [{id: "id3", score: 0.92}, {id: "id2", score: 0.85}]
    │
    ▼
Bookmark.loadMulti([id3, id2])
    ├─ 从 DB 读取完整数据
    └─ isAllowedToAccessBookmark() 权限校验
    │
    ▼
按相关性重新排序 → [id3, id2]
    │
    ▼
asZBookmark() 字段脱敏
    │
    ▼
返回给前端
```

---

## 九、关键文件索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 搜索接口定义 | `packages/shared/search.ts` | 全文件 |
| Meilisearch 插件 | `packages/plugins/search-meilisearch/src/index.ts` | 全文件 |
| 批量队列实现 | `packages/plugins/search-meilisearch/src/index.ts` | 46-207 |
| 搜索 Worker | `apps/workers/workers/searchWorker.ts` | 全文件 |
| 触发索引函数 | `packages/shared-server/src/queues.ts` | 189-203 |
| 查询解析器 | `packages/shared/searchQueryParser.ts` | 全文件 |
| 数据库过滤逻辑 | `packages/trpc/lib/search.ts` | 全文件 |
| 搜索 API 路由 | `packages/trpc/routers/bookmarks.ts` | 804-902 |
| 创建书签触发索引 | `packages/trpc/routers/bookmarks.ts` | 431-451 |
| 权限校验 | `packages/trpc/models/bookmarks.ts` | 122-131 |
| 字段脱敏 | `packages/trpc/models/bookmarks.ts` | 753-766 |
| 全量重索引 | `packages/trpc/routers/admin.ts` | 248-263 |
| 搜索结果排序 | `packages/trpc/routers/bookmarks.ts` | 880-890 |

---

## 十、潜在优化点

1. **高亮显示**：当前未实现，可按 5.2 节所述扩展
2. **搜索缓存**：高频查询可考虑增加缓存层（如 Redis）
3. **分页优化**：当前使用 offset 分页，大数据量时可改为 keyset 分页
4. **同义词支持**：可配置 Meilisearch 同义词词典提升召回率
5. **中文分词**：Meilisearch 默认中文分词效果一般，可考虑接入更专业的中文分词器
6. **索引监控**：增加索引延迟、失败率等监控指标
