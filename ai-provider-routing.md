# Karakeep AI Provider 抽象与模型切换代码分析

本文档深入分析 Karakeep 项目中 AI Provider 的抽象设计、模型选择、超时处理、失败兜底、Token 计费和 Worker 分层架构。

---

## 1. Provider 接口定义与抽象层

### 1.1 核心接口 `InferenceClient`

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L115-L127)

`InferenceClient` 是整个 AI 推理抽象的核心接口，定义了三个能力：

```typescript
export interface InferenceClient {
  // 纯文本推理（打标 / 摘要）
  inferFromText(prompt: string, opts: Partial<InferenceOptions>): Promise<InferenceResponse>;
  
  // 图像推理（图片打标）
  inferFromImage(prompt: string, contentType: string, image: string, opts: Partial<InferenceOptions>): Promise<InferenceResponse>;
  
  // 文本嵌入向量生成
  generateEmbeddingFromText(inputs: string[]): Promise<EmbeddingResponse>;
}
```

### 1.2 返回值结构

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L11-L20)

```typescript
// 推理响应：文本+Token
export interface InferenceResponse {
  response: string;                // LLM 原始返回文本
  totalTokens: number | undefined;    // 本次调用总 Token 数
}

// 嵌入响应：向量数组+Token
export interface EmbeddingResponse {
  embeddings: number[][];             // 嵌入向量数组（多个输入对应多个向量）
  totalTokens: number | undefined;       // 总 Token 数
  promptTokens: number | undefined;    // prompt Token 数
}
```

### 1.3 推理选项 `InferenceOptions`

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L105-L109)

```typescript
export interface InferenceOptions {
  schema: z.ZodSchema<any> | null;        // 结构化输出 Schema（Zod）
  abortSignal?: AbortSignal;                     // 外部传入的取消信号
}
```

---

## 2. 模型选择与工厂模式

### 2.1 Provider 工厂 `InferenceClientFactory`

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L154-L165)

```typescript
export class InferenceClientFactory {
  static build(): InferenceClient | null {
    // 优先级 1：OpenAI API Key 已配置 → 使用 OpenAI 兼容客户端
    if (serverConfig.inference.openAIApiKey) {
      return OpenAIInferenceClient.fromConfig();
    }

    // 优先级 2：Ollama Base URL 已配置 → 使用本地 Ollama
    if (serverConfig.inference.ollamaBaseUrl) {
      return OllamaInferenceClient.fromConfig();
    }

    // 均未配置 → 返回 null（AI 功能不可用）
    return null;
  }
}
```

> **设计决策**：优先顺序是 OpenAI → Ollama → null。OpenAI 兼容客户端可对接任何 OpenAI API 兼容服务（如 Groq、Together、OpenRouter 等），通过 `OPENAI_BASE_URL` 切换。

### 2.2 配置驱动的模型选择

**文件位置**: [config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/config.ts#L305-L340)

```typescript
inference: {
  isConfigured: !!val.OPENAI_API_KEY || !!val.OLLAMA_BASE_URL,
  textModel: val.INFERENCE_TEXT_MODEL,       // 默认 "gpt-4.1-mini"
  imageModel: val.INFERENCE_IMAGE_MODEL,     // 默认 "gpt-4o-mini"
  contextLength: val.INFERENCE_CONTEXT_LENGTH, // 默认 2048
  maxOutputTokens: val.INFERENCE_MAX_OUTPUT_TOKENS, // 默认 2048
  outputSchema: ...                           // structured / json / plain
  // ...
},
embedding: {
  textModel: val.EMBEDDING_TEXT_MODEL,         // 默认 "text-embedding-3-small"
  dimensions: val.EMBEDDING_DIMENSIONS,         // 默认 1536
  // ...
}
```

### 2.3 输出 Schema 映射

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L129-L137)

通过 `mapInferenceOutputSchema` 统一适配不同 Provider 的结构化输出方式：

| outputSchema 模式 | OpenAI 实现 | Ollama 实现 |
|---|---|---|
| `structured` | `zodResponseFormat(schema)` 原生结构化输出 | `z.toJSONSchema(schema)` Zod 4 JSON Schema |
| `json` | `{ type: "json_object" }` | `"json"` 字符串格式 |
| `plain` | undefined（无格式约束） | undefined（无格式约束） |

### 2.4 OpenAI Provider 实现

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L167-L317)

`OpenAIInferenceClient` 关键特性：
- 支持 `serviceTier`（`auto`/`default`/`flex`）用于不同服务层级
- 支持 `reasoningEffort`（`none`~`xhigh`）控制推理深度
- 支持 HTTP 代理（`undici.ProxyAgent`）
- 区分 `max_tokens` 与 `max_completion_tokens` 两种 API 参数
- 默认携带 `X-Title` 和 `HTTP-Referer` 请求头（标识 Karakeep 应用）

### 2.5 Ollama Provider 实现

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L329-L473)

`OllamaInferenceClient` 关键特性：
- 使用 `customFetch`（自定义超时 fetch）
- 使用流式模式（`stream: true`）+ 逐块累积响应
- **Ollama Bug 兼容**：流式处理中即使抛出异常也尝试使用已累积的响应
- `keep_alive` 参数控制模型在内存中的保留时间
- 嵌入生成支持 `truncate: true` 自动截断超长输入

---

## 3. 超时处理机制（四层超时）

Karakeep 采用 **四层嵌套超时** 设计，从外到内逐层收敛：

### 3.1 第一层：Worker 级 Job 超时

**文件位置**: [inferenceWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/inference/inferenceWorker.ts#L72-L76)

```typescript
{
  concurrency: serverConfig.inference.numWorkers,
  pollIntervalMs: 1000,
  timeoutSecs: serverConfig.inference.jobTimeoutSec, // 默认 30 秒
}
```

由队列执行器在 Job 级别强制中止，超时后 Job 进入重试流程。

### 3.2 第二层：AbortSignal 级联取消

**文件位置**: [queueing.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/queueing.ts#L34-L40)

```typescript
export interface DequeuedJob<T> {
  id: string;
  data: T;
  priority: number;
  runNumber: number;
  abortSignal: AbortSignal;  // 由队列 Runner 注入的取消信号
}
```

队列 Runner 将 Job 的 `abortSignal` 传递到业务代码，再层层传入推理客户端：

```
Job.abortSignal
  └─> runOpenAI(job)
        └─> runTagging(bookmarkId, job, inferenceClient)
              └─> inferenceClient.inferFromText(..., { abortSignal: job.abortSignal })
                    └─> OpenAI SDK / Ollama SDK abort
```

### 3.3 第三层：OpenAI SDK 级超时

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L174-L186)

```typescript
this.openAI = new OpenAI({
  apiKey: config.apiKey,
  timeout: config.timeoutSec !== undefined 
    ? config.timeoutSec * 1000  // OPENAI_TIMEOUT_SEC（可独立配置）
    : undefined,
  // ...
});
```

### 3.4 第四层：Ollama 自定义 Fetch 超时

**文件位置**: [customFetch.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/customFetch.ts#L1-L24)

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

Ollama SDK 在构造时注入此 `customFetch`：

```typescript
this.ollama = new Ollama({
  host: config.baseUrl,
  fetch: customFetch,
});
```

> **注意**：Ollama 的 AbortSignal 处理还有特殊逻辑：

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L364-L370)

```typescript
let newAbortSignal = undefined;
if (optsWithDefaults.abortSignal) {
  // 将外部传入的 signal 合并（any = 任一触发即取消）
  newAbortSignal = AbortSignal.any([optsWithDefaults.abortSignal]);
  newAbortSignal.onabort = () => {
    this.ollama.abort();  // 显式调用 ollama.abort() 取消当前生成
  };
}
```

---

## 4. 失败兜底与重试策略

### 4.1 队列级重试（Queue + numRetries）

**文件位置**: [queues.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared-server/src/queues.ts#L130-L135)

```typescript
export const OpenAIQueue = createDeferredQueue<ZOpenAIRequest>("openai_queue", {
  defaultJobArgs: {
    numRetries: 3,   // 最多重试 3 次
  },
  keepFailedJobs: false,
});
```

`EmbeddingsQueue` 同样配置 `numRetries: 3`。

### 4.2 错误回调与状态标记

**文件位置**: [inferenceWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/inference/inferenceWorker.ts#L54-L70)

```typescript
onComplete: async (job) => {
  workerStatsCounter.labels("inference", "completed").inc();
  await attemptMarkStatus(job.data, "success"); // 标记 bookmark 状态为 success
},
onError: async (job) => {
  workerStatsCounter.labels("inference", "failed").inc();
  if (job.numRetriesLeft == 0) {
    // 最后一次重试也失败，永久失败
    workerStatsCounter.labels("inference", "failed_permanent").inc();
    await attemptMarkStatus(job?.data, "failure"); // 标记 bookmark 状态为 failure
  }
},
```

`attemptMarkStatus` 更新 `bookmarks` 表的 `taggingStatus` / `summarizationStatus` 字段。

### 4.3 Embedding → Tagging 降级兜底

**文件位置**: [embeddingsWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/embeddingsWorker.ts#L47-L64)

这是最关键的兜底逻辑：当 Embedding 永久失败时，**仍然调度不带 embedding 的 tagging**，避免 bookmark 处于无标签状态：

```typescript
onError: async (job) => {
  // ...
  if (job.numRetriesLeft == 0) {
    workerStatsCounter.labels("embeddings", "failed_permanent").inc();
    await attemptMarkEmbeddingStatus(job.data, "failure");
    // 降级兜底：即使 embedding 永久失败，也要打标签（无相似性上下文）
    if (
      job.data?.type === "embed" &&
      job.data.runTaggingOnComplete !== false
    ) {
      await enqueueTaggingFallback(job);  // ← 关键兜底逻辑
    }
  }
},
```

`enqueueTaggingFallback` 从数据库查找 bookmark 的 userId，然后提交不携带 embedding 的 tagging job：

**文件位置**: [embeddingsWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/embeddingsWorker.ts#L127-L147)

```typescript
async function enqueueTaggingFallback(job) {
  const bookmarkId = job.data?.bookmarkId;
  // 查库获取 userId
  const bookmark = await db.query.bookmarks.findFirst({ where: eq(bookmarks.id, bookmarkId) });
  // 提交不带 embedding 的 tagging job
  await enqueueTagging(bookmarkId, bookmark.userId, job.priority);
}
```

### 4.4 正常流程 vs 降级流程对比

```
正常流程（Embedding 成功）:
  Crawler ──> EmbeddingsQueue(embed)
                ├─> 生成 embedding 成功
                ├─> EmbeddingsQueue(index)    ──> 向量入库
                └─> OpenAIQueue(tag + embedding) ──> 带相似性上下文打标签

降级流程（Embedding 永久失败）:
  Crawler ──> EmbeddingsQueue(embed)
                ├─> 重试 3 次全部失败
                ├─> embeddingStatus = "failure"
                └─> enqueueTaggingFallback()
                     └─> OpenAIQueue(tag) ──> 不带相似性上下文打标签
```

### 4.5 Tagging 内部 JSON 解析兜底

**文件位置**: [tagging.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/inference/tagging.ts#L45-L79)

当 LLM 不遵守结构化输出 Schema 时（即使 LLM 返回了非预期格式，`parseJsonFromLLMResponse` 提供 4 层解析兜底：

1. 直接 `JSON.parse(response)
2. 从 Markdown 代码块 ```json ... ``` 中提取
3. 用正则查找最外层 `{...}` 边界匹配
4. 最后尝试原始 parse 抛原始错误

---

## 5. Token 计费与可观测性

### 5.1 Token 数据采集

#### 文本推理 Token

**OpenAI**: 直接使用 SDK 返回的 `usage.total_tokens`

[inference.ts L245, L302

```typescript
return { response, totalTokens: chatCompletion.usage?.total_tokens };
```

**Ollama**: 流式逐块累加 `eval_count` + `prompt_eval_count`

[inference.ts L394-L405

```typescript
let totalTokens = 0;
for await (const part of chatCompletion) {
  response += part.response;
  if (!isNaN(part.eval_count)) {
    totalTokens += part.eval_count;       // 生成 Token
  }
  if (!isNaN(part.prompt_eval_count)) {
    totalTokens += part.prompt_eval_count; // Prompt Token
  }
}
```

#### 嵌入 Token 采集

**文件位置**: [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts#L76-L103)

`parseEmbeddingUsage` 兼容多种响应格式：

```typescript
// 优先从 response.usage.prompt_tokens / total_tokens
// 兜底从 response.prompt_eval_count / eval_count
```

### 5.2 Token 字段与事件日志关联

**文件位置**: [eventLogTypes.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared-server/src/eventLogTypes.ts#L10-L27)

`inferenceWorker.run` 事件类型包含完整的 Token 与计费相关字段：

```typescript
{
  ["event.name"]: "inferenceWorker.run";
  "inference.model"?: string;               // 使用的模型名
  "inference.total_tokens"?: number;          // 总 Token 数
  "inference.prompt.custom_count"?: number;  // 自定义 Prompt 数量
  "inference.prompt.size"?: number;        // Prompt 字节数
  "inference.summary.size"?: number;         // 摘要结果字节数
  "inference.tagging.num_generated_tags"?: number;
  "inference.tagging.num_potential_relevant_tags"?: number;
}
```

`embeddingsWorker.run` 事件：

```typescript
{
  ["event.name"]: "embeddingsWorker.run";
  "embedding.prompt_tokens"?: number;
  "embedding.total_tokens"?: number;
  "embedding.text_size"?: number;
}
```

### 5.3 日志注入方式

通过 `addLogFields<T>()` 渐进式填充，在执行过程中任何位置都可以追加字段：

**示例（tagging.ts L399-L402

```typescript
addLogFields<"inferenceWorker.run">({
  "inference.tagging.num_generated_tags": tags.length,
  "inference.total_tokens": response.totalTokens,
});
```

最后 `withEventLog` 在函数结束时统一输出到 OTLP / Console。

### 5.4 端到端链路追踪

**文件位置**: [tracing.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared-server/src/tracing.ts)

OpenTelemetry Tracing + Event Logging 结合，实现：
- `workerTracing wrapper 自动创建 Span
- `setSpanAttributes` 设置属性
- 错误自动 `recordException`
- 与日志与 Trace 关联

---

## 6. Worker 分层架构

### 6.1 整体分层图

```
┌─────────────────────────────────────────────────────────────────┐
│                      tRPC / Web API 层                           │
│  (用户创建 bookmark → 提交 Crawler Job)                            │
└────────────────────┬────────────────────────────────────────────┘
                     │ enqueue
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                   LinkCrawlerQueue / LowPriorityCrawlerQueue    │
│                   (numRetries=5, 抓取网页内容)                     │
└────────────────────┬────────────────────────────────────────────┘
                     │ 抓取完成后触发
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                   EmbeddingsQueue                        │
│  ┌───────────────────────────────────────────────────┐     │
│  │  type: "embed" (入口点)                     │     │
│  │  ├─ 生成 embedding 向量                         │     │
│  │  ├─ EmbeddingsQueue.enqueue(type: "index")    │     │
│  │  └─ OpenAIQueue.enqueue(type: "tag" + embedding)│  │
│  │                                              │     │
│  │  type: "index" (向量入库，重试隔离)               │     │
│  │  └─ vectorStoreClient.addVectors()           │     │
│  │                                              │     │
│  │  type: "delete"                              │     │
│  │  └─ vectorStoreClient.deleteVectors()           │     │
│  └───────────────────────────────────────────────────┘     │
└────────────────────┬────────────────────────────────────┘
                     │ 或降级 (embed 成功 / fallback
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                      OpenAIQueue                          │
│  ┌───────────────────────────────────────────────────┐     │
│  │  type: "tag"                                   │     │
│  │  ├─ runTagging()                            │     │
│  │  │   ├─ inferFromText / inferFromImage       │     │
│  │  │   ├─ parseJsonFromLLMResponse()        │     │
│  │  │   └─ connectTags() (DB 事务)              │     │
│  │  │       ├─ 匹配现有标签                      │     │
│  │  │       ├─ 创建新标签                         │     │
│  │  │       ├─ 删除旧 AI 标签                    │     │
│  │  │       └─ 关联新标签                         │     │
│  │  │                                             │     │
│  │  └─ triggerSearchReindex()                    │     │
│  │                                               │     │
│  │  type: "summarize"                            │     │
│  │  └─ runSummarization()                    │     │
│  │      └─ inferFromText() → 写入 summary 字段     │     │
│  └───────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Inference Worker 调度层

**文件位置**: [inferenceWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/inference/inferenceWorker.ts#L44-L124)

`OpenAiWorker.build()` 负责：
1. 构建推理客户端（`InferenceClientFactory.build()`）
2. 校验 Job 数据（`zOpenAIRequestSchema`）
3. 根据 `type` 分发到 `runTagging` 或 `runSummarization`
4. 包装 tracing + event log middleware

```typescript
run: withWorkerTracing(
  "inferenceWorker.run",
  withWorkerEventLog("inferenceWorker.run", runOpenAI),
),
```

### 6.3 Tagging 业务层

**文件位置**: [tagging.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/inference/tagging.ts#L619-L728)

`runTagging` 执行流程：

```
1. 全局开关检查 (enableAutoTagging)
2. 用户级开关检查 (autoTaggingEnabled)
3. 解析用户偏好:
   ├─ tagStyle: as-generated / curated
   ├─ inferredTagLang: 语言
   └─ curatedTagIds: 精选标签 ID 列表
4. 构建 Prompt 上下文:
   ├─ 有 curatedTagIds → 直接使用精选标签
   └─ 无 → getPotentiallyRelevantTags() 向量相似性推荐
      ├─ 有 embedding 参数 → search({vector}) 无需等待索引
      └─ 无 → findSimilar({id}) 需已索引
5. 根据内容类型分发:
   ├─ link/text → inferTagsFromText()
   ├─ asset:image → inferTagsFromImage()
   └─ asset:pdf → inferTagsFromPDF()
6. parseJsonFromLLMResponse() 解析 + Zod 校验
7. connectTags() 数据库事务（标签匹配/创建/解绑/重绑）
8. 触发 RuleEngine + Webhook + Search 重索引
```

关键设计：**向量相似性推荐标签** 通过 `getPotentiallyRelevantTags` 实现 few-shot 上下文增强，使用 Meilisearch 向量搜索找到最多 10 个相似 bookmark，提取它们的标签作为 few-shot 参考。

### 6.4 Summarization 业务层

**文件位置**: [summarize.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/inference/summarize.ts#L47-L192)

与 Tagging 类似，但输出自由文本（`schema: null`），结果直接写入 `bookmarks.summary` 字段。

### 6.5 Embeddings Worker 分层（重试隔离设计）

**文件位置**: [embeddingsWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/embeddingsWorker.ts#L394-L498)

这是整个 AI 流水线中最精巧的设计：**将向量生成与向量索引解耦**。

```
type: "embed" Job:
  ├─ 生成 embedding 向量 (调用 LLM)
  ├─ 成功 →
  │   ├─ EmbeddingsQueue.enqueue(type: "index")  // 独立重试
  │   └─ OpenAIQueue.enqueue(type: "tag" + embedding)  // 立即携带 embedding
  └─ 失败 →
      └─ enqueueTaggingFallback() // 兜底，依然打标签

type: "index" Job:
  └─ 单独负责 vectorStoreClient.addVectors()
  └─ 即使 Meilisearch 很慢 / 挂掉，也不会影响 tagging
```

> 这样设计的好处：向量索引（通常依赖外部 Meilisearch，可能很慢）即使失败重试，也**绝不会重复触发 tagging**，避免重复消费 Token。

### 6.6 插件化的 Queue Provider

**文件位置**: [plugins.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/plugins.ts)

Queue 本身也是通过 PluginManager 插件化的：

```typescript
// loadAllPlugins() 加载顺序（后者优先）
await import("@karakeep/plugins/queue-liteque");   // 内存队列（默认）
await import("@karakeep/plugins/queue-restate");   // Restate 分布式队列
```

`PluginManager.getClient(PluginType.Queue)` 返回最后注册的 provider，实现队列后端可插拔。

---

## 7. 关键配置项总览

| 环境变量 | 默认值 | 作用 |
|---|---|---|
| `OPENAI_API_KEY` | - | OpenAI 兼容 API Key（存在则优先使用 OpenAI Provider） |
| `OPENAI_BASE_URL` | - | 兼容 API Base URL（可切换 Groq/Together 等） |
| `OPENAI_PROXY_URL` | - | HTTP 代理 URL |
| `OPENAI_TIMEOUT_SEC` | - | OpenAI SDK 级超时（秒） |
| `OPENAI_SERVICE_TIER` | - | `auto`/`default`/`flex` |
| `OPENAI_REASONING_EFFORT` | - | `none`~`xhigh` 推理深度 |
| `OLLAMA_BASE_URL` | - | Ollama 本地服务 URL（存在则次优先） |
| `OLLAMA_KEEP_ALIVE` | - | 模型内存保活时间 |
| `INFERENCE_JOB_TIMEOUT_SEC` | 30 | Worker 级 Job 超时（秒） |
| `INFERENCE_FETCH_TIMEOUT_SEC` | 300 | Ollama Fetch 级超时（秒） |
| `INFERENCE_TEXT_MODEL` | `gpt-4.1-mini` | 文本推理模型 |
| `INFERENCE_IMAGE_MODEL` | `gpt-4o-mini` | 图像推理模型 |
| `INFERENCE_CONTEXT_LENGTH` | 2048 | Prompt 上下文长度 |
| `INFERENCE_MAX_OUTPUT_TOKENS` | 2048 | 最大输出 Token |
| `INFERENCE_OUTPUT_SCHEMA` | `structured` | 输出模式 |
| `INFERENCE_NUM_WORKERS` | 1 | 推理并发 Worker 数 |
| `EMBEDDING_TEXT_MODEL` | `text-embedding-3-small` | 嵌入模型 |
| `EMBEDDING_DIMENSIONS` | 1536 | 嵌入维度 |
| `EMBEDDING_JOB_TIMEOUT_SEC` | 60 | Embedding Job 超时 |
| `EMBEDDING_NUM_WORKERS` | 1 | Embedding Worker 数 |

---

## 8. 总结：核心设计思想

### 8.1 关注点分离

| 层次 | 职责 | 关键文件 |
|---|---|---|
| **接口抽象层** | `InferenceClient` 统一三个能力 | [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts) |
| **Provider 实现层** | OpenAI / Ollama 差异封装 | [inference.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/inference.ts) |
| **配置驱动层** | 环境变量 → 强类型配置 | [config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared/config.ts) |
| **队列调度层** | 重试 / 超时 / 并发控制 | [queues.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared-server/src/queues.ts) |
| **Worker 执行层** | 业务逻辑（打标/摘要/嵌入） | [tagging.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/inference/tagging.ts) / [summarize.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/inference/summarize.ts) / [embeddingsWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/apps/workers/workers/embeddingsWorker.ts) |
| **可观测性层** | Token 统计 / 链路追踪 / 事件日志 | [eventLogger.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared-server/src/eventLogger.ts) + [tracing.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/41-karakeep/packages/shared-server/src/tracing.ts) |

### 8.2 弹性设计亮点

1. **四层超时** 确保无死角的超时保护（Job → AbortSignal → SDK → Fetch，每层独立可控
2. **Embedding-Tagging 解耦** 重试互不干扰，Embedding 失败不阻断 Tagging
3. **Index/Embed 职责拆分** 向量入库慢操作与 Tagging 生成操作的重试域隔离
4. **相似性上下文增强** 通过向量搜索实现动态 few-shot，提升打标签质量
5. **JSON 解析多层兜底** 对 LLM 不遵守 Schema 的场景做了充分容错
6. **插件化 Provider** Queue/RateLimit/Search/VectorStore 全部可热插拔切换后端
