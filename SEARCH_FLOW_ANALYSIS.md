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

## 二、收藏内容写入与索引触发流程

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

### 2.2 触发重新索引的完整事件列表

索引更新主要通过两种方式触发：
1. **`triggerSearchReindex` 函数** (`packages/shared-server/src/queues.ts:189-203`)：触发 `type: "index"` 操作
2. **直接入队**：仅删除书签时使用，触发 `type: "delete"` 操作

| 事件类型 | 触发位置 | 索引类型 | 说明 |
|---------|---------|---------|------|
| **书签生命周期** | | | |
| 创建书签 | `packages/trpc/routers/bookmarks.ts:443` | `index` | 创建后立即触发 |
| 更新书签 | `packages/trpc/routers/bookmarks.ts:626` | `index` | 更新元数据、内容后触发 |
| 更新书签文本 | `packages/trpc/routers/bookmarks.ts:677` | `index` | 废弃 API，仍兼容 |
| **删除书签** | `packages/trpc/models/bookmarks.ts:934-942` | **`delete`** | ⚠️ 直接调用 `SearchIndexingQueue.enqueue`，**不使用 `triggerSearchReindex`** |
| 更新书签标签 | `packages/trpc/routers/bookmarks.ts:1148` | `index` | 标签增删后触发 |
| 生成书签摘要 | `packages/trpc/routers/bookmarks.ts:1322` | `index` | AI 摘要生成后触发 |
| **标签操作** | | | |
| 标签合并 | `packages/trpc/models/tags.ts:285-295` | `index` | `Tag.merge()` 中对所有受影响书签批量触发 |
| 标签删除 | `packages/trpc/models/tags.ts:324-330` | `index` | `Tag.delete()` 中对所有受影响书签批量触发 |
| 标签重命名 | `packages/trpc/models/tags.ts:362-368` | `index` | `Tag.update()` 中对所有受影响书签批量触发 |
| **异步任务完成** | | | |
| 爬虫完成 | `apps/workers/workers/crawlerWorker.ts:2317` | `index` | 页面抓取完成后触发 |
| AI 打标完成 | `apps/workers/workers/inference/tagging.ts:596` | `index` | 自动标签生成后触发 |
| 资产预处理完成 | `apps/workers/workers/assetPreprocessingWorker.ts:480` | `index` | OCR/PDF 解析后触发 |
| 摘要生成完成 | `apps/workers/workers/inference/summarize.ts:188` | `index` | 摘要任务完成后触发 |
| **管理员操作** | | | |
| 管理员单条重索引 | `packages/trpc/routers/admin.ts:700` | `index` | 手动触发 |
| 管理员全量重索引 | `packages/trpc/routers/admin.ts:248-263` | `index` | 先清空再逐条重建 |

> **⚠️ 重要修正**：删除书签的代码位置是 `packages/trpc/models/bookmarks.ts:934-942`（模型层），**不是**路由层。且该位置直接调用 `SearchIndexingQueue.enqueue` 传入 `type: "delete"`，**不经过 `triggerSearchReindex` 函数**。

**删除书签触发 delete 索引的精确代码：**
```typescript
// packages/trpc/models/bookmarks.ts:934-942
// 注意：这里直接调用 SearchIndexingQueue.enqueue，不经过 triggerSearchReindex
// 且设置了 groupId（index 操作不设置 groupId，仅 delete 操作设置）
await SearchIndexingQueue.enqueue(
  {
    bookmarkId: this.bookmark.id,
    type: "delete",
  },
  {
    groupId: this.ctx.user.id,  // 仅 delete 操作设置 groupId
  },
);
```

**幂等性保证（仅 index 操作）：**
```typescript
// packages/shared-server/src/queues.ts:189-203
// triggerSearchReindex 仅设置 idempotencyKey，不设置 groupId
await SearchIndexingQueue.enqueue(
  { bookmarkId, type: "index" },
  {
    ...opts,
    idempotencyKey: `index:${bookmarkId}`,  // 同一书签的索引任务会去重
    // 注意：不设置 groupId，与 delete 操作不同
  }
);
```

---

## 三、索引构建机制

### 3.1 增量索引（默认模式）

**增量索引是系统的默认工作模式**，每次仅处理单个书签的索引更新。

**流程：**
1. `triggerSearchReindex(bookmarkId)` 入队
2. `SearchIndexingWorker` 消费队列 (`apps/workers/workers/searchWorker.ts:23-176`)
3. 从数据库读取完整书签数据，构建 `BookmarkSearchDocument`
4. 调用 `searchClient.addDocuments([document])` 或 `searchClient.deleteDocuments([id])`

> **⚠️ 接口名称勘误**：`SearchIndexClient` 接口中删除方法名为 `deleteDocuments`（`packages/shared/search.ts:76`），Meilisearch 实现中调用 `this.index.deleteDocuments(ids)`（`packages/plugins/search-meilisearch/src/index.ts:190,257`）。Worker 的 `runDelete` 函数调用的是 `searchClient.deleteDocuments([bookmarkId], { batch })`（`apps/workers/workers/searchWorker.ts:133`）。早期版本中曾误写为 `removeDocuments`，实际代码中不存在该方法名。

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

**文档构建逻辑** (`apps/workers/workers/searchWorker.ts:89-119`)：
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
| **触发入口** | `triggerSearchReindex(bookmarkId)` 或直接入队 | `packages/trpc/routers/admin.ts:248-263` |
| **数据范围** | 单条书签 | 所有书签 |
| **前置操作** | 无 | 先调用 `clearIndex()` 清空整个索引 |
| **队列优先级** | `QueuePriority.Default` (0) | `QueuePriority.Low` (50) |
| **并发控制** | 正常并发 | 低优先级，不影响正常业务 |

**全量索引实现：**
```typescript
// packages/trpc/routers/admin.ts:248-263
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

**设计要点** (`packages/plugins/search-meilisearch/src/index.ts:46-207`)：

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
   // apps/workers/workers/searchWorker.ts:160
   const batch = job.runNumber === 0;  // 首次执行启用批量，重试时禁用
   // 重试时直接发送，不经过批量队列，提高可靠性
   ```

### 3.4 多次索引的时间顺序一致性问题

**场景**：同一书签可能在短时间内多次触发索引，例如：
1. 创建书签 → 触发首次索引（T0）
2. 爬虫完成 → 触发第二次索引（T1）
3. AI 打标完成 → 触发第三次索引（T2）
4. 摘要生成完成 → 触发第四次索引（T3）

**一致性保证机制**：

| 层级 | 保证方式 | 效果 |
|------|---------|------|
| **队列层** | `idempotencyKey: index:{bookmarkId}` | 防止**同一时刻**的重复入队，但不保证顺序 |
| **批量队列层** | `BatchingDocumentQueue` 去重 | 同一批次内只保留最后一次操作 |
| **数据库读取** | Worker 执行时从 DB 读取最新数据 | 最终索引的是数据库中的最新状态 |

**潜在时序问题：**
- 如果任务分布在不同批次，且 Worker 并发执行，可能出现旧数据覆盖新数据的情况
- **`idempotencyKey` 机制**：`triggerSearchReindex` 为 index 操作设置 `idempotencyKey: index:{bookmarkId}`（`packages/shared-server/src/queues.ts:200`），防止同一时刻对同一书签重复入队，但不保证执行顺序
- **`groupId` 机制**：仅 delete 操作设置 `groupId: userId`（`packages/trpc/models/bookmarks.ts:100`），index 操作**不设置 `groupId`**（`packages/shared-server/src/queues.ts:193-202` 中 `triggerSearchReindex` 只传 `idempotencyKey`，不传 `groupId`）。若队列实现按 `groupId` 串行化，仅能约束同一用户的 delete 操作顺序，对 index 操作无约束

#### 可收敛性分析

索引状态是否最终收敛取决于以下前提条件：

| 前提条件 | 代码依据 | 不满足时的后果 |
|---------|---------|--------------|
| ① Worker 执行 `runIndex` 时从 DB 读取的数据是当时最新已提交状态 | `searchWorker.ts:68-80` 使用 `db.query.bookmarks.findFirst` 无锁读 | 若 DB 有未提交事务，Worker 可能读到稍旧数据；但事务提交后再次索引即可修正 |
| ② 对同一书签的多次索引任务最终全部被消费（无永久失败） | 队列配置 `numRetries: 5`（`packages/shared-server/src/queues.ts:145`），失败 5 次后永久丢弃 | 若 5 次重试均失败，该书签索引可能停留在旧状态，**不会自动收敛** |
| ③ 同一书签的并发索引任务不会无限交替执行 | `idempotencyKey` 防止同一时刻重复入队（`packages/shared-server/src/queues.ts:200`）；`BatchingDocumentQueue` 批次内去重（`packages/plugins/search-meilisearch/src/index.ts:100-105`） | 若任务持续产生且交替进入不同批次，理论上存在活锁风险，但实际场景中书签数据变更频率远低于索引消费速度 |

> **审慎结论**：在以下前提同时满足时，索引状态**可收敛至数据库最新已提交状态**：
> - ① Worker 正常消费、重试未耗尽（5 次，`packages/shared-server/src/queues.ts:145`）
> - ② DB 读取时无长事务阻塞导致 Worker 读到旧快照
> - ③ 对同一书签不再产生新的索引触发（否则新的触发可能因 `idempotencyKey` 去重而被丢弃，此时正在队列中的旧任务读取的可能是旧数据）
>
> 系统不提供强顺序一致性保证，且存在以下边界：
> - 重试 5 次全部失败后，索引将永久停留在失败前的状态，需管理员手动触发 `reindexAllBookmarks` 修复
> - index 操作不设置 `groupId`（仅 delete 操作设置），因此 Worker 并发数 `> 1`（`serverConfig.search.numWorkers`，`apps/workers/workers/searchWorker.ts:53`）时，同一书签的两次 index 任务可能交错执行，短暂呈现旧数据覆盖新数据的中间态
> - `BatchingDocumentQueue` 的去重仅在同一批次内生效（`packages/plugins/search-meilisearch/src/index.ts:100-105`），跨批次的不同任务仍会各自执行一次完整的 DB 读取 + 索引写入
> - `idempotencyKey: index:{bookmarkId}` 在任务仍在队列中时阻止新入队，但若旧任务失败后新触发已因去重被丢弃，可能出现索引停留于旧状态直到下一次变更触发

---

## 四、搜索查询流程

### 4.1 查询入口

搜索主入口是 `bookmarksAppRouter.searchBookmarks` (`packages/trpc/routers/bookmarks.ts:804-902`)。

### 4.2 高级查询语法解析与语义降级

用户输入的查询字符串首先经过 `parseSearchQuery` 解析器 (`packages/shared/searchQueryParser.ts:414-451`)，分离出**全文搜索文本**和**结构化过滤条件**。

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

**语义降级策略（Graceful Degradation）**：

解析器返回三种结果类型：

```typescript
function parseSearchQuery(
  query: string,
): TextAndMatcher & { result: "full" | "partial" | "invalid" }
```

| 结果类型 | 触发条件 | 降级行为 | 代码位置 |
|---------|---------|---------|---------|
| **`full`** | 完全解析成功，所有 Token 被消费 | 正常返回 `text` 和 `matcher` | `searchQueryParser.ts:447-451` |
| **`partial`** | 解析成功但部分 Token 未被消费（通常是用户正在输入） | 已解析的 matcher 保留，未消费的 Token 追加到 `text` 中 | `searchQueryParser.ts:433-444` |
| **`invalid`** | 解析器返回零个候选或多个歧义候选 | **整个查询作为纯文本处理**，`matcher` 为 undefined | `searchQueryParser.ts:419-424` |

> **⚠️ `invalid` 触发条件说明**：`invalid` 仅在 `!parsed.successful || parsed.candidates.length != 1` 时触发（`searchQueryParser.ts:419`）。MATCHER 规则的四个分支依次为：① `is:` + Ident（`searchQueryParser.ts:122-189`）、② Qualifier/Hash + Ident/StringLiteral（`searchQueryParser.ts:191-311`）、③ **兜底分支** `alt(tok(TokenType.Ident), tok(TokenType.Qualifier))`（`searchQueryParser.ts:314`）、④ `kmid(LParen, EXP, RParen)`（`searchQueryParser.ts:320`）。由于兜底分支的存在，`Ident` 和 `Qualifier` 类型的 Token 永远能被成功解析为纯文本。**只有当 Token 流的首个 Token 不属于 `Ident` 或 `Qualifier` 类型、且无法匹配其他分支时**，才会触发 `invalid`。具体而言，以下 Token 类型不在兜底分支覆盖范围内：`RParen`、`StringLiteral`（作为独立首 Token）、`LParen`（若 `kmid` 内部解析失败）。

**单限定符解析失败的降级：**
```typescript
// searchQueryParser.ts:253-307 - 日期解析失败时降级为纯文本
case "after:":
  try {
    return {
      text: "",
      matcher: { type: "dateAfter", dateAfter: z.coerce.date().parse(ident), ... },
    };
  } catch {
    return {
      text: (minus?.text ?? "") + qualifier.text + ident,  // 降级为纯文本
      matcher: undefined,
    };
  }
```

**未知限定符的降级：**
```typescript
// searchQueryParser.ts:304-310
default:
  // If the token is not known, emit it as pure text
  return {
    text: (minus?.text ?? "") + qualifier.text + ident,
    matcher: undefined,
  };
```

**降级示例：**
| 输入查询 | 结果类型 | 降级后 text | matcher |
|---------|---------|------------|---------|
| `is:fav is:helloworld` | `full` | `"is:helloworld"` | `{type: "favourited", favourited: true}` |
| `(is:archived) or ` | `partial` | `"or"` | `{type: "archived", archived: true}` |
| `is:fav is: ( random` | `partial` | `"is: ( random"` | `{type: "favourited", favourited: true}` |
| `hello world` | `full` | `"hello world"` | `undefined` |
| `@#$%` | `full` | `"@#$%"` | `undefined`（Ident 规则兜底） |
| `)` | **`invalid`** | `")"` | `undefined`（首 Token 为 RParen，MATCHER 无分支可匹配） |
| `"hello"` | **`invalid`** | `"\"hello\""` | `undefined`（首 Token 为 StringLiteral，MATCHER 无独立分支处理） |

> `hello world` 和 `@#$%` 被 `Ident` 词法规则 `/^[^ )(]+/`（`searchQueryParser.ts:59`）tokenize 后，MATCHER 的兜底分支（`searchQueryParser.ts:314`）将其归为纯文本，因此结果为 `full`。而 `)` 产生 `RParen` Token、`"hello"` 产生 `StringLiteral` Token——这两种 Token 类型不在兜底分支 `alt(tok(Ident), tok(Qualifier))` 的覆盖范围内，也不匹配其他 MATCHER 分支，因此触发 `invalid`。测试文件（`packages/shared/searchQueryParser.test.ts`）中未覆盖 `invalid` 场景。

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
// packages/trpc/lib/search.ts:40-69 - 交集实现
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
// packages/trpc/routers/bookmarks.ts:831-860
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
// packages/plugins/search-meilisearch/src/index.ts:262-281
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

**排序策略** (`packages/trpc/routers/bookmarks.ts:880-890`)：
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
attributesToRetrieve: ["id"],  // packages/plugins/search-meilisearch/src/index.ts:268

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

1. **Meilisearch 查询参数** (`packages/plugins/search-meilisearch/src/index.ts:262-271`)：
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

3. **搜索响应处理** (`packages/plugins/search-meilisearch/src/index.ts:273-280`)：
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

## 六、权限过滤层级与协作者访问机制

### 6.1 三层权限过滤机制

系统采用**三层权限过滤机制**，但各层的作用范围和协作者支持程度不同：

| 层级 | 位置 | 过滤逻辑 | 协作者支持 | 粒度 |
|------|------|---------|-----------|------|
| **L1 搜索引擎** | `routers/bookmarks.ts:838-842` | `userId = currentUser` | ❌ **不支持** | 行级 |
| **L2 数据库加载** | `models/bookmarks.ts:122-131` | 所有权 + 共享列表权限 | ✅ 支持 | 行级 |
| **L3 字段脱敏** | `models/bookmarks.ts:753-766` | 非所有者隐藏敏感字段 | ✅ 支持 | 字段级 |

### 6.2 第一层：搜索引擎级过滤（查询时）

**位置：** `packages/trpc/routers/bookmarks.ts:838-842`

在搜索请求中强制加入 `userId` 过滤条件，在搜索引擎层面就排除其他用户的数据。

```typescript
filter = [
  { type: "eq", field: "userId", value: ctx.user.id },  // 🔒 强制过滤
];
```

**⚠️ 关键限制**：这一层过滤**只返回当前用户作为所有者**的书签，**完全排除协作者可以访问的共享书签**。

**Meilisearch 配置：**
```typescript
// packages/plugins/search-meilisearch/src/index.ts:364
const desiredFilterableAttributes = ["id", "userId"].sort();
// userId 被配置为可过滤字段，确保可以在查询时过滤
```

### 6.3 第二层：数据加载时权限校验

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

**协作者支持**：这一层支持协作者访问，但由于 L1 已经过滤掉了所有非所有者的书签，**在 `searchBookmarks` 流程中这一层实际上不会检测到协作者书签**。

### 6.4 第三层：数据返回时字段过滤

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

### 6.5 协作者访问机制与全局搜索的关系

**关键发现**：协作者无法通过 `searchBookmarks` API 搜索到共享列表中的书签。

**原因分析**：

```
用户 A 分享列表给 用户 B
         │
         ▼
用户 B 调用 searchBookmarks("keyword")
         │
         ▼
L1: userId = B 的 ID 过滤 → ❌ 排除所有用户 A 的书签
         │
         ▼
结果为空（即使有匹配的共享书签）
```

#### 6.5.1 全局搜索路径 `searchBookmarks` 与列表浏览路径 `getBookmarks` 的区别

这两条数据路径在权限过滤的入口处就已分叉，不可混淆：

| 维度 | `searchBookmarks` 路径 | `getBookmarks` 路径 |
|------|----------------------|---------------------|
| **入口** | `routers/bookmarks.ts:804` | `routers/bookmarks.ts:951` |
| **数据获取** | 先经 Meilisearch 全文搜索，再 `Bookmark.loadMulti` | 直接 `Bookmark.loadMulti`，不经搜索引擎 |
| **过滤逻辑** | `parseSearchQuery` → `getBookmarkIdsFromMatcher`（数据库结构化过滤）→ Meilisearch 全文搜索 + `userId` 过滤 | `Bookmark.loadMulti` 内部的 DB 查询过滤（按 `listId`/`tagId`/`rssFeedId`/`archived`/`favourited`/`type`/`ids`，`models/bookmarks.ts:400-460`） |
| **L1 权限过滤** | Meilisearch 强制 `userId = currentUser`（`routers/bookmarks.ts:838-842`） | 无 L1 搜索引擎过滤；`loadMulti` 的 DB 查询本身不限制 `userId` |
| **L2 权限校验** | `Bookmark.loadMulti` → `isAllowedToAccessBookmark`（`models/bookmarks.ts:122-131`），但 L1 已排除协作者可见书签 | 同样经过 `isAllowedToAccessBookmark`，但输入 ID 未被 L1 截断，**协作者可见书签可以通过** |
| **全文搜索** | ✅ 支持（Meilisearch） | ❌ 不支持 |
| **协作者可见** | ❌ 不可以 | ✅ 可以（`isAllowedToAccessBookmark` 会校验共享列表权限） |

> **⚠️ 两条路径的过滤机制完全不同**：`searchBookmarks` 使用 `parseSearchQuery` + `getBookmarkIdsFromMatcher`（`packages/trpc/lib/search.ts`）将查询解析为 Matcher AST 再转为 DB 查询，而 `getBookmarks` 使用 `Bookmark.loadMulti`（`packages/trpc/models/bookmarks.ts:400-460`）直接根据 `zGetBookmarksRequestSchema` 的字段构建 DB 查询。**两者不共享过滤逻辑**，不能将 `getBookmarks` 的协作者可见性推广到 `searchBookmarks`。

#### 6.5.2 协作者的实际访问路径

协作者只能通过**列表上下文**查看共享书签：
1. 进入共享列表详情页 → 前端调用 `getBookmarks` 接口（`routers/bookmarks.ts:951`）
2. 该接口走 `Bookmark.loadMulti(ctx, input)`（`models/bookmarks.ts:400-460`），根据 `listId` 等 `zGetBookmarksRequestSchema` 字段构建 DB 查询，**不经过全局搜索引擎**
3. `loadMulti` 内部对每条结果调用 `isAllowedToAccessBookmark`（`models/bookmarks.ts:122-131`），校验共享列表权限
4. **协作者对共享书签没有全文搜索能力**——因为全文搜索唯一走 `searchBookmarks`，而 `searchBookmarks` 的 L1 已排除所有非所有者书签
5. 列表浏览中的过滤（按标签、按类型、按收藏等）由 `loadMulti` 内部 DB 查询实现（`models/bookmarks.ts:428-460`），**不使用 `getBookmarkIdsFromMatcher`**（该函数仅用于 `searchBookmarks` 路径）

**设计权衡**：
- ✅ 简化全局搜索的权限模型，性能最优
- ✅ 避免索引中存储复杂的共享权限信息
- ❌ 协作者无法使用全局搜索功能
- ❌ 共享书签的全文搜索能力受限

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

**同步机制**：标签更新时触发重新索引，确保数据一致。标签变更会批量触发所有受影响书签的重新索引：
- 标签合并：`packages/trpc/models/tags.ts:285-295`
- 标签删除：`packages/trpc/models/tags.ts:324-330`
- 标签重命名：`packages/trpc/models/tags.ts:362-368`

### 7.4 为什么批量处理仅在首次执行时启用？

**设计：** `batch = job.runNumber === 0`，重试时禁用批量

**理由：**
1. **首次执行**：批量可显著减少 Meilisearch 的任务数，提高吞吐量
2. **重试场景**：批量中的单个文档失败会导致整个批次重试，浪费资源
3. **可靠性优先**：重试时应该直接发送单个任务，避免被其他文档牵连

### 7.5 为什么全局搜索不支持协作者访问？

**设计：** L1 搜索引擎层仅过滤 `userId = currentUser`，不考虑共享权限

**理由：**
1. **性能**：避免在搜索引擎中存储复杂的权限矩阵
2. **简化模型**：索引文档只需存储 `userId`，无需维护协作者列表
3. **更新成本**：协作者变更时无需重新索引所有相关书签
4. **使用场景**：协作者通常在列表上下文内工作，全局搜索需求较少

**权衡**：
- ✅ 索引结构简单，更新成本低
- ✅ 搜索查询性能最优
- ❌ 协作者无法使用全局搜索

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
    ├─ result: "full" / "partial" / "invalid"
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
    filter: [id IN [id2, id3], userId = currentUser]  ← ⚠️ 协作者被排除
    sort: createdAt:desc
    │
    ▼
返回结果: [{id: "id3", score: 0.92}, {id: "id2", score: 0.85}]
    │
    ▼
Bookmark.loadMulti([id3, id2])
    ├─ 从 DB 读取完整数据
    └─ isAllowedToAccessBookmark() 权限校验  ← ⚠️ 但 L1 已过滤协作者
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
| **deleteDocuments 接口** | `packages/shared/search.ts` | **76** |
| Meilisearch 插件 | `packages/plugins/search-meilisearch/src/index.ts` | 全文件 |
| **deleteDocuments 实现** | `packages/plugins/search-meilisearch/src/index.ts` | **190, 247-259** |
| 批量队列实现 | `packages/plugins/search-meilisearch/src/index.ts` | 46-207 |
| 搜索 Worker | `apps/workers/workers/searchWorker.ts` | 全文件 |
| **runDelete 调用 deleteDocuments** | `apps/workers/workers/searchWorker.ts` | **133** |
| 触发索引函数 | `packages/shared-server/src/queues.ts` | 189-203 |
| **队列 numRetries 配置** | `packages/shared-server/src/queues.ts` | **145** |
| **删除书签触发 delete** | `packages/trpc/models/bookmarks.ts` | **934-942** |
| 查询解析器 | `packages/shared/searchQueryParser.ts` | 全文件 |
| **Ident 词法规则** | `packages/shared/searchQueryParser.ts` | **59** |
| **MATCHER 兜底分支** | `packages/shared/searchQueryParser.ts` | **314** |
| **语义降级逻辑** | `packages/shared/searchQueryParser.ts` | **414-451** |
| 数据库过滤逻辑 | `packages/trpc/lib/search.ts` | 全文件 |
| 搜索 API 路由 | `packages/trpc/routers/bookmarks.ts` | 804-902 |
| 创建书签触发索引 | `packages/trpc/routers/bookmarks.ts` | 431-451 |
| **L1 userId 过滤** | `packages/trpc/routers/bookmarks.ts` | **838-842** |
| **getBookmarks 入口** | `packages/trpc/routers/bookmarks.ts` | **951** |
| **Bookmark.loadMulti** | `packages/trpc/models/bookmarks.ts` | **400-460** |
| 权限校验 | `packages/trpc/models/bookmarks.ts` | 122-131 |
| 字段脱敏 | `packages/trpc/models/bookmarks.ts` | 753-766 |
| **标签合并触发索引** | `packages/trpc/models/tags.ts` | **285-295** |
| **标签删除触发索引** | `packages/trpc/models/tags.ts` | **324-330** |
| **标签重命名触发索引** | `packages/trpc/models/tags.ts` | **362-368** |
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
7. **协作者全局搜索**：可考虑在 L1 过滤后补充协作者可访问的书签 ID，或使用更复杂的权限模型
8. **索引顺序保证**：可考虑使用有序队列或版本号确保索引操作的顺序一致性
