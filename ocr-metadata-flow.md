# OCR 与文档 Metadata 抽取链路分析

## 1. 整体架构概览

Karakeep 的 OCR 与文档 metadata 抽取链路是一个多阶段、异步驱动的处理管道，主要由以下核心组件构成：

```
用户上传/导入资产
      ↓
[资产预处理队列] → AssetPreprocessingWorker
      ↓
  ┌─┴───────────────┐
  │                 │
图片处理         PDF 处理
(OCR)         (文本提取+截图)
  │                 │
  └─┬───────────────┘
      ↓
结果回写数据库 (bookmarkAssets)
      ↓
  ┌─┴───────────────┐
  │                 │
[嵌入队列]      [推理队列]
Embeddings      OpenAI
Worker          Worker
(向量索引)    (标签+摘要)
  │                 │
  └─┬───────────────┘
      ↓
[搜索索引队列] → SearchIndexingWorker
      ↓
MeiliSearch 全文索引 + 向量存储
```

## 2. 图片处理流程（OCR）

### 2.1 OCR 处理入口
入口文件：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L384-L464)

主处理函数 `run()` 根据资产类型分发处理：
- `image` 类型 → 调用 `extractAndSaveImageText()`
- `pdf` 类型 → 调用 `extractAndSavePDFText()` + `extractAndSavePDFScreenshot()`

### 2.2 OCR 双引擎架构
Karakeep 支持两种 OCR 引擎，可通过配置切换：

#### 2.2.1 Tesseract OCR（传统 OCR）
代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L120-L136)

```typescript
async function readImageText(buffer: Buffer) {
  const worker = await createWorker(serverConfig.ocr.langs, undefined, {
    cachePath: serverConfig.ocr.cacheDir ?? os.tmpdir(),
  });
  try {
    const ret = await worker.recognize(buffer);
    if (ret.data.confidence <= serverConfig.ocr.confidenceThreshold) {
      return null;
    }
    return ret.data.text;
  } finally {
    await worker.terminate();
  }
}
```

**关键特性：**
- 支持多语言配置（`serverConfig.ocr.langs`）
- 置信度阈值过滤（`confidenceThreshold`）
- 本地缓存模型文件

#### 2.2.2 LLM OCR（基于大模型的视觉识别）
代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L138-L168)

```typescript
async function readImageTextWithLLM(
  buffer: Buffer,
  contentType: string,
): Promise<string | null> {
  const inferenceClient = InferenceClientFactory.build();
  if (!inferenceClient) {
    logger.warn(
      "[assetPreprocessing] LLM OCR is enabled but no inference client is configured. Falling back to Tesseract.",
    );
    return readImageText(buffer);
  }

  const base64 = buffer.toString("base64");
  const prompt = buildOCRPrompt();

  const response = await inferenceClient.inferFromImage(
    prompt, contentType, base64, { schema: null }
  );

  const extractedText = response.response.trim();
  if (!extractedText) {
    return null;
  }

  return extractedText;
}
```

> **重要：** `readImageTextWithLLM` 函数内部有**第一层回退机制**——当推理客户端未配置时，会直接调用 `readImageText()` 回退到 Tesseract。

#### 2.2.3 OCR 引擎选择与回退逻辑
代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L278-L335)

**外层选择逻辑（`extractAndSaveImageText` 函数）：**

```typescript
let imageText = null;

if (serverConfig.ocr.useLLM) {
  logger.info(`Attempting to extract text from image using LLM OCR.`);
  try {
    imageText = await readImageTextWithLLM(asset, contentType);
  } catch (e) {
    logger.error(`Failed to read image text with LLM: ${e}`);
  }
} else {
  logger.info(`Attempting to extract text from image using Tesseract.`);
  try {
    imageText = await readImageText(asset);
  } catch (e) {
    logger.error(`Failed to read image text: ${e}`);
  }
}

if (!imageText) {
  return false;
}
```

**回退机制的真实行为：**

| 场景 | 是否回退到 Tesseract | 行为 |
|------|---------------------|------|
| LLM 启用 + 推理客户端未配置 | ✅ 是 | 在 `readImageTextWithLLM` 内部直接调用 `readImageText()` 回退 |
| LLM 启用 + 推理调用**抛出异常** | ❌ 否 | catch 后仅记录错误，`imageText` 保持为 null，不回退 |
| LLM 启用 + 返回空文本 | ❌ 否 | 返回 null，不回退 |
| LLM 启用 + 置信度不适用 | - | LLM OCR 没有置信度过滤机制 |
| Tesseract 模式 + 置信度过低 | ❌ 否 | 返回 null，任务视为未提取到文本 |
| Tesseract 模式 + 调用抛出异常 | ❌ 否 | catch 后仅记录错误，不回退 |

> **关键澄清**：之前的"LLM OCR 失败后自动回退 Tesseract"的说法不准确。实际上**仅当推理客户端未配置时**才会自动回退。其他所有失败场景（调用异常、返回空文本等）都**不会**回退，而是直接返回 false，任务被视为"未提取到文本"但不报错。

### 2.3 OCR 相关配置
配置文件：[config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/shared/config.ts)

- `ocr.langs`: OCR 识别语言列表
- `ocr.cacheDir`: 模型缓存目录
- `ocr.confidenceThreshold`: 置信度阈值（低于此值的结果会被丢弃）
- `ocr.useLLM`: 是否启用 LLM OCR

## 3. PDF 处理流程

### 3.1 PDF 文本提取
代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L170-L185)

使用 `pdf2json` 库解析 PDF：

```typescript
async function readPDFText(buffer: Buffer): Promise<{
  text: string;
  metadata: Record<string, object>;
}> {
  return new Promise((resolve, reject) => {
    const pdfParser = new PDFParser(null, true);
    pdfParser.on("pdfParser_dataError", reject);
    pdfParser.on("pdfParser_dataReady", (pdfData) => {
      resolve({
        text: pdfParser.getRawTextContent(),
        metadata: pdfData.Meta,
      });
    });
    pdfParser.parseBuffer(buffer);
  });
}
```

**返回数据：**
- `text`: PDF 原始文本内容
- `metadata`: PDF 元数据（作者、标题、创建日期等）

### 3.2 PDF 截图生成
代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L187-L276)

使用 `pdf2pic` 库生成 PDF 第一页截图：

```typescript
const screenshot = await fromBuffer(asset, {
  density: 100,
  quality: 100,
  format: "png",
  preserveAspectRatio: true,
})(1, { responseType: "buffer" });
```

**配额检查：** 生成截图前会检查用户存储配额：
```typescript
const quotaApproved = await QuotaService.checkStorageQuota(
  db, bookmark.userId, screenshot.buffer.byteLength
);
```

**存储流程：**
1. 调用 `saveAsset()` 保存截图文件到资产存储（本地文件系统或 S3）
2. 插入记录到 `assets` 表，类型为 `AssetTypes.ASSET_SCREENSHOT`

### 3.3 PDF 文本提取与保存
代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L337-L372)

```typescript
await db.update(bookmarkAssets).set({
  content: pdfParse.text,
  metadata: pdfParse.metadata ? JSON.stringify(pdfParse.metadata) : null,
}).where(eq(bookmarkAssets.id, bookmark.id));
```

## 4. 异步任务队列系统

### 4.1 队列架构
队列系统基于插件化设计，支持多种后端实现：

**核心接口定义：** [queueing.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/shared/queueing.ts)

```typescript
interface Queue<T> {
  enqueue(payload: T, options?: EnqueueOptions): Promise<string | undefined>;
  stats(): Promise<{ pending: number; pending_retry: number; running: number; failed: number }>;
}

interface QueueClient {
  createQueue<T>(name: string, options: QueueOptions): Queue<T>;
  createRunner<T, R>(queue: Queue<T>, funcs: RunnerFuncs<T, R>, opts: RunnerOptions<T>): Runner<T>;
}
```

**队列实现：**
- Restate 队列：[queue-restate](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/plugins/queue-restate/src/index.ts)
- Liteque 队列：[queue-liteque](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/plugins/queue-liteque/src/index.ts)

### 4.2 相关队列定义
队列定义：[queues.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/shared-server/src/queues.ts)

| 队列名称 | 重试次数 | 用途 |
|---------|---------|------|
| `AssetPreprocessingQueue` | 2 | 资产预处理（OCR、PDF 处理） |
| `EmbeddingsQueue` | 3 | 向量嵌入生成与索引 |
| `OpenAIQueue` | 3 | AI 推理（打标签、摘要） |
| `SearchIndexingQueue` | 5 | 全文索引构建 |

### 4.3 任务调度与执行模型（Restate）

#### Dispatcher（调度器）
代码位置：[dispatcher.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/plugins/queue-restate/src/dispatcher.ts)

**核心职责：**
- 管理并发控制（信号量）
- 处理重试逻辑
- 调用 Runner 执行实际任务
- 幂等性检查

**重试调度策略：**
```typescript
retryPolicy: {
  maxAttempts: NUM_RETRIES,  // 从队列配置获取
  initialInterval: { seconds: 5 },
  maxInterval: { minutes: 1 },
}
```

#### Runner（执行器）
代码位置：[runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/plugins/queue-restate/src/runner.ts)

**核心职责：**
- 执行实际的任务处理函数
- 超时控制（`AbortSignal`）
- 调用 `onComplete` / `onError` 回调
- 特殊错误处理：`QueueRetryAfterError`（速率限制）

### 4.4 Worker 初始化模式
所有 Worker 都遵循相同的构建模式：

```typescript
export class AssetPreprocessingWorker {
  static async build() {
    const worker = (await getQueueClient())!.createRunner<AssetPreprocessingRequest>(
      AssetPreprocessingQueue,
      {
        run: withWorkerTracing("assetPreprocessingWorker.run",
             withWorkerEventLog("assetPreprocessingWorker.run", run)),
        onComplete: async (job) => { /* 成功回调 */ },
        onError: async (job) => { /* 错误回调 */ },
      },
      {
        concurrency: serverConfig.assetPreprocessing.numWorkers,
        pollIntervalMs: 1000,
        timeoutSecs: serverConfig.assetPreprocessing.jobTimeoutSec,
      }
    );
    return worker;
  }
}
```

## 5. 资产存储层

### 5.1 资产存储接口
代码位置：[assetdb.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/shared/assetdb.ts)

支持两种存储后端：
- **本地文件系统**：`LocalFileSystemAssetStore`
- **S3 兼容存储**：`S3AssetStore`

**存储结构：**
```
{userId}/{assetId}/
  ├─ asset.bin       # 原始文件内容
  └─ metadata.json   # 元数据（contentType, fileName）
```

### 5.2 核心读写操作

```typescript
// 读取资产
const { asset, metadata } = await readAsset({
  userId: bookmark.userId,
  assetId: bookmark.asset.assetId,
});

// 保存资产
await saveAsset({
  userId: bookmark.userId,
  assetId,
  asset: screenshot.buffer,
  metadata: { contentType, fileName },
  quotaApproved,
});
```

## 6. 结果回写机制

### 6.1 数据库表结构

#### bookmarkAssets 表（主资产表）
代码位置：[schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/db/schema.ts#L407-L419)

```typescript
export const bookmarkAssets = sqliteTable("bookmarkAssets", {
  id: text("id").notNull().primaryKey(),
  assetType: text("assetType", { enum: ["image", "pdf"] }).notNull(),
  assetId: text("assetId").notNull(),        // 关联到资产存储
  content: text("content"),                   // OCR 提取的文本 / PDF 文本
  metadata: text("metadata"),                 // JSON 格式的元数据
  fileName: text("fileName"),
  sourceUrl: text("sourceUrl"),
});
```

#### assets 表（附属资产表）
代码位置：[schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/db/schema.ts#L298-L336)

用于存储 PDF 截图、链接截图等附属资产。

### 6.2 OCR 结果回写
代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L327-L333)

```typescript
await db.update(bookmarkAssets).set({
  content: imageText,    // OCR 识别的文本
  metadata: null,
}).where(eq(bookmarkAssets.id, bookmark.id));
```

### 6.3 PDF 结果回写
代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L364-L370)

```typescript
await db.update(bookmarkAssets).set({
  content: pdfParse.text,                          // PDF 文本内容
  metadata: JSON.stringify(pdfParse.metadata),     // PDF 元数据（JSON 字符串）
}).where(eq(bookmarkAssets.id, bookmark.id));
```

### 6.4 处理状态标记
在 `bookmarks` 表中有三个状态字段跟踪处理进度：

```typescript
taggingStatus: text("taggingStatus", { enum: ["pending", "failure", "success"] })
summarizationStatus: text("summarizationStatus", { enum: ["pending", "failure", "success"] })
embeddingStatus: text("embeddingStatus", { enum: ["pending", "failure", "success"] })
```

**失败状态清理：** [assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L68-L104)

当资产预处理任务永久失败时（重试次数耗尽），会清理相关的 pending 状态：

```typescript
if (bookmarkId && job.numRetriesLeft == 0) {
  await db.transaction(async (tx) => {
    await tx.update(bookmarks).set({ taggingStatus: null })
      .where(and(eq(bookmarks.id, bookmarkId), eq(bookmarks.taggingStatus, "pending")));
    await tx.update(bookmarks).set({ summarizationStatus: null })
      .where(and(eq(bookmarks.id, bookmarkId), eq(bookmarks.summarizationStatus, "pending")));
    await tx.update(bookmarks).set({ embeddingStatus: null })
      .where(and(eq(bookmarks.id, bookmarkId), eq(bookmarks.embeddingStatus, "pending")));
  });
}
```

## 7. 下游任务触发

资产预处理完成后，会触发一系列下游任务：

代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L470-L504)

```typescript
if (!isFixMode || anythingChanged) {
  if (serverConfig.embedding.enableAutoIndexing) {
    // 路径1：自动索引模式 → 生成嵌入向量
    await EmbeddingsQueue.enqueue({
      bookmarkId,
      type: "embed",
      runTaggingOnComplete: true,
    }, enqueueOpts);
  } else {
    // 路径2：非自动索引模式 → 直接打标签
    await OpenAIQueue.enqueue({
      bookmarkId,
      type: "tag",
    }, enqueueOpts);
  }
  // 生成摘要
  await OpenAIQueue.enqueue({
    bookmarkId,
    type: "summarize",
  }, enqueueOpts);
  // 更新搜索索引
  await triggerSearchReindex(bookmarkId, enqueueOpts);
}
```

### 7.1 优先级传递
所有子任务会继承父任务的优先级和用户组：

```typescript
const enqueueOpts: EnqueueOptions = {
  priority: req.priority,   // 继承父任务优先级
  groupId: bookmark.userId, // 按用户分组，确保公平调度
};
```

## 8. 全文索引流程

### 8.1 触发搜索索引
代码位置：[queues.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/shared-server/src/queues.ts#L231-L245)

```typescript
export async function triggerSearchReindex(bookmarkId: string, opts?: EnqueueOptions) {
  await SearchIndexingQueue.enqueue(
    { bookmarkId, type: "index" },
    { ...opts, idempotencyKey: `index:${bookmarkId}` }
  );
}
```

**幂等性设计：** 使用 `index:${bookmarkId}` 作为幂等键，避免重复索引。

### 8.2 索引文档构建
代码位置：[searchWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/searchWorker.ts#L63-L126)

构建 `BookmarkSearchDocument`：

```typescript
const document: BookmarkSearchDocument = {
  id: bookmark.id,
  userId: bookmark.userId,
  // 链接类型字段
  ...(bookmark.link ? {
    url: bookmark.link.url,
    linkTitle: bookmark.link.title,
    description: bookmark.link.description,
    content: await Bookmark.getBookmarkPlainTextContent(bookmark.link, bookmark.userId),
    publisher: bookmark.link.publisher,
    author: bookmark.link.author,
  } : {}),
  // 资产类型字段（包含 OCR/PDF 提取的内容）
  ...(bookmark.asset ? {
    content: bookmark.asset.content,    // OCR 文本 / PDF 文本
    metadata: bookmark.asset.metadata,  // PDF 元数据
  } : {}),
  // 其他字段
  note: bookmark.note,
  summary: bookmark.summary,
  title: bookmark.title,
  tags: bookmark.tagsOnBookmarks.map((t) => t.tag.name),
};
```

### 8.3 批量处理策略
代码位置：[searchWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/searchWorker.ts#L159-L160)

```typescript
// 首次运行启用批量，重试时禁用批量以提高可靠性
const batch = job.runNumber === 0;
```

## 9. 向量索引与 Metadata 抽取

### 9.1 Embeddings Worker 流程
代码位置：[embeddingsWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/embeddingsWorker.ts)

**三段式设计：**
1. `embed` 类型：生成嵌入向量，然后分发 tagging 和 index 子任务
2. `index` 类型：将预计算的向量持久化到向量存储
3. `delete` 类型：从向量存储删除向量

**关键设计：** [embeddingsWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/embeddingsWorker.ts#L394-L397)

> `embed` 任务从不调用 `addVectors`，而是将向量传递给独立的 `index` 任务。这样即使索引失败重试，也不会重复触发 tagging 任务。

### 9.2 嵌入文本构建
代码位置：[embeddingsWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/embeddingsWorker.ts#L256-L344)

对于资产类型书签，构建文本时会包含：
- 标题、类型、标签、摘要、笔记
- 源 URL、域名、文件名、资产类型
- Metadata（JSON 解析后格式化）
- 内容摘要（截断到配置的长度限制）

```typescript
} else if (bookmark.asset) {
  appendProfileField(parts, "Source URL", urlForEmbedding(bookmark.asset.sourceUrl));
  appendProfileField(parts, "File name", bookmark.asset.fileName);
  appendProfileField(parts, "Asset type", bookmark.asset.assetType);
  appendProfileField(parts, "Metadata", metadataForEmbedding(bookmark.asset.metadata));
  rawContent = bookmark.asset.content;  // OCR/PDF 提取的文本
}
```

### 9.3 AI 打标签（Tagging）
代码位置：[tagging.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/inference/tagging.ts)

**不同资产类型的处理：**
- **图片**：调用 `inferTagsFromImage()`，直接将图片发送给多模态模型
- **PDF**：调用 `inferTagsFromPDF()`，使用提取的文本内容
- **链接/文本**：调用 `inferTagsFromText()`

**标签关联逻辑：** [tagging.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/inference/tagging.ts#L413-L513)

1. 标准化标签名称（小写、去除空格/连字符/下划线）
2. 匹配已存在的标签，避免重复创建
3. 创建不存在的新标签
4. 删除旧的 AI 标签，插入新标签
5. 触发规则引擎事件

### 9.4 AI 摘要（Summarization）
代码位置：[summarize.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/inference/summarize.ts)

#### 完整的分支逻辑

代码位置：[summarize.ts#L97-L126](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/inference/summarize.ts#L97-L126)

```typescript
if (bookmarkData.type === BookmarkTypes.LINK && bookmarkData.link) {
  // LINK 类型：构建待摘要文本，执行摘要
  textToSummarize = `
Title: ${link.title ?? ""}
Description: ${link.description ?? ""}
Content: ${content}
Publisher: ${link.publisher ?? ""}
Author: ${link.author ?? ""}
URL: ${link.url ?? ""}
`;
} else {
  // 非 LINK 类型（IMAGE / PDF 等）：仅记录警告，直接 return
  logger.warn(
    `[inference][${jobId}] Bookmark ${bookmarkId} (type: ${bookmarkData.type}) ` +
    `is not a LINK or TEXT type with content, or content is missing. Skipping summary.`,
  );
  return;  // ← 不抛异常，不写回摘要，不触发搜索索引
}
```

#### 非 LINK 类型的完整行为链

尽管资产预处理阶段会为**所有类型**的书签入队摘要任务（包括 IMAGE 和 PDF），但推理阶段的摘要分支对非 LINK 类型的处理如下：

| 步骤 | 行为 |
|------|------|
| `runSummarization()` 进入 else 分支 | 记录 `logger.warn` |
| 函数 `return` | 不抛异常，正常退出 |
| `onComplete` 回调触发 | 标记 `summarizationStatus: "success"` |
| 数据库 `bookmarks.summary` | 保持为 `null`（未写入任何摘要） |
| 搜索索引更新 | **不会触发**（`return` 前没有调用 `triggerSearchReindex`） |

> **⚠️ 关键澄清**：IMAGE/PDF 类型书签的 `summarizationStatus` 最终被标记为 `"success"`，但实际上**没有任何摘要被写入**。这是一个"成功跳过"——任务本身没有出错，但也没有产出。

#### 代码中的扩展预留

[summarize.ts#L36-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/inference/summarize.ts#L36-L38) 中的注释：

```typescript
with: {
  link: { columns: { ... } },
  // If assets (like PDFs with extracted text) should be summarized, extend here
},
```

这表明开发者预留了对资产类型摘要的扩展点，但当前尚未实现。

#### LINK 类型摘要的结果回写

```typescript
await db.update(bookmarks).set({
  summary: summaryResult.response,
  modifiedAt: new Date(),
}).where(eq(bookmarks.id, bookmarkId));

await triggerSearchReindex(bookmarkId, {
  priority: job.priority,
  groupId: bookmarkData.userId,
});
```

## 10. 重跑策略

### 10.1 自动重试（Restate 级）

#### 指数退避 + 全抖动算法
代码位置：[dispatcher.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L169-L173)

```typescript
// 指数退避：基础延迟 5s，每次翻倍，最大 60s
const baseMs = Math.min(5000 * 2 ** runNumber, 60000);
// 全抖动：随机化延迟，避免惊群效应
const delayMs = Math.floor(ctx.rand.random() * baseMs);
await ctx.sleep(delayMs, "rpc error retry");
```

#### 特殊错误类型

**QueueRetryAfterError（速率限制）：** [queueing.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/shared/queueing.ts#L10-L18)

```typescript
export class QueueRetryAfterError extends Error {
  constructor(message: string, public readonly delayMs: number) {
    super(message);
  }
}
```

处理逻辑：[dispatcher.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L179-L186)

- 释放信号量
- 休眠指定的延迟时间
- **不增加 runNumber**（不消耗重试次数）

### 10.2 不同队列的重试次数

| 队列 | 重试次数 | 最大延迟 |
|------|---------|---------|
| AssetPreprocessingQueue | 2 | 20s |
| EmbeddingsQueue | 3 | 40s |
| OpenAIQueue | 3 | 40s |
| SearchIndexingQueue | 5 | 60s |

### 10.3 手动重跑（Fix Mode）

**触发方式：** 入队时设置 `fixMode: true`

代码位置：[assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L285-L293)

```typescript
// 图片文本提取的 fixMode 检查
{
  const alreadyHasText = !!bookmark.asset.content;
  if (alreadyHasText && isFixMode) {
    logger.info(`Skipping image text extraction as it's already been extracted.`);
    return false;
  }
}
```

**行为差异：**
- `fixMode = false`（默认）：即使内容已存在也会重新处理
- `fixMode = true`：如果内容已存在则跳过处理

**下游任务触发逻辑：** [assetPreprocessingWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/assetPreprocessingWorker.ts#L475-L504)

```typescript
// 仅当非 fixMode 或内容有变化时，才触发下游任务
if (!isFixMode || anythingChanged) {
  // 触发 Embeddings、OpenAI、SearchIndexing 任务
}
```

### 10.4 失败后的状态标记

**Embeddings 失败处理：** [embeddingsWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/embeddingsWorker.ts#L47-L64)

```typescript
onError: async (job) => {
  if (job.numRetriesLeft == 0) {
    await attemptMarkEmbeddingStatus(job.data, "failure");
    // 即使嵌入失败，仍然触发 tagging（不带向量相似度上下文）
    if (job.data?.type === "embed" && job.data.runTaggingOnComplete !== false) {
      await enqueueTaggingFallback(job);
    }
  }
}
```

**推理任务失败处理：** [inferenceWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/46-karakeep/apps/workers/workers/inference/inferenceWorker.ts#L60-L70)

```typescript
onError: async (job) => {
  if (job.numRetriesLeft == 0) {
    await attemptMarkStatus(job?.data, "failure");
  }
}
```

## 11. 完整数据流时序

```
用户上传图片/PDF
      │
      ▼
  创建 bookmark + bookmarkAssets 记录
      │
      ▼
  入队 AssetPreprocessingQueue
      │
      ▼
  AssetPreprocessingWorker 处理
      │
      ├─ 读取资产文件（assetdb.readAsset）
      │
      ├─ 图片类型：
      │    │
      │    ├─ 判断 ocr.useLLM 配置
      │    │
      │    ├─ useLLM = false：
      │    │    └─ 直接使用 Tesseract OCR
      │    │         ├─ 成功 → 写入 bookmarkAssets.content
      │    │         └─ 失败（异常/置信度低/空）→ 返回 null
      │    │
      │    └─ useLLM = true：
      │         └─ 进入 readImageTextWithLLM()
      │              │
      │              ├─ 【回退仅发生于此】
      │              │   推理客户端未配置 → 调用 readImageText() 回退 Tesseract ✅
      │              │
      │              ├─ 推理调用抛出异常 → catch 记录错误，不回退 ❌
      │              ├─ 推理返回空文本 → 返回 null，不回退 ❌
      │              └─ 提取成功 → 写入 bookmarkAssets.content
      │
      └─ PDF 类型：
           ├─ 提取文本 + metadata
           │    ├─ 成功 → 写入 bookmarkAssets
           │    └─ 文本为空 → 抛出异常，任务失败 ⚠️
           │
           └─ 生成第一页截图 → 保存到资产存储 + assets 表
                ├─ 成功 → anythingChanged = true
                └─ 失败（配额超限）→ 返回 true（不算失败）
      │
      ▼
  触发下游任务（条件：!isFixMode || anythingChanged）
      │
      ├─ 非 fixMode：总是触发（即使 OCR 失败）
      └─ fixMode：仅在内容变化时触发
      │
      ▼
  触发下游任务：
  ├─ EmbeddingsQueue（如果启用自动索引）
  ├─ OpenAIQueue（tag）
  ├─ OpenAIQueue（summarize）
  └─ SearchIndexingQueue
      │
      ▼
EmbeddingsWorker 处理：
  ├─ 构建嵌入文本（包含 OCR/PDF 内容）
  ├─ 生成向量
  ├─ 入队 EmbeddingsQueue（index 类型）
  └─ 入队 OpenAIQueue（tag，携带向量）
      │
      ▼
OpenAIWorker 处理：
  ├─ Tagging：
  │   ├─ 图片→多模态模型推理标签
  │   ├─ PDF→文本模型推理标签
  │   ├─ 保存标签关联（tagsOnBookmarks）
  │   └─ 触发 SearchIndexingQueue
  │
  └─ Summarize：
      ├─ 生成摘要文本
      ├─ 写入 bookmarks.summary
      └─ 触发 SearchIndexingQueue
      │
      ▼
SearchIndexingWorker 处理：
  ├─ 读取完整 bookmark 数据
  ├─ 构建搜索文档（含 OCR/PDF 内容、标签、摘要）
  └─ 写入 MeiliSearch 索引
```

## 12. 关键设计决策总结

### 12.1 OCR 与 PDF 处理

1. **双引擎 OCR**：支持 Tesseract（传统 OCR）和 LLM OCR（大模型视觉识别），通过配置切换
2. **有限回退机制**：仅当推理客户端未配置时，LLM OCR 才会自动回退到 Tesseract；其他失败场景不回退
3. **不对称的失败处理**：
   - 图片 OCR 失败：静默返回 false，不影响任务整体成功
   - PDF 文本为空：抛出异常，导致整个任务失败并重试
4. **配额感知的截图生成**：PDF 截图生成失败（如配额超限）不算任务失败，返回 true 继续执行

### 12.2 任务调度与重试

5. **任务解耦**：每个处理阶段都是独立的队列任务，可独立扩展和重试
6. **幂等性设计**：使用 idempotencyKey 避免重复处理
7. **分级重试**：不同队列有不同的重试次数（2-5次）和策略
8. **指数退避 + 全抖动**：重试延迟采用指数增长 + 随机化，避免惊群效应
9. **速率限制特殊处理**：`QueueRetryAfterError` 不消耗重试次数

### 12.3 下游任务触发

10. **非 fixMode 总是触发**：默认模式下，无论 OCR 是否成功，都会触发下游任务（下游任务自行处理无内容场景）
11. **fixMode 增量触发**：修复模式下，只有内容实际变化时才触发下游任务，避免无效计算

### 12.4 可靠性与公平性

12. **优雅降级**：嵌入失败时仍能进行 tagging（不带相似度上下文）
13. **按用户分组**：任务调度时按用户分组，确保资源公平分配
14. **信号量控制**：Restate Dispatcher 使用信号量控制并发，避免系统过载
15. **批量策略**：首次运行启用批量处理以提高性能，重试时禁用批量以提高可靠性
