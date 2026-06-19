# Karakeep AI Provider 抽象与模型切换代码分析

本文档基于代码逐行核对，深入分析 Karakeep 项目中 AI Provider 的接口抽象、模型选择、超时处理、失败兜底、Token 计费和 Worker 分层架构。

---

## 1. Provider 接口抽象层

### 1.1 核心接口 `InferenceClient`

位置：`packages/shared/inference.ts`（第 115-127 行）

`InferenceClient` 是 AI 推理能力的统一抽象，定义了三个核心方法：

```typescript
export interface InferenceClient {
  // 纯文本推理（打标 / 摘要）
  inferFromText(
    prompt: string,
    opts: Partial<InferenceOptions>,
  ): Promise<InferenceResponse>;

  // 图像推理（图片打标）
  inferFromImage(
    prompt: string,
    contentType: string,
    image: string,
    opts: Partial<InferenceOptions>,
  ): Promise<InferenceResponse>;

  // 文本嵌入向量生成
  generateEmbeddingFromText(inputs: string[]): Promise<EmbeddingResponse>;
}
```

> 注意：`generateEmbeddingFromText` 方法没有 `opts` 参数，也不支持 `abortSignal`。

### 1.2 返回值结构

位置：`packages/shared/inference.ts`（第 11-20 行）

```typescript
// 文本推理响应
export interface InferenceResponse {
  response: string;                // LLM 返回的原始文本
  totalTokens: number | undefined; // 本次调用消耗的总 Token 数
}

// 嵌入向量响应
export interface EmbeddingResponse {
  embeddings: number[][];             // 嵌入向量数组（多输入对应多向量）
  totalTokens: number | undefined;    // 总 Token 数
  promptTokens: number | undefined;   // Prompt Token 数
}
```

### 1.3 推理选项 `InferenceOptions`

位置：`packages/shared/inference.ts`（第 105-109 行）

```typescript
export interface InferenceOptions {
  schema: z.ZodSchema<any> | null;  // 结构化输出 Schema（Zod 定义）
  abortSignal?: AbortSignal;           // 取消信号（仅文本/图像推理支持）
}
```

---

## 2. Provider 选择与工厂模式

### 2.1 选择优先级

位置：`packages/shared/inference.ts`（第 154-165 行）

`InferenceClientFactory.build()` 按以下优先级选择 Provider：

```
优先级 1: openAIApiKey 已配置 → OpenAIInferenceClient
优先级 2: ollamaBaseUrl 已配置 → OllamaInferenceClient
都未配置  → 返回 null（AI 功能不可用）
```

若两者同时配置，**OpenAI 优先**。OpenAI 兼容客户端可对接任何 OpenAI API 兼容服务（Groq、Together、OpenRouter 等），通过 `OPENAI_BASE_URL` 切换端点。

### 2.2 配置驱动的模型参数

位置：`packages/shared/config.ts`（第 305-340 行）

模型相关的核心配置项：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `textModel` | `"gpt-4.1-mini"` | 文本推理模型名 |
| `imageModel` | `"gpt-4o-mini"` | 图像推理模型名 |
| `contextLength` | `2048` | 上下文长度（用于 Prompt 截断） |
| `maxOutputTokens` | `2048` | 最大输出 Token 数 |
| `outputSchema` | `"structured"` | 输出模式：`structured` / `json` / `plain` |
| `embedding.textModel` | `"text-embedding-3-small"` | 嵌入模型名 |
| `embedding.dimensions` | `1536` | 嵌入向量维度 |

### 2.3 结构化输出 Schema 映射

位置：`packages/shared/inference.ts`（第 129-137 行）

`mapInferenceOutputSchema` 是一个类型安全的映射函数，根据 `outputSchema` 配置和 Provider 特性，返回不同的结构化输出参数：

| `outputSchema` | OpenAI 实现 | Ollama 实现 |
|---|---|---|
| `"structured"` | `zodResponseFormat(schema, "schema")` | `z.toJSONSchema(schema)` |
| `"json"` | `{ type: "json_object" }` | `"json"` 字符串 |
| `"plain"` | `undefined`（无格式约束） | `undefined`（无格式约束） |

### 2.4 OpenAI Provider 实现

位置：`packages/shared/inference.ts`（第 167-317 行）

`OpenAIInferenceClient` 关键特性：

- 支持 `serviceTier`（`auto` / `default` / `flex`）用于不同服务层级
- 支持 `reasoningEffort`（`none` ~ `xhigh`）控制推理深度
- 支持 HTTP 代理（通过 `undici.ProxyAgent`）
- 区分 `max_tokens` 与 `max_completion_tokens` 两种 API 参数（由 `useMaxCompletionTokens` 配置切换）
- 默认携带 `X-Title` 和 `HTTP-Referer` 请求头

构造时设置 **SDK 级全局超时**：

```typescript
this.openAI = new OpenAI({
  apiKey: config.apiKey,
  baseURL: config.baseURL,
  timeout: config.timeoutSec !== undefined
    ? config.timeoutSec * 1000  // 来自 OPENAI_TIMEOUT_SEC
    : undefined,
  // ...
});
```

### 2.5 Ollama Provider 实现

位置：`packages/shared/inference.ts`（第 329-473 行）

`OllamaInferenceClient` 关键特性：

- 使用 `customFetch`（自定义超时 fetch）注入 SDK
- 采用**流式模式**（`stream: true`）逐块累积响应
- `keep_alive` 参数控制模型在内存中的保留时间
- 嵌入生成支持 `truncate: true` 自动截断超长输入
- 对 Ollama JS SDK 已知 Bug 做了兼容（流式异常时仍尝试使用已累积响应）

---

## 3. 超时处理机制

Karakeep 的超时设计是**分层嵌套**的，但 OpenAI 和 Ollama 走的路径不同。

### 3.1 共有层：队列 Job 级超时

位置：`apps/workers/workers/inference/inferenceWorker.ts`（第 72-76 行）

最外层是队列 Runner 级别的 Job 超时：

```typescript
{
  concurrency: serverConfig.inference.numWorkers,
  pollIntervalMs: 1000,
  timeoutSecs: serverConfig.inference.jobTimeoutSec, // 默认 30 秒
}
```

超时后 Job 中止并进入重试流程。这一层对所有 Provider 都生效。

### 3.2 共有层：`AbortSignal` 级联取消（仅推理）

位置：`packages/shared/queueing.ts`（第 34-40 行）

队列 Runner 为每个 Job 注入 `abortSignal`，沿调用链传递：

```
job.abortSignal
  └─ runOpenAI(job)
       └─ runTagging(bookmarkId, job, inferenceClient)
            └─ inferenceClient.inferFromText(prompt, { abortSignal })
                 └─ OpenAI / Ollama SDK 接收 signal
```

注意：`generateEmbeddingFromText` 方法没有 `abortSignal` 参数，嵌入生成不走这一层取消。

### 3.3 OpenAI 专属：SDK 级超时

位置：`packages/shared/inference.ts`（第 174-186 行）

OpenAI SDK 在构造时设置全局超时（`OPENAI_TIMEOUT_SEC`），所有请求（包括 `chat.completions.create` 和 `embeddings.create`）都受此超时约束。

调用时还可额外传入 `signal`（来自 `abortSignal`），两者是**或关系**——任一触发即取消。

### 3.4 Ollama 专属：`customFetch` 级超时

位置：`packages/shared/customFetch.ts`（第 1-24 行）

Ollama SDK 在构造时注入 `customFetch`，每次请求都会加上 `AbortSignal.timeout()`：

```typescript
export function createCustomFetch(fetchImpl = globalThis.fetch) {
  return function customFetch(input, init?) {
    const timeout = serverConfig.inference.fetchTimeoutSec * 1000; // 默认 300 秒
    return fetchImpl(input, {
      signal: AbortSignal.timeout(timeout),
      ...init,
    });
  };
}
```

Ollama 还有一层特殊的 AbortSignal 处理：

位置：`packages/shared/inference.ts`（第 364-370 行）

```typescript
let newAbortSignal = undefined;
if (optsWithDefaults.abortSignal) {
  newAbortSignal = AbortSignal.any([optsWithDefaults.abortSignal]);
  newAbortSignal.onabort = () => {
    this.ollama.abort();  // 显式调用 SDK 的 abort() 方法
  };
}
```

将外部传入的 `abortSignal` 包装一层，触发时调用 `ollama.abort()` 取消当前流式生成。

### 3.5 超时路径总览

```
┌──────────────────────────────────────────────────────────────┐
│                  队列 Job 超时 (30s)                         │
│            (所有 Provider，所有任务类型)                     │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 ├─【文本/图像推理】─ AbortSignal 级联取消
                 │                   (embedding 不走这一层)
                 │
                 ▼
┌─────────────────────────────┐    ┌─────────────────────────────┐
│    OpenAI SDK 超时          │    │  Ollama customFetch 超时    │
│  (OPENAI_TIMEOUT_SEC)       │    │  (INFERENCE_FETCH_TIMEOUT) │
│  所有请求(含 embedding)     │    │  所有请求(含 embedding)     │
│                             │    │                             │
│  + 调用级 AbortSignal       │    │  + 调用级 AbortSignal       │
│    (仅 inferFromText/Image) │    │    → ollama.abort()        │
│                             │    │    (仅 inferFromText/Image) │
└─────────────────────────────┘    └─────────────────────────────┘
```

---

## 4. 失败兜底与重试策略

### 4.1 队列级重试

位置：`packages/shared-server/src/queues.ts`（第 130-135 行）

两个 AI 相关队列都配置了重试：

```typescript
// 推理队列（打标 + 摘要）
export const OpenAIQueue = createDeferredQueue<ZOpenAIRequest>("openai_queue", {
  defaultJobArgs: { numRetries: 3 },
  keepFailedJobs: false,
});

// 嵌入队列
export const EmbeddingsQueue = createDeferredQueue<ZEmbeddingsRequest>("embeddings_queue", {
  defaultJobArgs: { numRetries: 3 },
  keepFailedJobs: false,
});
```

### 4.2 状态标记与永久失败

位置：`apps/workers/workers/inference/inferenceWorker.ts`（第 54-70 行）

`onError` 回调中，通过 `job.numRetriesLeft` 判断是否为最后一次重试：

- 每次失败：`failed` 计数 +1
- 最后一次重试也失败：`failed_permanent` 计数 +1，并将 bookmark 对应状态标记为 `"failure"`

`attemptMarkStatus` 会更新 `bookmarks` 表的 `taggingStatus` 或 `summarizationStatus` 字段。

### 4.3 Embedding → Tagging 降级兜底

位置：`apps/workers/workers/embeddingsWorker.ts`（第 53-64 行）

这是最关键的兜底设计：**Embedding 永久失败时，仍然调度不带向量的 Tagging，确保 bookmark 不会处于无标签状态**。

```typescript
onError: async (job) => {
  // ...
  if (job.numRetriesLeft == 0) {
    workerStatsCounter.labels("embeddings", "failed_permanent").inc();
    await attemptMarkEmbeddingStatus(job.data, "failure");
    // 降级兜底
    if (
      job.data?.type === "embed" &&
      job.data.runTaggingOnComplete !== false
    ) {
      await enqueueTaggingFallback(job);
    }
  }
},
```

`enqueueTaggingFallback` 从数据库查询 bookmark 的 `userId`，然后提交一个**不携带 `embedding` 参数**的 tagging job：

位置：`apps/workers/workers/embeddingsWorker.ts`（第 127-147 行）

```typescript
async function enqueueTaggingFallback(job) {
  const bookmarkId = job.data?.bookmarkId;
  const bookmark = await db.query.bookmarks.findFirst({
    where: eq(bookmarks.id, bookmarkId),
  });
  await enqueueTagging(bookmarkId, bookmark.userId, job.priority);
  // 注意：这里不传 embedding 参数
}
```

### 4.4 Tagging 内部：JSON 解析多层兜底

位置：`apps/workers/workers/inference/tagging.ts`（第 45-79 行）

当 LLM 不遵守结构化输出 Schema 时，`parseJsonFromLLMResponse` 提供 4 层解析兜底：

1. **直接解析**：`JSON.parse(trimmedResponse)`
2. **Markdown 代码块提取**：用正则匹配 `` ```json ... ``` `` 中的内容
3. **边界匹配**：用正则 `\{[\s\S]*\}` 查找最外层 JSON 对象边界
4. **最终重试**：再用原始响应 `JSON.parse` 一次，抛出原始错误

### 4.5 Ollama 流式异常兜底

位置：`packages/shared/inference.ts`（第 396-416 行）

Ollama JS SDK 存在已知 Bug：流式返回部分成功结果后仍可能抛出异常。代码通过 `try-catch` 包裹迭代，异常时保留已累积的响应：

```typescript
try {
  for await (const part of chatCompletion) {
    response += part.response;
    // ... 累加 token
  }
} catch (e) {
  if (e instanceof Error && e.name === "AbortError") {
    throw e;  // AbortError 正常向上抛
  }
  totalTokens = NaN;  // 异常时 token 不可信，设为 NaN
  logger.warn(`Got an exception from ollama, will still attempt to deserialize...`);
}
```

---

## 5. Token 统计

### 5.1 文本推理 Token 采集

#### OpenAI 路径

位置：`packages/shared/inference.ts`（第 241-245 行）

直接使用 SDK 返回的 `usage.total_tokens`：

```typescript
return { response, totalTokens: chatCompletion.usage?.total_tokens };
```

#### Ollama 路径

位置：`packages/shared/inference.ts`（第 378-395 行）

流式逐块累加 `eval_count` 和 `prompt_eval_count`：

```typescript
let totalTokens = 0;
for await (const part of chatCompletion) {
  response += part.response;
  if (!isNaN(part.eval_count)) {
    totalTokens += part.eval_count;        // 生成 Token 数
  }
  if (!isNaN(part.prompt_eval_count)) {
    totalTokens += part.prompt_eval_count; // Prompt Token 数
  }
}
```

注意：若流式迭代中发生非 Abort 异常，`totalTokens` 会被设为 `NaN`。

### 5.2 嵌入 Token 采集

位置：`packages/shared/inference.ts`（第 76-103 行）

`parseEmbeddingUsage` 函数兼容多种响应格式，按优先级读取：

1. 优先从 `response.usage.prompt_tokens` / `response.usage.total_tokens` 读取（OpenAI 风格）
2. 兜底从 `response.prompt_eval_count` / `response.eval_count` 读取（Ollama 风格）

两个 Provider 的 `generateEmbeddingFromText` 都调用此函数统一解析。

### 5.3 Token 与事件日志关联

位置：`packages/shared-server/src/eventLogTypes.ts`（第 10-27 行）

`inferenceWorker.run` 事件包含的 Token / 计费字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `inference.model` | `string` | 使用的模型名 |
| `inference.total_tokens` | `number` | 本次调用总 Token 数 |
| `inference.prompt.custom_count` | `number` | 自定义 Prompt 数量 |
| `inference.prompt.size` | `number` | Prompt 字节数 |
| `inference.summary.size` | `number` | 摘要结果字节数（仅 summarize） |
| `inference.tagging.num_generated_tags` | `number` | 生成的标签数（仅 tag） |
| `inference.tagging.num_potential_relevant_tags` | `number` | 相似性推荐标签数（仅 tag） |

`embeddingsWorker.run` 事件：

| 字段 | 类型 | 说明 |
|---|---|---|
| `embedding.prompt_tokens` | `number` | Prompt Token 数 |
| `embedding.total_tokens` | `number` | 总 Token 数 |
| `embedding.text_size` | `number` | 输入文本字符数 |

### 5.4 日志注入方式

位置：`packages/shared-server/src/eventLogger.ts`（第 211-220 行）

通过 `addLogFields<T>()` 渐进式填充事件日志字段，执行过程中任意位置均可追加：

```typescript
addLogFields<"inferenceWorker.run">({
  "inference.total_tokens": response.totalTokens,
  "inference.tagging.num_generated_tags": tags.length,
});
```

`withEventLog` wrapper 在函数结束时统一输出（OTLP 或 Console）。

---

## 6. 队列入口矩阵

### 6.1 四个场景的队列入口与 summarize 触发总览

| 触发场景 | 触发位置 | 进入队列路径 | 是否触发 summarize |
|---|---|---|---|
| **用户创建链接书签** | `packages/trpc/routers/bookmarks.ts`（第 395-408 行） | `LinkCrawlerQueue` / `LowPriorityCrawlerQueue` → 爬虫完成后按 `enableAutoIndexing` 分支 | ✅ 是 |
| **用户创建文本书签** | `packages/trpc/routers/bookmarks.ts`（第 410-428 行） | 直接进入 `EmbeddingsQueue(embed)` 或 `OpenAIQueue(tag)` | ❌ 否 |
| **用户创建资产书签** | `packages/trpc/routers/bookmarks.ts`（第 431-439 行） | `AssetPreprocessingQueue` → 预处理完成后按 `enableAutoIndexing` 分支 | ✅ 是 |
| **管理端重跑** | `packages/trpc/routers/admin.ts`（第 277-420 行） | 直接进入对应队列 | 取决于具体操作 |

### 6.2 场景 1：用户创建链接书签（Crawler 路径）

**触发代码**：`packages/trpc/routers/bookmarks.ts`（第 395-408 行）

```typescript
case BookmarkTypes.LINK: {
  const crawlerQueue = shouldUseLowPriority
    ? LowPriorityCrawlerQueue
    : LinkCrawlerQueue;
  await crawlerQueue.enqueue(
    { bookmarkId: bookmark.id },
    enqueueOpts,
  );
  break;
}
```

**Crawler 完成后的分支**：`apps/workers/workers/crawlerWorker.ts`（第 2312-2338 行）

```typescript
if (job.data.runInference !== false) {
  if (serverConfig.embedding.enableAutoIndexing) {
    // 分支 1：走 embedding 路径
    await EmbeddingsQueue.enqueue(
      { bookmarkId, type: "embed", runTaggingOnComplete: true },
      enqueueOpts,
    );
  } else {
    // 分支 2：直接触发 tagging
    await OpenAIQueue.enqueue({ bookmarkId, type: "tag" }, enqueueOpts);
  }
  // summarize 始终独立触发
  await OpenAIQueue.enqueue({ bookmarkId, type: "summarize" }, enqueueOpts);
}
```

> 注意：Crawler 在抓取时如果发现链接实际是一个资产（PDF/图片），会自动将 bookmark 类型从 `LINK` 改为 `ASSET`，并触发 `AssetPreprocessingQueue`（`apps/workers/workers/crawlerWorker.ts` 第 1744 行）。

### 6.3 场景 2：用户创建文本书签

**触发代码**：`packages/trpc/routers/bookmarks.ts`（第 410-428 行）

```typescript
case BookmarkTypes.TEXT: {
  if (serverConfig.embedding.enableAutoIndexing) {
    await EmbeddingsQueue.enqueue(
      { bookmarkId: bookmark.id, type: "embed", runTaggingOnComplete: true },
      enqueueOpts,
    );
  } else {
    await OpenAIQueue.enqueue(
      { bookmarkId: bookmark.id, type: "tag" },
      enqueueOpts,
    );
  }
  break;
}
```

文本没有内容抓取步骤，创建后直接进入 AI 队列。**Search 索引**通过 `triggerSearchReindex()` 触发（第 455 行）。

### 6.4 场景 3：用户创建资产书签 + 资产预处理

**触发代码**：`packages/trpc/routers/bookmarks.ts`（第 431-439 行）

```typescript
case BookmarkTypes.ASSET: {
  await AssetPreprocessingQueue.enqueue(
    { bookmarkId: bookmark.id, fixMode: false },
    enqueueOpts,
  );
  break;
}
```

**资产预处理 Worker 路径**：`apps/workers/workers/assetPreprocessingWorker.ts`（第 384-504 行）

`AssetPreprocessingWorker` 根据资产类型执行不同预处理：

| 资产类型 | 预处理操作 |
|---|---|
| `image` | 提取文本：优先 LLM OCR → 降级 Tesseract OCR |
| `pdf` | 1. 提取文本（`pdf-parse`）<br>2. 生成 PDF 首屏截图（`playwright`） |

预处理完成后，非 `fixMode` 或内容有变化时，触发后续 AI 队列（第 475-503 行）：

```typescript
if (!isFixMode || anythingChanged) {
  if (serverConfig.embedding.enableAutoIndexing) {
    await EmbeddingsQueue.enqueue(
      { bookmarkId, type: "embed", runTaggingOnComplete: true },
      enqueueOpts,
    );
  } else {
    await OpenAIQueue.enqueue(
      { bookmarkId, type: "tag" },
      enqueueOpts,
    );
  }
  await OpenAIQueue.enqueue(
    { bookmarkId, type: "summarize" },
    enqueueOpts,
  );
  await triggerSearchReindex(bookmarkId, enqueueOpts);
}
```

> 资产预处理也会调用 LLM（图像 OCR），使用与 tagging 同一个 `InferenceClientFactory.build()` 客户端，但超时和重试走 `AssetPreprocessingQueue` 自己的配置（`numRetries: 5`，`jobTimeoutSec: 300`）。

### 6.5 场景 4：管理端重跑

**触发代码**：`packages/trpc/routers/admin.ts`

管理端提供 5 个重跑入口，直接操作不同队列：

| 管理端操作 | 代码位置 | 进入队列 |
|---|---|---|
| `recrawlLinks` | 第 247-276 行 | `LowPriorityCrawlerQueue`（携带 `runInference` 参数） |
| `reindexAllBookmarks` | 第 277-293 行 | 先 `clearIndex()` 清空，再 `triggerSearchReindex()` → `SearchIndexingQueue(index)` |
| `regenerateAllBookmarkEmbeddings` | 第 294-384 行 | 先可选 `clearIndex()`，批量翻页更新 `embeddingStatus` 为 `pending`，再 `EmbeddingsQueue(embed, force: true, runTaggingOnComplete: false)` |
| `reprocessAssetsFixMode` | 第 365-384 行 | `AssetPreprocessingQueue(fixMode: true)` |
| `reRunInferenceOnAllBookmarks` | 第 386-420 行 | 按 `tag` / `summarize` 类型筛选，直接 `OpenAIQueue(type)` |

> 注意：`regenerateAllBookmarkEmbeddings` 会设置 `runTaggingOnComplete: false`，即重跑 embedding 时**不会自动触发重打标签**，避免重复消费 Token。

### 6.6 搜索索引队列 `SearchIndexingQueue`

**定义**：`packages/shared-server/src/queues.ts`（第 183-191 行）

```typescript
export const SearchIndexingQueue = createDeferredQueue<ZSearchIndexingRequest>(
  "searching_indexing",
  { defaultJobArgs: { numRetries: 5 }, keepFailedJobs: false },
);
```

**触发函数 `triggerSearchReindex`**：`packages/shared-server/src/queues.ts`（第 231-245 行）

```typescript
export async function triggerSearchReindex(bookmarkId, opts?) {
  await SearchIndexingQueue.enqueue(
    { bookmarkId, type: "index" },
    { ...opts, idempotencyKey: `index:${bookmarkId}` },
  );
}
```

**触发场景**：
1. 创建 bookmark 后：`packages/trpc/routers/bookmarks.ts` 第 455 行
2. 更新 bookmark 后：第 638 行
3. 打标签完成后：`apps/workers/workers/inference/tagging.ts` 第 723 行
4. 资产预处理完成后：`apps/workers/workers/assetPreprocessingWorker.ts` 第 503 行
5. 管理端 `reindexAllBookmarks`：`packages/trpc/routers/admin.ts` 第 288 行
6. 删除 bookmark 时：`packages/trpc/models/bookmarks.ts` 第 935 行（`type: "delete"`）

**Worker 执行**：`apps/workers/workers/searchWorker.ts`

- `type: "index"`：从 DB 读取 bookmark 完整信息（含链接/文本/资产/标签），组装成 `BookmarkSearchDocument`，调用 `searchClient.addDocuments()`
- `type: "delete"`：调用 `searchClient.deleteDocuments()`

重试时（`runNumber > 0`）禁用批量写入以提高可靠性。

### 6.7 完整队列流转图

```
用户创建 bookmark
     │
     ├─ type == LINK  ──> LinkCrawlerQueue ─┐
     │                                       │
     ├─ type == TEXT  ───────────────────────┤
     │                                       │
     └─ type == ASSET ──> AssetPreprocessingQueue ──┐
                                                     │
                     ┌───────────────────────────────┘
                     │
                     ▼
        enableAutoIndexing ?
             │       │
             │ Yes   ▼ No
             │    ┌───────────────────────────┐
             │    │ OpenAIQueue(tag)         │
             │    └───────────────────────────┘
             │
             ▼
    ┌─────────────────────────────────────────┐
    │ EmbeddingsQueue(embed)                  │
    │  ├─ 生成 embedding 向量                │
    │  ├─ 成功 → EmbeddingsQueue(index)       │
    │  │       └─ vectorStore.addVectors()    │
    │  ├─ 成功 → OpenAIQueue(tag + embedding) │
    │  │       └─ 带向量相似性上下文打标签     │
    │  └─ 永久失败 → enqueueTaggingFallback() │
    │          └─ OpenAIQueue(tag)            │
    └─────────────────────────────────────────┘
                     │
                     └─ OpenAIQueue(summarize) ← 始终独立触发（LINK 和 ASSET）
                     │
                     └─ triggerSearchReindex()
                           └─ SearchIndexingQueue(index)
```

---

## 7. Embedding 维度配置的实际作用

### 7.1 维度配置来源

位置：`packages/shared/config.ts`（第 336 行）

```typescript
dimensions: val.EMBEDDING_DIMENSIONS,  // 默认 1536
```

### 7.2 在 Meilisearch 向量库中的作用

位置：`packages/plugins/vectorstore-meilisearch/src/index.ts`（第 255-280 行）

Meilisearch 插件在 `ensureInit()` 启动时检查并配置 embedder 维度：

```typescript
const desiredDimensions = serverConfig.embedding.dimensions;
const currentEmbedder = currentEmbedders?.default;

if (
  !currentEmbedder ||
  currentEmbedder.source != "userProvided" ||
  currentEmbedder.dimensions !== desiredDimensions
) {
  await index.updateEmbedders({
    default: {
      source: "userProvided",
      dimensions: desiredDimensions,
    },
  }).waitTask();
}
```

**作用 1：向量库 Schema 校验**

Meilisearch 的 `userProvided` embedder 要求预先声明维度。写入向量时，Meilisearch 会校验每个向量的长度是否匹配 `dimensions` 设置，不匹配会拒绝写入。

**作用 2：切换模型时自动重建**

当切换到不同维度的嵌入模型（如从 `text-embedding-3-small` 的 1536 维 → `text-embedding-3-large` 的 3072 维），只需修改 `EMBEDDING_DIMENSIONS` 环境变量，Meilisearch 会在启动时自动检测到维度变化并更新 embedder 配置。

> 注意：Meilisearch 更改 embedder 维度后，**已有的向量数据不会自动重建**。需要管理员调用 `regenerateAllBookmarkEmbeddings` 接口重新生成所有 embedding。

**作用 3：Tagging 中的向量相似性搜索**

位置：`apps/workers/workers/inference/tagging.ts`（第 558-580 行）

`getPotentiallyRelevantTags` 函数使用 embedding 向量做相似性搜索时，传入的向量长度必须与 Meilisearch 中配置的 `dimensions` 一致，否则搜索会失败：

```typescript
const similarBookmarkIds =
  embedding && embedding.length > 0
    ? await client.search({
        vector: embedding,       // 长度必须等于 dimensions
        limit: 11,
        filter: userFilter,
        rankingScoreThreshold: RELEVANT_TAG_SCORE_THRESHOLD,
      })
    : await client.findSimilar({ ... });
```

### 7.3 与 `InferenceClient` 的关系

注意：`dimensions` 配置**不会传递给嵌入模型 API**。对于 OpenAI 兼容 API，部分模型（如 `text-embedding-3-small`）支持通过 `dimensions` 请求参数裁剪维度，但 Karakeep 目前**没有使用这个参数**，而是依赖模型的原生输出维度。

`EMBEDDING_DIMENSIONS` 配置的主要作用是**告知向量存储后端预期的向量长度**，它需要与 `EMBEDDING_TEXT_MODEL` 配置的模型实际输出维度保持一致。

---

## 8. Worker 分层架构

### 8.1 整体架构（与第 6 章队列流转一致）

本章不再重复第 6 章已详述的队列入口矩阵和流转图，仅补充 Worker 内部的分层职责和关键设计要点。

**核心分层总览**：

```
┌───────────────────────────────────────────────────────────────┐
│  API / tRPC 层                                                 │
│  (创建 bookmark，根据类型 enqueue 到对应队列)                  │
└─────────────────────────────┬─────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│  预处理 Worker 层                                              │
│  ├─ crawlerWorker.ts         LINK 类型：抓取网页内容          │
│  └─ assetPreprocessingWorker.ts  ASSET 类型：OCR/文本提取     │
└─────────────────────────────┬─────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│  AI Worker 层                                                  │
│  ├─ embeddingsWorker.ts      向量生成 + 向量入库（解耦）      │
│  ├─ inferenceWorker.ts       统一调度 tagging / summarize     │
│  │  ├─ tagging.ts            打标业务逻辑（含 JSON 兜底）     │
│  │  └─ summarize.ts          摘要业务逻辑                     │
│  └─ searchWorker.ts          全文搜索索引（独立队列）         │
└───────────────────────────────────────────────────────────────┘
```

### 8.2 五个 Worker 的职责对比（含 summarize 触发）

| Worker | 处理类型 | 队列配置 | 主要职责 | 是否触发 summarize |
|---|---|---|---|---|
| **crawlerWorker** | LINK | `LinkCrawlerQueue`（numRetries=5）<br>`LowPriorityCrawlerQueue` | 抓取网页内容、截图、PDF 存档；<br>完成后根据 `enableAutoIndexing` 分支触发 embedding/tagging + **summarize** | ✅ 是 |
| **assetPreprocessingWorker** | ASSET | `AssetPreprocessingQueue`（numRetries=5，jobTimeout=300s） | 图像 OCR（LLM → Tesseract 降级）、PDF 文本提取 + 首屏截图；<br>完成后根据 `enableAutoIndexing` 分支触发 embedding/tagging + **summarize** | ✅ 是 |
| **embeddingsWorker** | ALL（TEXT/LINK/ASSET） | `EmbeddingsQueue`（numRetries=3） | 生成 embedding 向量 → 触发 `EmbeddingsQueue(index)` + `OpenAIQueue(tag)`；<br>永久失败 → `enqueueTaggingFallback()` 降级为无向量 tag | ❌ 否（summarize 由预处理层触发） |
| **inferenceWorker** | ALL | `OpenAIQueue`（numRetries=3） | 统一调度：`type="tag"` → tagging.ts；`type="summarize"` → summarize.ts | -（被调用者，不是触发者） |
| **searchWorker** | ALL | `SearchIndexingQueue`（numRetries=5） | 全文搜索索引的增删改，与 AI 队列完全解耦 | ❌ 否 |

> 注意 1：文本书签（TEXT 类型）创建时**不会触发 summarize**，只触发 embedding/tagging。summarize 只有 LINK 和 ASSET 类型在预处理完成后才会触发。
>
> 注意 2：summarize 与 tagging 完全独立，失败不会互相影响。tagging 失败不影响 summarize，反之亦然。

### 8.3 Inference Worker 调度层

位置：`apps/workers/workers/inference/inferenceWorker.ts`（第 44-124 行）

`OpenAiWorker.build()` 职责：

1. 创建队列 Runner，配置并发数、轮询间隔、Job 超时
2. 包装 `withWorkerTracing` + `withWorkerEventLog` middleware
3. `runOpenAI` 内部构建 `InferenceClientFactory.build()`，按 `type` 分发

```typescript
run: withWorkerTracing(
  "inferenceWorker.run",
  withWorkerEventLog("inferenceWorker.run", runOpenAI),
),
```

### 8.4 Summarize Worker 业务逻辑

位置：`apps/workers/workers/inference/summarize.ts`

`runSummarize` 执行流程：

```
1. 全局开关检查 (enableAutoSummarization)
2. 用户级开关检查 (autoSummarizationEnabled)
3. 根据内容类型读取内容:
   ├─ link/text → bookmark.content.content
   └─ asset:pdf → 从 contentAsset 读取 PDF 文本
4. 调用 inferenceClient.inferFromText() 生成摘要
5. 将摘要写入 bookmarks.summary 字段
6. 触发 Search 重索引
```

### 8.5 Tagging 业务层

位置：`apps/workers/workers/inference/tagging.ts`（第 619-728 行）

`runTagging` 执行流程：

```
1. 全局开关检查 (enableAutoTagging)
2. 用户级开关检查 (autoTaggingEnabled)
3. 读取用户偏好:
   ├─ tagStyle: "as-generated" / "curated"
   ├─ inferredTagLang: 语言
   └─ curatedTagIds: 精选标签 ID 列表
4. 构建 Prompt 上下文:
   ├─ 有 curatedTagIds → 直接使用精选标签
   └─ 无 → getPotentiallyRelevantTags() 向量相似性推荐
      ├─ 有 embedding 参数 → search({vector}) 无需等待索引
      └─ 无 → findSimilar({id}) 需已索引
5. 按内容类型分发:
   ├─ link/text → inferTagsFromText()
   ├─ asset:image → inferTagsFromImage()
   └─ asset:pdf → inferTagsFromPDF()
6. parseJsonFromLLMResponse() 解析 + Zod 校验
7. connectTags() 数据库事务:
   ├─ 匹配现有标签（按 normalizedName）
   ├─ 创建不存在的新标签
   ├─ 删除旧的 AI 标签关联
   └─ 插入新的 AI 标签关联
8. 触发 RuleEngine + Webhook + Search 重索引
```

### 8.6 Embeddings Worker 层（重试隔离设计）

位置：`apps/workers/workers/embeddingsWorker.ts`（第 394-498 行）

这是最精巧的设计——**将向量生成与向量入库解耦为两个独立 Job，各自拥有独立的重试域**：

| Job 类型 | 职责 | 失败影响 |
|---|---|---|
| `type: "embed"` | 调用 LLM 生成 embedding 向量，然后分发 `index` 和 `tag` | 失败会触发 fallback tagging |
| `type: "index"` | 将预生成的向量写入向量存储（Meilisearch） | 失败仅影响向量搜索，不影响 tagging |
| `type: "delete"` | 从向量存储中删除向量 | - |

设计收益：向量入库（通常依赖外部 Meilisearch，可能很慢）即使失败重试，也**绝不会重复触发 tagging**，避免重复消费 Token 和产生重复标签。

### 8.7 插件化的 Queue Provider

位置：`packages/shared/plugins.ts`

队列本身也是插件化的，通过 `PluginManager` 管理：

位置：`packages/shared-server/src/plugins.ts`（第 16-43 行）

```typescript
// 加载顺序（后者优先，Last one wins）
await import("@karakeep/plugins/queue-liteque");   // 内存队列（默认）
await import("@karakeep/plugins/queue-restate");   // Restate 分布式队列
```

`PluginManager.getClient(PluginType.Queue)` 返回最后注册的 provider，实现队列后端可插拔切换。

### 8.8 Summarize 触发关系总结（全场景核对）

| 场景 | 是否触发 summarize | 触发者 | 代码位置 |
|---|---|---|---|
| 创建文本书签 | ❌ 否 | - | `packages/trpc/routers/bookmarks.ts` 第 410-428 行 |
| 链接抓取完成 | ✅ 是 | crawlerWorker | `apps/workers/workers/crawlerWorker.ts` 第 2331-2337 行 |
| 资产预处理完成 | ✅ 是 | assetPreprocessingWorker | `apps/workers/workers/assetPreprocessingWorker.ts` 第 494-500 行 |
| 管理端 `reRunInferenceOnAllBookmarks(type="summarize")` | ✅ 是 | admin router | `packages/trpc/routers/admin.ts` 第 412 行 |
| 管理端 `reRunInferenceOnAllBookmarks(type="tag")` | ❌ 否 | - | - |
| 管理端 `regenerateAllBookmarkEmbeddings` | ❌ 否 | - | `runTaggingOnComplete: false` 且不 enqueue summarize |
| 管理端 `recrawlLinks(runInference=true)` | ✅ 是 | crawlerWorker | 间接触发 |
| 管理端 `reprocessAssetsFixMode` | ✅ 是 | assetPreprocessingWorker | `!isFixMode || anythingChanged` 成立 |
| 管理端 `reindexAllBookmarks` | ❌ 否 | - | - |

---

## 9. 关键配置项总览

| 环境变量 | 默认值 | 作用 |
|---|---|---|
| `OPENAI_API_KEY` | - | OpenAI 兼容 API Key（存在则优先使用 OpenAI Provider） |
| `OPENAI_BASE_URL` | - | 兼容 API Base URL（切换 Groq/Together 等） |
| `OPENAI_PROXY_URL` | - | HTTP 代理 URL |
| `OPENAI_TIMEOUT_SEC` | - | OpenAI SDK 级超时（秒），所有请求生效 |
| `OPENAI_SERVICE_TIER` | - | `auto` / `default` / `flex` |
| `OPENAI_REASONING_EFFORT` | - | `none` ~ `xhigh` 推理深度 |
| `OLLAMA_BASE_URL` | - | Ollama 本地服务 URL（存在则次优先） |
| `OLLAMA_KEEP_ALIVE` | - | 模型内存保活时间 |
| `INFERENCE_JOB_TIMEOUT_SEC` | `30` | Worker 级 Job 超时（秒） |
| `INFERENCE_FETCH_TIMEOUT_SEC` | `300` | Ollama Fetch 级超时（秒） |
| `INFERENCE_TEXT_MODEL` | `"gpt-4.1-mini"` | 文本推理模型 |
| `INFERENCE_IMAGE_MODEL` | `"gpt-4o-mini"` | 图像推理模型 |
| `INFERENCE_CONTEXT_LENGTH` | `2048` | Prompt 上下文长度 |
| `INFERENCE_MAX_OUTPUT_TOKENS` | `2048` | 最大输出 Token |
| `INFERENCE_OUTPUT_SCHEMA` | `"structured"` | 输出模式 |
| `INFERENCE_NUM_WORKERS` | `1` | 推理并发 Worker 数 |
| `INFERENCE_ENABLE_AUTO_TAGGING` | `"true"` | 是否启用自动打标 |
| `INFERENCE_ENABLE_AUTO_SUMMARIZATION` | `"false"` | 是否启用自动摘要 |
| `EMBEDDING_TEXT_MODEL` | `"text-embedding-3-small"` | 嵌入模型 |
| `EMBEDDING_DIMENSIONS` | `1536` | 嵌入维度（需与模型实际输出一致） |
| `EMBEDDING_ENABLE_AUTO_INDEXING` | `"false"` | 是否启用自动向量索引 |
| `EMBEDDING_JOB_TIMEOUT_SEC` | `60` | Embedding Job 超时 |
| `EMBEDDING_NUM_WORKERS` | `1` | Embedding Worker 数 |

---

## 10. 核心设计思想总结

### 10.1 分层关注点分离

| 层次 | 职责 | 关键文件 |
|---|---|---|
| **接口抽象层** | `InferenceClient` 统一三能力接口 | `packages/shared/inference.ts` |
| **Provider 实现层** | OpenAI / Ollama 差异封装 | `packages/shared/inference.ts` |
| **配置驱动层** | 环境变量 → 强类型配置 | `packages/shared/config.ts` |
| **队列调度层** | 重试 / 超时 / 并发控制 | `packages/shared-server/src/queues.ts` |
| **Worker 执行层** | 业务逻辑（打标/摘要/嵌入/搜索索引） | `apps/workers/workers/inference/tagging.ts` · `apps/workers/workers/inference/summarize.ts` · `apps/workers/workers/embeddingsWorker.ts` · `apps/workers/workers/searchWorker.ts` · `apps/workers/workers/assetPreprocessingWorker.ts` |
| **可观测性层** | Token 统计 / 链路追踪 / 事件日志 | `packages/shared-server/src/eventLogger.ts` + `packages/shared-server/src/tracing.ts` |

### 10.2 弹性设计要点

1. **分层超时**：Job 级 → AbortSignal → SDK/Fetch 级，每层独立可控，覆盖不同故障场景
2. **Embed / Index 职责拆分**：向量生成与向量入库分属两个 Job，重试域隔离，避免慢操作拖累快速路径
3. **Embedding → Tagging 降级**：Embedding 永久失败时自动降级为无向量 Tagging，功能不中断
4. **JSON 解析多层兜底**：对 LLM 不遵守 Schema 的常见问题做了充分容错
5. **Ollama 流式异常兼容**：针对已知 SDK Bug 做了 Graceful Degrade
6. **插件化 Provider**：Queue / VectorStore / Search / RateLimit 全部可热插拔切换后端
7. **summarize 独立触发**：与 embedding 解耦，不受 embedding 开关和失败影响
8. **搜索索引独立队列**：`SearchIndexingQueue` 有自己的重试策略（`numRetries: 5`）和 Worker，不与 AI 推理队列争抢资源
9. **资产预处理 + AI 流水线**：`AssetPreprocessingQueue` 完成文本提取/OCR 后再触发 AI 队列，且自己也可调用 LLM 做图像 OCR
10. **维度一致性保障**：`EMBEDDING_DIMENSIONS` 配置与 Meilisearch embedder 强校验，确保写入/搜索时向量长度一致
