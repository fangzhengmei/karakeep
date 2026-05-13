# Karakeep 书签推理系统技术分析报告

**报告日期**: 2026-05-13
**分析范围**: 推理工作器 (Inference Worker) - 摘要生成与自动打标签

---

## 目录
1. [整体架构概述](#整体架构概述)
2. [任务调度与队列管理](#任务调度与队列管理)
3. [内容提取流程](#内容提取流程)
4. [摘要生成机制](#摘要生成机制)
5. [自动打标签系统](#自动打标签系统)
6. [内容截断策略](#内容截断策略)
7. [标签去重与规范化](#标签去重与规范化)
8. [用户偏好配置](#用户偏好配置)
9. [数据库写入路径](#数据库写入路径)
10. [失败回退与错误处理](#失败回退与错误处理)
11. [关键配置项](#关键配置项)

---

## 整体架构概述

Karakeep 的推理系统采用**队列驱动的异步工作器架构**，主要负责：
- 书签内容的自动摘要生成
- 智能标签推荐
- 支持多种内容类型（链接、文本、图片、PDF）

### 核心组件
| 组件 | 文件位置 | 职责 |
|------|----------|------|
| 推理工作器 | `apps/workers/workers/inference/inferenceWorker.ts` | 协调 AI 推理任务 |
| 摘要生成器 | `apps/workers/workers/inference/summarize.ts` | 处理书签摘要生成逻辑 |
| 标签生成器 | `apps/workers/workers/inference/tagging.ts` | 处理自动打标签逻辑 |
| 队列系统 | `packages/shared-server/src/queues.ts` | 任务调度与重试管理 |
| AI 客户端 | `packages/shared/inference.ts` | OpenAI/Ollama 模型调用 |
| 提示词模板 | `packages/shared/prompts.ts` | 构建推理提示词 |
| 内容提取 | `packages/trpc/models/bookmarks.ts` | 书签内容读取与 HTML 转文本 |

---

## 任务调度与队列管理

### 队列配置
推理任务通过 `OpenAIQueue` 进行调度，配置如下：

```typescript
// packages/shared-server/src/queues.ts:126-131
export const OpenAIQueue = createDeferredQueue<ZOpenAIRequest>("openai_queue", {
  defaultJobArgs: {
    numRetries: 3,  // 默认重试3次
  },
  keepFailedJobs: false,
});
```

### 任务负载类型
```typescript
export const zOpenAIRequestSchema = z.object({
  bookmarkId: z.string(),
  type: z.enum(["summarize", "tag"]).default("tag"),
});
```

### Worker 初始化流程
```typescript
// apps/workers/workers/inference/inferenceWorker.ts:44-81
export class OpenAiWorker {
  static async build() {
    const worker = (await getQueueClient())!.createRunner<ZOpenAIRequest>(
      OpenAIQueue,
      {
        run: withWorkerTracing("inferenceWorker.run", ...),  // 链路追踪
        onComplete: async (job) => { ... },                   // 成功回调
        onError: async (job) => { ... },                      // 错误处理
      },
      {
        concurrency: serverConfig.inference.numWorkers,  // 并发数
        pollIntervalMs: 1000,                            // 轮询间隔
        timeoutSecs: serverConfig.inference.jobTimeoutSec,  // 超时时间
      },
    );
  }
}
```

### 任务路由
```typescript
// apps/workers/workers/inference/inferenceWorker.ts:114-123
switch (request.data.type) {
  case "summarize":
    await runSummarization(bookmarkId, job, inferenceClient);
    break;
  case "tag":
    await runTagging(bookmarkId, job, inferenceClient);
    break;
  default:
    throw new Error(`Unknown inference type: ${request.data.type}`);
}
```

---

## 内容提取流程

### 双源内容读取策略
书签内容支持两种存储方式，按优先级读取：

```typescript
// packages/trpc/models/bookmarks.ts:862-883
static async getBookmarkHtmlContent(
  { contentAssetId, htmlContent }: { contentAssetId: string | null; htmlContent: string | null },
  userId: string,
): Promise<string | null> {
  if (contentAssetId) {
    // 1. 从资产存储读取大文件（适用于大 HTML 内容）
    const asset = await readAsset({ userId, assetId: contentAssetId });
    return asset.asset.toString("utf8");
  } else if (htmlContent) {
    // 2. 直接使用数据库存储的小内容（适用于小 HTML 片段）
    return htmlContent;
  }
  return null;
}
```

### HTML 转纯文本
```typescript
// packages/trpc/models/bookmarks.ts:885-898
static async getBookmarkPlainTextContent(...): Promise<string | null> {
  const content = await this.getBookmarkHtmlContent(...);
  return content ? htmlToPlainText(content) : null;
}
```

**转换效果**：
- 移除 HTML 标签、样式、脚本
- 保留文本结构
- 移除冗余空白字符

### 摘要输入的元数据封装
```typescript
// apps/workers/workers/inference/summarize.ts:113-120
textToSummarize = `
Title: ${link.title ?? ""}
Description: ${link.description ?? ""}
Content: ${content}
Publisher: ${link.publisher ?? ""}
Author: ${link.author ?? ""}
URL: ${link.url ?? ""}
`;
```

---

## 摘要生成机制

### 触发条件（三级开关）
摘要生成仅在**全部**条件满足时执行：
1. ✅ 服务器全局配置 `enableAutoSummarization` 为 `true`
2. ✅ 用户级设置 `autoSummarizationEnabled` 不为 `false`
3. ✅ 书签存在有效内容（描述或 HTML 内容）

### 提示词模板
```typescript
// packages/shared/prompts.ts:71-82
export function constructSummaryPrompt(lang: string, customPrompts: string[], content: string): string {
  return `
Summarize the following content responding ONLY with the summary. You MUST follow the following rules:
- Summary must be in 3-4 sentences.
- The summary must be in ${lang}.
${customPrompts && customPrompts.map((p) => `- ${p}`).join("\n")}
    ${content}`;
}
```

### 自定义提示词
用户可以配置应用于摘要的自定义提示词，这些提示词会被追加到规则列表中。

### 完整执行流程
```typescript
// apps/workers/workers/inference/summarize.ts:47-191
async function runSummarization(bookmarkId, job, inferenceClient) {
  // 1. 前置检查：全局开关、用户开关
  // 2. 读取书签详情（链接、标题、描述、内容资产）
  // 3. 提取纯文本内容（HTML -> Plain Text）
  // 4. 无内容则跳过，不标记失败
  // 5. 封装元数据（标题、描述、内容、发布者、作者、URL）
  // 6. 加载用户自定义提示词
  // 7. 构建提示词（应用截断策略）
  // 8. 调用 AI 模型（OpenAI 或 Ollama）
  // 9. 空响应则抛出错误
  // 10. 将摘要写入 bookmarks 表
  // 11. 触发搜索索引更新
}
```

---

## 自动打标签系统

### 支持的内容类型
| 书签类型 | 处理方式 | 模型 |
|---------|----------|------|
| 链接/网页 | 提取 HTML 纯文本内容 | 文本模型 |
| 纯文本笔记 | 直接使用文本内容 | 文本模型 |
| 图片资产 | Base64 编码 + Vision API | 图像模型 |
| PDF 资产 | 读取 PDF 文本内容 | 文本模型 |

### 标签生成提示词（文本）
```typescript
// packages/shared/prompts.ts:37-66
export function constructTextTaggingPrompt(
  lang: string,
  customPrompts: string[],
  content: string,
  tagStyle: ZTagStyle,
  curatedTags?: string[],
): string {
  return `
You are an expert whose responsibility is to help with automatic tagging for a read-it-later/bookmarking app.
Analyze the TEXT_CONTENT below and suggest relevant tags that describe its key themes, topics, and main ideas. The rules are:
- Aim for a variety of tags, including broad categories, specific keywords, and potential sub-genres.
- The tags must be in ${lang}.
- If the tag is not generic enough, don't include it.
- Do NOT generate tags related to:
    - An error page (404, 403, blocked, not found, dns errors)
    - Boilerplate content (cookie consent, login walls, GDPR notices)
- Aim for 3-5 tags.
- If there are no good tags, leave the array empty.
${curatedInstruction}
${tagStyleInstruction}
${customPrompts && customPrompts.map((p) => `- ${p}`).join("\n")}

<TEXT_CONTENT>
${content}
</TEXT_CONTENT>
You must respond in JSON with the key "tags" and the value is an array of string tags.`;
}
```

### 图片打标签提示词
```typescript
// packages/shared/prompts.ts:11-32
export function buildImagePrompt(lang, customPrompts, tagStyle, curatedTags) {
  return `
You are an expert whose responsibility is to help with automatic text tagging for a read-it-later/bookmarking app.
Analyze the attached image and suggest relevant tags that describe its key themes, topics, and main ideas. The rules are:
- Aim for a variety of tags, including broad categories, specific keywords, and potential sub-genres.
- The tags must be in ${lang}.
- If the tag is not generic enough, don't include it.
- Aim for 10-15 tags.
- If there are no good tags, don't emit any.
${curatedInstruction}
${tagStyleInstruction}
${customPrompts && customPrompts.map((p) => `- ${p}`).join("\n")}
You must respond in valid JSON with the key "tags" and the value is list of tags. Don't wrap the response in a markdown code.`;
}
```

### 自定义提示词变量替换
```typescript
// apps/workers/workers/inference/tagging.ts:202-224
async function replaceTagsPlaceholders(prompts: string[], userId: string): Promise<string[]> {
  const api = await buildImpersonatingTRPCClient(userId);
  const tags = (await api.tags.list({})).tags;
  const tagsString = `[${tags.map((tag) => tag.name).join(", ")}]`;
  const aiTagsString = `[${tags.filter(...).map((tag) => tag.name).join(", ")}]`;
  const userTagsString = `[${tags.filter(...).map((tag) => tag.name).join(", ")}]`;

  return prompts.map((p) =>
    p
      .replaceAll("$tags", tagsString)
      .replaceAll("$aiTags", aiTagsString)
      .replaceAll("$userTags", userTagsString),
  );
}
```

| 变量 | 替换内容 |
|------|----------|
| `$tags` | 用户所有标签 |
| `$aiTags` | 仅 AI 生成的标签 |
| `$userTags` | 仅用户手动添加的标签 |

---

## 内容截断策略

### Token 编码方式
使用 OpenAI 的 `o200k_base` 编码（通过 `js-tiktoken` 库实现）：

```typescript
// packages/shared/prompts.server.ts:12-19
async function getEncodingInstance(): Promise<Tiktoken> {
  if (!encoding) {
    const { getEncoding } = await import("js-tiktoken");
    encoding = getEncoding("o200k_base");
  }
  return encoding;
}
```

### 截断算法
```typescript
// packages/shared/prompts.server.ts:26-37
async function truncateContent(content: string, length: number): Promise<string> {
  const enc = await getEncodingInstance();
  const tokens = enc.encode(content);
  if (tokens.length <= length) {
    return content;  // 无需截断
  }
  const truncatedTokens = tokens.slice(0, length);  // 只保留前 N 个 token
  return enc.decode(truncatedTokens);
}
```

### 动态上下文分配
**核心思想**：优先保证提示词模板完整性，剩余 token 分配给内容。

```typescript
// packages/shared/prompts.server.ts:46-73
export async function buildTextPrompt(
  lang: string,
  customPrompts: string[],
  content: string,
  contextLength: number,
  tagStyle: ZTagStyle,
  curatedTags?: string[],
): Promise<string> {
  content = preprocessContent(content);  // 前置处理
  
  // 1. 构建空内容的提示词模板
  const promptTemplate = constructTextTaggingPrompt(lang, customPrompts, "", tagStyle, curatedTags);
  
  // 2. 计算模板占用的 token 数
  const promptSize = await calculateNumTokens(promptTemplate);
  
  // 3. 计算剩余可用 token
  const available = Math.max(0, contextLength - promptSize);
  
  // 4. 将内容截断到可用 token 范围内
  const truncatedContent = available === 0 ? "" : await truncateContent(content, available);
  
  // 5. 重新构建完整提示词
  return constructTextTaggingPrompt(lang, customPrompts, truncatedContent, tagStyle, curatedTags);
}
```

### 前置内容清理
在 token 计算前，先清理冗余空白：
```typescript
// packages/shared/prompts.ts:7-9
function preprocessContent(content: string) {
  return content.replace(/(\s){10,}/g, "$1");  // 连续10+空白 -> 单个空白
}
```

---

## 标签去重与规范化

### 标签归一化函数
```typescript
// apps/workers/workers/inference/tagging.ts:75-84
function tagNormalizer() {
  function normalizeTag(tag: string) {
    // 转小写 + 移除空格/连字符/下划线
    return tag.toLowerCase().replace(/[ \-_]/g, "");
  }
  return { normalizeTag };
}
```

**⚠️ 同步约束**：此函数必须与数据库虚拟列定义**完全一致**：
```sql
-- packages/db/schema.ts:426-433
normalizedName GENERATED ALWAYS AS (
  lower(replace(replace(replace(name, ' ', ''), '-', ''), '_', ''))
) VIRTUAL
```

### 标签清洗（AI 返回后）
```typescript
// apps/workers/workers/inference/tagging.ts:371-377
tags = tags.map((t) => {
  let tag = t;
  if (tag.startsWith("#")) {
    tag = t.slice(1);  // 移除 # 前缀
  }
  return tag.trim();   // 移除首尾空白
});
```

### 去重与标签关联流程
```typescript
// apps/workers/workers/inference/tagging.ts:392-480
async function connectTags(bookmarkId: string, inferredTags: string[], userId: string) {
  await db.transaction(async (tx) => {
    // ─── 步骤 1: 归一化 + 匹配现有标签 ───
    const normalizedInferredTags = inferredTags.map(t => ({
      originalTag: t,
      normalizedTag: normalizeTag(t)
    }));

    const matchedTags = await tx.query.bookmarkTags.findMany({
      where: and(
        eq(bookmarkTags.userId, userId),
        inArray(bookmarkTags.normalizedName, normalizedInferredTags.map(t => t.normalizedTag)),
      ),
    });

    const matchedTagIds = matchedTags.map(r => r.id);
    const notFoundTagNames = normalizedInferredTags
      .filter(t => !matchedTags.some(mt => normalizeTag(mt.name) === t.normalizedTag))
      .map(t => t.originalTag);

    // ─── 步骤 2: 创建不存在的新标签 ───
    let newTagIds: string[] = [];
    if (notFoundTagNames.length > 0) {
      newTagIds = (
        await tx
          .insert(bookmarkTags)
          .values(notFoundTagNames.map(t => ({ name: t, userId })))
          .onConflictDoNothing()  // 幂等：冲突则忽略
          .returning()
      ).map(t => t.id);
    }

    // ─── 步骤 3: 删除旧的 AI 标签 ───
    // 先清除此书签之前所有 AI 生成的标签，避免累积
    await tx
      .delete(tagsOnBookmarks)
      .where(and(
        eq(tagsOnBookmarks.attachedBy, "ai"),
        eq(tagsOnBookmarks.bookmarkId, bookmarkId),
      ));

    // ─── 步骤 4: 关联新标签 ───
    const allTagIds = new Set([...matchedTagIds, ...newTagIds]);
    if (allTagIds.size > 0) {
      await tx
        .insert(tagsOnBookmarks)
        .values([...allTagIds].map(tagId => ({
          tagId,
          bookmarkId,
          attachedBy: "ai" as const,  // 标记为 AI 生成
        })))
        .onConflictDoNothing();
    }
  });
}
```

**去重效果**：
- `"Machine Learning"`、`"machine-learning"`、`"machine_learning"` 被视为同一标签
- 避免因大小写、分隔符不同导致的重复标签

---

## 用户偏好配置

### 用户级 AI 设置（数据库 Schema）
```typescript
// packages/db/schema.ts:84-100
export const users = sqliteTable("users", {
  // ... 其他字段
  
  // AI Settings (nullable = opt-in, null means use server default)
  autoTaggingEnabled: integer("autoTaggingEnabled", { mode: "boolean" }),
  autoSummarizationEnabled: integer("autoSummarizationEnabled", { mode: "boolean" }),
  tagStyle: text("tagStyle", {
    enum: [
      "lowercase-hyphens",
      "lowercase-spaces",
      "lowercase-underscores",
      "titlecase-spaces",
      "titlecase-hyphens",
      "camelCase",
      "as-generated",
    ],
  }).default("titlecase-spaces"),
  curatedTagIds: text("curatedTagIds", { mode: "json" }).$type<string[]>(),
  inferredTagLang: text("inferredTagLang"),
});
```

### 标签风格映射
| 风格值 | 说明 | 示例 |
|--------|------|------|
| `lowercase-hyphens` | 小写 + 连字符 | `machine-learning` |
| `lowercase-spaces` | 小写 + 空格 | `machine learning` |
| `lowercase-underscores` | 小写 + 下划线 | `machine_learning` |
| `titlecase-spaces` | 首字母大写 + 空格 | `Machine Learning` |
| `titlecase-hyphens` | 首字母大写 + 连字符 | `Machine-Learning` |
| `camelCase` | 驼峰命名 | `machineLearning` |
| `as-generated` | 保持模型原样 | 由模型输出决定 |

### 精选标签（Curated Tags）
如果用户配置了 `curatedTagIds`，AI 会优先从这些标签中选择：

```typescript
// packages/shared/utils/tag.ts
export function getCuratedTagsPrompt(curatedTags?: string[]): string {
  if (!curatedTags || curatedTags.length === 0) return "";
  return `- Prefer using tags from the following list: ["${curatedTags.join('", "')}"]`;
}
```

**工作流**：
1. 从用户设置读取 `curatedTagIds`（ID 数组）
2. 查库解析为标签名称
3. 注入提示词，引导 AI 优先推荐

---

## 数据库写入路径

### 摘要写入
```typescript
// apps/workers/workers/inference/summarize.ts:180-191
await db
  .update(bookmarks)
  .set({
    summary: summaryResult.response,  // 摘要内容直接写入
    modifiedAt: new Date(),
  })
  .where(eq(bookmarks.id, bookmarkId));

await triggerSearchReindex(bookmarkId, { priority: job.priority, groupId: bookmarkData.userId });
```

### 标签写入（事务性）
**涉及 3 张表的原子操作**：

| 表 | 操作 | 关键字段 |
|----|------|---------|
| `bookmarkTags` | INSERT | `name`, `userId` |
| `tagsOnBookmarks` | DELETE + INSERT | `bookmarkId`, `tagId`, `attachedBy` |
| `bookmarks` | UPDATE | `taggingStatus` |

### 状态字段
书签表包含两个状态字段用于追踪推理进度：

```typescript
// bookmarks 表字段
summarizationStatus: "success" | "failure" | null;
taggingStatus: "success" | "failure" | null;
```

### 状态更新逻辑
```typescript
// apps/workers/workers/inference/inferenceWorker.ts:21-42
async function attemptMarkStatus(jobData: object | undefined, status: "success" | "failure") {
  if (!jobData) return;
  try {
    const request = zOpenAIRequestSchema.parse(jobData);
    await db
      .update(bookmarks)
      .set({
        ...(request.type === "summarize" ? { summarizationStatus: status } : {}),
        ...(request.type === "tag" ? { taggingStatus: status } : {}),
      })
      .where(eq(bookmarks.id, request.bookmarkId));
  } catch (e) {
    logger.error(`Something went wrong when marking the tagging status: ${e}`);
  }
}
```

---

## 失败回退与错误处理

### 重试机制配置
| 配置项 | 值 | 说明 |
|--------|-----|------|
| `numRetries` | 3 | 最多重试 3 次（共 4 次尝试） |
| `timeoutSecs` | 配置项 | 单任务超时时间 |
| `keepFailedJobs` | false | 失败任务不保留在队列 |

### 错误回调流程
```typescript
// apps/workers/workers/inference/inferenceWorker.ts:60-70
onError: async (job) => {
  workerStatsCounter.labels("inference", "failed").inc();  // 统计失败
  
  const jobId = job.id;
  logger.error(`[inference][${jobId}] inference job failed: ${job.error}\n${job.error.stack}`);
  
  if (job.numRetriesLeft == 0) {
    // 重试耗尽，标记为永久失败
    workerStatsCounter.labels("inference", "failed_permanent").inc();
    await attemptMarkStatus(job?.data, "failure");
  }
}
```

### 成功回调
```typescript
// apps/workers/workers/inference/inferenceWorker.ts:54-59
onComplete: async (job) => {
  workerStatsCounter.labels("inference", "completed").inc();
  const jobId = job.id;
  logger.info(`[inference][${jobId}] Completed successfully`);
  await attemptMarkStatus(job.data, "success");
}
```

### LLM 响应解析容错
```typescript
// apps/workers/workers/inference/tagging.ts:39-73
function parseJsonFromLLMResponse(response: string): unknown {
  const trimmedResponse = response.trim();

  // 策略 1: 尝试直接解析 JSON
  try { return JSON.parse(trimmedResponse); }
  catch { /* 继续 */ }

  // 策略 2: 从 Markdown 代码块提取 (```json ... ```)
  const jsonBlockRegex = /```(?:json)?\s*(\{[\s\S]*?\})\s*```/i;
  const match = trimmedResponse.match(jsonBlockRegex);
  if (match) {
    try { return JSON.parse(match[1]); }
    catch { /* 继续 */ }
  }

  // 策略 3: 查找文本中的 JSON 对象边界
  const jsonObjectRegex = /\{[\s\S]*\}/;
  const objectMatch = trimmedResponse.match(jsonObjectRegex);
  if (objectMatch) {
    try { return JSON.parse(objectMatch[0]); }
    catch { /* 继续 */ }
  }

  // 策略 4: 最后尝试 - 抛出原始错误
  return JSON.parse(trimmedResponse);
}
```

### 内容缺失的优雅降级
**行为**：如果书签没有可用于推理的内容（无描述、无 HTML 内容）：
- ❌ 不标记任务为失败
- ✅ 直接跳过推理
- ✅ 记录 INFO 级别日志
- ✅ 不消耗重试次数

```typescript
// apps/workers/workers/inference/summarize.ts:105-111
if (!link.description && !content) {
  logger.info(`[inference] No content found for link "${bookmarkId}". Skipping summary.`);
  return;  // 静默跳过，不报错
}
```

### 模型客户端选项
支持两种推理后端，带有不同容错策略：

```typescript
// packages/shared/inference.ts
export class OpenAIInferenceClient implements InferenceClient {
  // 使用官方 SDK，结构化输出支持 Zod schema
  // 支持 Proxy 配置
  // 支持 reasoning_effort 等高级参数
}

class OllamaInferenceClient implements InferenceClient {
  // 流式响应积累（避免部分成功但整体报错的情况）
  // AbortSignal 支持取消
  // 流中断时返回已积累的部分响应
}
```

---

## 关键配置项

```typescript
// serverConfig.inference 配置结构
{
  // 工作器配置
  numWorkers: number;           // 并发 worker 数量
  jobTimeoutSec: number;        // 单任务超时时间（秒）
  
  // 功能开关
  enableAutoTagging: boolean;   // 全局自动打标签开关
  enableAutoSummarization: boolean;  // 全局自动摘要开关
  
  // 模型配置
  textModel: string;            // 文本推理模型（GPT-4、Claude 等）
  imageModel: string;           // 图像推理模型（GPT-4V、Claude 3 Opus 等）
  contextLength: number;        // 上下文窗口大小（token 数）
  maxOutputTokens: number;      // 最大输出 token 数
  useMaxCompletionTokens: boolean;  // 使用 max_completion_tokens 参数
  
  // OpenAI 专属
  openAIApiKey?: string;
  openAIBaseUrl?: string;
  openAIProxyUrl?: string;
  openAIServiceTier?: string;
  openAIReasoningEffort?: "none" | "minimal" | "low" | "medium" | "high" | "xhigh";
  
  // Ollama 专属
  ollamaBaseUrl?: string;
  ollamaKeepAlive?: string;
  
  // 输出配置
  outputSchema: "structured" | "json" | "plain";
  inferredTagLang: string;      // 默认标签语言
}
```

---

## 总结

### 架构设计亮点
1. **分层解耦**：队列、内容提取、AI 调用、结果存储各司其职
2. **三级开关**：全局 → 用户级 → 内容存在性，逐级检查
3. **事务原子性**：标签操作通过数据库事务保证一致性
4. **幂等设计**：`onConflictDoNothing` 避免重复创建

### 容错机制
- ✅ 多级重试（最多 3 次）
- ✅ 超时保护
- ✅ LLM 响应多级解析容错
- ✅ 内容缺失的优雅降级
- ✅ 状态标记可追溯

### 用户体验优化
- 🎯 自定义提示词支持变量注入
- 🎯 7 种标签风格灵活切换
- 🎯 精选标签引导 AI 推荐偏好
- 🎯 多语言支持

### 性能优化
- ⚡ Token 精确计算与动态截断
- ⚡ 内容前置清理减少 token 浪费
- ⚡ 资产分离存储避免大字段性能问题
- ⚡ 并发控制保障服务稳定性

---

**文档维护**：本报告对应代码版本为 Karakeep v1.x，如有架构变更请同步更新。
