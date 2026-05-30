# 附件类资源处理流程梳理

## 概述

本文档梳理 Karakeep 项目中附件类资源从上传暂存到信息提取再回写到主记录的完整处理流程，重点回答：

1. 哪些上传类型能进入附件预处理队列
2. 哪些类型会在创建资产书签时被拒绝
3. 图片与 PDF 的后台处理分支分别执行了什么任务
4. "提取未落库但任务成功完成"的分支在什么情况下仍会入队 AI 任务
5. tagging 和 summarization 的状态如何变化，特别是 `taggingStatus = "success"` 的统一判定标准
6. 这些条件为何会影响状态卡顿的排查

---

## 一、两层类型约束：上传入口 vs 书签入口

代码中存在两个不同的类型白名单，分别约束「上传文件」和「创建资产书签」，这是理解流程的关键。

### 1.1 SUPPORTED_UPLOAD_ASSET_TYPES（上传入口白名单）

**位置：** `packages/shared/assetdb.ts:51-56`

```
图片：image/gif, image/jpeg, image/png, image/webp
视频：video/mp4, video/webm, video/x-matroska
HTML：text/html
PDF：  application/pdf
```

这组类型控制的是 `uploadAsset()` 函数能接收什么文件。上传成功后，文件以 `assetType = UNKNOWN`、`bookmarkId = null` 的状态存入 `assets` 表，此时还没有关联任何书签。

### 1.2 SUPPORTED_BOOKMARK_ASSET_TYPES（书签入口白名单）

**位置：** `packages/shared/assetdb.ts:59-62`

```
图片：image/gif, image/jpeg, image/png, image/webp
PDF：  application/pdf
```

**仅包含图片和 PDF，不包含视频和 HTML。**

这组类型控制的是 `createBookmark(type=ASSET)` 时是否允许将已上传的资产绑定到书签。校验逻辑位于 `packages/trpc/routers/bookmarks.ts:329-338`：

```typescript
if (
  !uploadedAsset.asset.contentType ||
  !SUPPORTED_BOOKMARK_ASSET_TYPES.has(uploadedAsset.asset.contentType)
) {
  throw new TRPCError({
    code: "BAD_REQUEST",
    message: "Unsupported asset type",
  });
}
```

### 1.3 被拒绝的类型

| 上传类型 | 能上传? | 能创建资产书签? | 原因 |
|----------|---------|----------------|------|
| image/gif, jpeg, png, webp | ✅ | ✅ | — |
| application/pdf | ✅ | ✅ | — |
| video/mp4, webm, mkv | ✅ | ❌ | 不在 SUPPORTED_BOOKMARK_ASSET_TYPES 中 |
| text/html | ✅ | ❌ | 不在 SUPPORTED_BOOKMARK_ASSET_TYPES 中 |

视频和 HTML 文件虽然可以上传，但无法作为 `type=ASSET` 的书签使用。HTML 上传后仅用于链接类型的 `precrawledArchive`（预抓取存档），视频目前没有书签入口。

---

## 二、哪些类型能进入附件预处理队列

### 2.1 入队时机

只有 `createBookmark(type=ASSET)` 成功后才会入队 `AssetPreprocessingQueue`：

```typescript
// packages/trpc/routers/bookmarks.ts:419-428
case BookmarkTypes.ASSET: {
  await AssetPreprocessingQueue.enqueue(
    { bookmarkId: bookmark.id, fixMode: false },
    enqueueOpts,
  );
  break;
}
```

注意 `fixMode: false` 是正常路径，这对后续的 AI 入队条件非常重要。

因此**只有通过了 SUPPORTED_BOOKMARK_ASSET_TYPES 校验的类型才能进入预处理队列**，即**只有图片和 PDF**。

### 2.2 Worker 内的分支约束

`assetPreprocessingWorker.ts:420-452` 中 `run()` 函数的 switch 语句按 `bookmarkAssets.assetType` 分派：

```typescript
switch (bookmark.asset.assetType) {
  case "image": { ... }
  case "pdf":   { ... }
  default:
    throw new Error(`[assetPreprocessing][${jobId}] Unsupported bookmark type`);
}
```

`bookmarkAssets.assetType` 的枚举值在 schema 中定义为 `"image" | "pdf"`（`packages/db/schema.ts:410`），也在请求 schema `zNewBookmarkRequestSchema` 中限定为 `z.enum(["image", "pdf"])`（`packages/shared/types/bookmarks.ts:187`）。

因此 Worker 只处理这两种类型，其余走 `default` 分支直接抛异常。

---

## 三、图片与 PDF 的后台处理分支详解

### 3.1 图片分支 (`case "image"`)

**入口：** `assetPreprocessingWorker.ts:421-430`

调用 `extractAndSaveImageText()`，执行以下任务：

1. **fixMode 检查**：若已有 content 且为 fixMode，跳过
2. **OCR 文本提取**（根据配置二选一）：
   - `serverConfig.ocr.useLLM = true` → 调用 `readImageTextWithLLM()`
   - `serverConfig.ocr.useLLM = false` → 调用 `readImageText()`
3. **回写 bookmarkAssets.content**：提取成功时更新，提取失败（返回 null）时不更新

**图片分支只做 OCR 文本提取，不生成截图。** 图片本身就是视觉资产，无需额外截图。

### 3.2 PDF 分支 (`case "pdf"`)

**入口：** `assetPreprocessingWorker.ts:432-447`

PDF 分支串行执行**两个独立任务**：

#### 任务 1：文本提取 (`extractAndSavePDFText`)

1. **fixMode 检查**：若已有 content 且为 fixMode，跳过
2. **pdf2json 解析**：提取原始文本和 PDF 元数据
3. **空文本抛异常**：`if (!pdfParse?.text) throw new Error(...)` — PDF 文本为空时直接抛错，导致整个 Worker 任务失败并进入重试
4. **回写 bookmarkAssets**：
   - `content` ← 提取的文本
   - `metadata` ← PDF 元数据（JSON 序列化）

#### 任务 2：首页截图 (`extractAndSavePDFScreenshot`)

1. **fixMode 检查**：若已有 ASSET_SCREENSHOT 且为 fixMode，跳过
2. **pdf2pic 渲染**：将 PDF 第 1 页渲染为 PNG
3. **存储配额检查**：超限时跳过（不报错，返回 true）
4. **保存截图**：作为独立资产插入 `assets` 表（`assetType = ASSET_SCREENSHOT`），通过 `bookmarkId` 关联主记录

**注意：文本提取和截图生成是串行但独立的。文本提取失败（抛异常）会中断后续截图生成；截图生成失败不影响已完成的文本提取。**

---

## 四、AI 任务入队条件的深度解析

这是之前分析的关键错误点。现在详细分析 `assetPreprocessingWorker.ts:463` 的条件：

```typescript
if (!isFixMode || anythingChanged) {
  await OpenAIQueue.enqueue({ bookmarkId, type: "tag" }, enqueueOpts);
  await OpenAIQueue.enqueue({ bookmarkId, type: "summarize" }, enqueueOpts);
  await triggerSearchReindex(bookmarkId, enqueueOpts);
}
```

### 4.1 真值表分析

`isFixMode` 来自 `req.data.fixMode`，默认值 `false`（`packages/shared-server/src/queues.ts:238`）。

| 执行路径 | isFixMode | anythingChanged | `!isFixMode \|\| anythingChanged` | 是否入队 AI |
|----------|-----------|-----------------|-----------------------------------|------------|
| 正常创建书签 | `false` | `true`（提取成功） | `true` | ✅ 入队 |
| 正常创建书签 | `false` | `false`（提取失败/跳过） | **`true`** | **✅ 仍入队！** |
| 修复模式（admin） | `true` | `true`（有变更） | `true` | ✅ 入队 |
| 修复模式（admin） | `true` | `false`（无变更） | `false` | ❌ 不入队 |

**关键结论：正常路径下（`isFixMode = false`），无论 `anythingChanged` 是 true 还是 false，都会入队 AI 任务！**

这是因为 `!isFixMode = true`，逻辑或的结果恒为 `true`，`anythingChanged` 被短路忽略。

### 4.2 什么情况下会出现 "提取未落库但任务成功完成"

只有**图片 OCR 静默失败**会出现这种情况：

- OCR 置信度低于阈值
- OCR 配置为空语言列表
- LLM OCR 返回空文本

此时 `extractAndSaveImageText()` 返回 `false`：
- `anythingChanged = false`
- `bookmarkAssets.content` 保持 `null`
- 但**任务不会抛异常**，正常执行到入队判断
- 由于 `!isFixMode = true`，条件成立，入队 tag 和 summarize

**PDF 不会出现这种情况**，因为 PDF 文本提取失败会直接抛异常（`throw new Error(...)`），任务失败进入重试，不会执行到入队步骤。

---

## 五、`taggingStatus = "success"` 的统一判定标准

这是之前结论存在矛盾的核心。现在统一梳理所有能得到 `taggingStatus = "success"` 的执行路径。

### 5.1 状态更新的唯一入口

`taggingStatus` 的更新只有一个入口：`attemptMarkStatus()` 函数。

```typescript
// inferenceWorker.ts:21-41
async function attemptMarkStatus(
  jobData: object | undefined,
  status: "success" | "failure",
) {
  if (!jobData) return;
  try {
    const request = zOpenAIRequestSchema.parse(jobData);
    await db
      .update(bookmarks)
      .set({
        ...(request.type === "tag" ? { taggingStatus: status } : {}),
      })
      .where(eq(bookmarks.id, request.bookmarkId));
  } catch (e) {
    logger.error(`Something went wrong when marking the tagging status: ${e}`);
  }
}
```

`attemptMarkStatus()` 仅在两个地方被调用：
1. **`onComplete` 回调**（第 58 行）：`attemptMarkStatus(job.data, "success")`
2. **`onError` 回调**（第 68 行）：`attemptMarkStatus(job?.data, "failure")`（仅当 `job.numRetriesLeft == 0`）

**核心规则：只要 `runOpenAI()` 函数正常返回（不抛异常），`onComplete` 就会被调用，`taggingStatus` 就会被设为 `"success"`。**

### 5.2 `taggingStatus = "success"` 且**有标签产物**的路径

**唯一一条路径**：

```
入队 tag 任务
  ↓
inferenceClient 存在
  ↓
全局开关 enableAutoTagging = true
  ↓
用户开关 autoTaggingEnabled = true
  ↓
inferTags() 返回非 null 且非空数组
  ↓
connectTags() 执行标签匹配/创建/关联
  ↓
runOpenAI() 正常返回
  ↓
onComplete → taggingStatus = "success"
  ↓
✅ tagsOnBookmarks 表中有记录
```

**触发条件：**
- AI 配置完整且服务可用
- 用户开启了自动标签功能
- 内容充足（非 GIF、有 OCR 文本或可视觉分析）
- AI 返回了符合 schema 的非空标签数组

**可观察证据：**
- `tagsOnBookmarks` 表中存在 `attachedBy = "ai"` 的记录
- 日志中有 `Inferring tag for bookmark ... used X tokens and inferred: [...]`
- 日志中有 `triggerSearchReindex` 记录（`tagging.ts:596`）

---

### 5.3 `taggingStatus = "success"` 但**没有标签产物**的路径

共 **5 条路径**，全部满足「`runOpenAI()` 正常返回 → `onComplete` 标记 success」但「`connectTags()` 未执行或未写入任何标签」。

---

#### 路径 A：`inferenceClient` 未配置

**触发条件：**
```typescript
// inferenceWorker.ts:86-92
const inferenceClient = InferenceClientFactory.build();
if (!inferenceClient) {
  logger.debug(`[inference][${jobId}] No inference client configured, nothing to do now`);
  return;  // 直接返回，不抛异常
}
```
- `InferenceClientFactory.build()` 返回 `null`（未配置 API Key 或模型）
- `runOpenAI()` 在第 91 行直接 `return`
- **不会调用 `runTagging()`**
- 任务标记为成功

**可观察证据：**
- 日志（debug 级别）：`No inference client configured, nothing to do now`
- `tagsOnBookmarks` 表中没有新记录
- 没有 `Starting an inference job` 日志（该日志在 `runTagging()` 第 555 行）

---

#### 路径 B：全局自动标签开关关闭

**触发条件：**
```typescript
// tagging.ts:510-514
if (!serverConfig.inference.enableAutoTagging) {
  logger.debug(`[inference][${jobId}] Skipping tagging job ... because it's disabled in the config.`);
  return;
}
```
- `serverConfig.inference.enableAutoTagging = false`
- `runTagging()` 在第 514 行直接 `return`
- 不调用 `connectTags()`

**可观察证据：**
- 日志（debug 级别）：`Skipping tagging job ... because it's disabled in the config.`
- 有 `Starting an inference job` 日志吗？**没有**，因为第 555 行的日志在检查之后
- `tagsOnBookmarks` 表中没有新记录

---

#### 路径 C：用户自动标签开关关闭

**触发条件：**
```typescript
// tagging.ts:535-539
if (userSettings?.autoTaggingEnabled === false) {
  logger.debug(`[inference][${jobId}] Skipping tagging job ... because user has disabled auto-tagging.`);
  return;
}
```
- 用户在设置中关闭了自动标签
- `runTagging()` 在第 539 行直接 `return`
- 不调用 `connectTags()`

**可观察证据：**
- 日志（debug 级别）：`Skipping tagging job ... because user has disabled auto-tagging.`
- 有 `Starting an inference job` 日志吗？**有**，在第 555 行
- `tagsOnBookmarks` 表中没有新记录

---

#### 路径 D：`inferTags()` 返回 null（内容不足）

**触发条件：** `inferTags()` 返回 `null`，由以下子路径触发：

| 子路径 | 触发场景 | 代码位置 |
|--------|---------|---------|
| D-1 | GIF 图片 | `tagging.ts:152-157` |
| D-2 | Link 无标题且无内容 | `tagging.ts:99-105` |
| D-3 | `buildPrompt()` 返回 null | `tagging.ts:278-279` |

```typescript
// tagging.ts:569-574
if (tags === null) {
  logger.info(`[inference][${jobId}] Skipping tagging for bookmark "${bookmark.id}" due to missing content.`);
  return;
}
```
- `runTagging()` 在第 574 行直接 `return`
- 不调用 `connectTags()`

**可观察证据：**
- 日志（info 级别）：`Skipping tagging for bookmark ... due to missing content.`
- 有 `Starting an inference job` 日志
- `tagsOnBookmarks` 表中没有新记录
- 对于 GIF：日志中还有 `Skipping inference ... because it's a GIF.`（info 级别）

---

#### 路径 E：AI 返回空数组 `tags = []`

**触发条件：**
- `inferTags()` 返回非 null
- AI 返回的 JSON 符合 schema 但 `tags` 是空数组 `[]`
- `openAIResponseSchema = z.object({ tags: z.array(z.string()) })` 允许空数组

```typescript
// tagging.ts:397-398
async function connectTags(bookmarkId, inferredTags, userId) {
  if (inferredTags.length == 0) {
    return;  // 空数组直接返回，不执行任何操作
  }
  // ... 后续标签关联逻辑
}
```
- `connectTags()` 检测到空数组，直接 `return`
- 不执行标签匹配、创建、关联
- 但 `runTagging()` 正常完成，不抛异常
- 后续 webhook 和搜索索引仍会触发吗？**会**，在 `tagging.ts:585-596`

**可观察证据：**
- 日志中有 `Inferring tag for bookmark ... used X tokens and inferred: []`
- `tagsOnBookmarks` 表中没有新记录（但旧的 AI 标签也不会被删除，因为删除逻辑在 `connectTags()` 第 451-459 行）
- 有 `triggerSearchReindex` 日志
- 注意：这条路径的特殊之处在于**消耗了 token** 但没有产生任何标签

---

### 5.4 各路径对比汇总

| 路径 | taggingStatus | 有标签? | 消耗 token? | 日志级别 | 关键日志关键词 |
|------|--------------|---------|------------|---------|---------------|
| 正常成功 | success | ✅ | ✅ | info | `Inferring tag ... and inferred: [...]` |
| A. inferenceClient 未配置 | success | ❌ | ❌ | debug | `No inference client configured` |
| B. 全局开关关闭 | success | ❌ | ❌ | debug | `disabled in the config` |
| C. 用户开关关闭 | success | ❌ | ❌ | debug | `user has disabled auto-tagging` |
| D. 内容不足（GIF/空） | success | ❌ | ❌（D-1）/ ✅（D-2/3） | info | `due to missing content` |
| E. AI 返回空数组 | success | ❌ | ✅ | info | `and inferred: []` |

---

## 六、AI 任务入队后的处理流程

### 6.1 图片类型：打标签绕过 OCR 结果

当图片 OCR 失败但仍入队 tag 任务时，`inferenceWorker` 的处理路径：

```typescript
// apps/workers/workers/inference/tagging.ts:325-336
} else if (bookmark.asset) {
  switch (bookmark.asset.assetType) {
    case "image":
      response = await inferTagsFromImage(...);
      break;
    ...
  }
}
```

`inferTagsFromImage()` **不依赖 OCR 提取的文本**，它直接读取图片文件并调用视觉模型：

```typescript
// tagging.ts:142-173
const { asset, metadata } = await readAsset({
  userId: bookmark.userId,
  assetId: bookmark.asset.assetId,
});
...
const base64 = asset.toString("base64");
return inferenceClient.inferFromImage(
  buildImagePrompt(...),
  metadata.contentType,
  base64,
  { ... },
);
```

**重要设计：** 图片 OCR 和 AI 打标签是两条独立路径：
- OCR → `bookmarkAssets.content` → 用于全文搜索
- AI 打标签 → 直接读图片 → 视觉模型 → 生成标签

即使 OCR 失败，只要图片文件存在且非 GIF，AI 打标签仍然可以工作（路径 D-1 除外）。

### 6.2 PDF 类型：打标签依赖提取的文本

```typescript
// tagging.ts:237-262
async function inferTagsFromPDF(...) {
  const prompt = await buildTextPrompt(
    ...
    `Content: ${bookmark.asset.content}`,  // 依赖 OCR 提取的文本
    ...
  );
  return inferenceClient.inferFromText(prompt, { ... });
}
```

PDF 打标签使用提取的文本。但 PDF 文本提取失败会抛异常，所以不会走到 AI 入队步骤。

### 6.3 资产类型的 summarize 任务：会被静默跳过

```typescript
// summarize.ts:20-45
async function fetchBookmarkDetailsForSummary(bookmarkId: string) {
  const bookmark = await db.query.bookmarks.findFirst({
    where: eq(bookmarks.id, bookmarkId),
    columns: { id: true, userId: true, type: true },
    with: {
      link: { ... },
      // If assets (like PDFs with extracted text) should be summarized, extend here
    },
  });
  ...
}

// summarize.ts:97-126
if (bookmarkData.type === BookmarkTypes.LINK && bookmarkData.link) {
  // ... 生成摘要
} else {
  logger.warn(
    `[inference][${jobId}] Bookmark ${bookmarkId} (type: ${bookmarkData.type}) is not a LINK or TEXT type with content, or content is missing. Skipping summary.`,
  );
  return;  // 直接返回，不抛异常
}
```

**关键：**
- `fetchBookmarkDetailsForSummary()` 明确没有查询 `asset` 关联（注释说明）
- 第 97 行只处理 `bookmarkData.type === BookmarkTypes.LINK` 的情况
- 非 LINK 类型（包括 ASSET）直接 `return`，**不抛异常**
- 任务被标记为成功完成
- `onComplete` 回调仍然会调用 `attemptMarkStatus(job.data, "success")`

这也是一条 "success 但无产物" 的路径：`summarizationStatus = "success"` 但 `summary = null`。

---

## 七、完整状态流转

### 7.1 初始状态

创建资产类型书签时（`packages/trpc/routers/bookmarks.ts:242-258` + `packages/db/schema.ts:205-210`）：

| 字段 | 初始值 | 说明 |
|------|--------|------|
| `taggingStatus` | `"pending"` | schema 默认值 |
| `summarizationStatus` | `null` | 代码显式设置，仅 LINK 类型为 `"pending"` |

**资产类型书签的 `summarizationStatus` 初始为 `null`，不是 `"pending"`。**

### 7.2 图片 OCR 成功路径（anythingChanged = true）

```
创建书签
  ↓
taggingStatus="pending", summarizationStatus=null
  ↓
AssetPreprocessingWorker
  ↓
OCR 成功 → anythingChanged=true
  ↓
入队 tag + summarize + 搜索索引
  ↓
inferenceWorker (tag)
  ├─ inferTagsFromImage() → 视觉模型打标签
  ├─ 保存标签
  └─ onComplete → taggingStatus="success"
  ↓
inferenceWorker (summarize)
  ├─ 检测非 LINK → 跳过
  └─ onComplete → summarizationStatus="success"  ⚠️  虚假成功！
  ↓
最终状态：
  taggingStatus="success" ✅ （有真实标签）
  summarizationStatus="success" ⚠️  （但 summary=null）
  bookmarkAssets.content="..." ✅ （有提取文本）
```

### 7.3 图片 OCR 静默失败路径（anythingChanged = false）

```
创建书签
  ↓
taggingStatus="pending", summarizationStatus=null
  ↓
AssetPreprocessingWorker
  ↓
OCR 失败 → anythingChanged=false
  ↓
!isFixMode=true → 仍入队 tag + summarize + 搜索索引  ⚠️
  ↓
inferenceWorker (tag)
  ├─ 非 GIF → inferTagsFromImage() → 视觉模型打标签（绕过 OCR）
  │    ├─ 正常 → 保存标签 → taggingStatus="success" ✅
  │    └─ AI 返回 [] → 无标签 → taggingStatus="success" ⚠️ （路径 E）
  ├─ GIF → 返回 null → 无标签 → taggingStatus="success" ⚠️ （路径 D-1）
  └─ inferenceClient=null → 无标签 → taggingStatus="success" ⚠️ （路径 A）
  ↓
inferenceWorker (summarize)
  ├─ 检测非 LINK → 跳过
  └─ onComplete → summarizationStatus="success"  ⚠️  虚假成功！
  ↓
最终状态：
  taggingStatus="success" （可能有标签，也可能没有，取决于具体路径）
  summarizationStatus="success" ⚠️  （但 summary=null）
  bookmarkAssets.content=null ⚠️ （OCR 失败，无提取文本）
```

### 7.4 PDF 文本提取成功路径

```
创建书签
  ↓
taggingStatus="pending", summarizationStatus=null
  ↓
AssetPreprocessingWorker
  ↓
PDF 文本提取成功 → anythingChanged=true
  ↓
PDF 截图生成 → 通常成功
  ↓
入队 tag + summarize + 搜索索引
  ↓
inferenceWorker (tag)
  ├─ inferTagsFromPDF() → 使用提取的文本打标签
  ├─ 保存标签
  └─ onComplete → taggingStatus="success"
  ↓
inferenceWorker (summarize)
  ├─ 检测非 LINK → 跳过
  └─ onComplete → summarizationStatus="success"  ⚠️  虚假成功！
```

### 7.5 PDF 文本提取失败路径

```
创建书签
  ↓
taggingStatus="pending", summarizationStatus=null
  ↓
AssetPreprocessingWorker
  ↓
PDF 文本为空 → throw Error
  ↓
任务失败 → 进入重试（最多 2 次）
  ↓
重试耗尽 → onError 回调
  ├─ taggingStatus: "pending" → null
  └─ summarizationStatus: null → null（无变化）
  ↓
最终状态：
  taggingStatus=null
  summarizationStatus=null
  bookmarkAssets.content=null
```

**不会入队 AI 任务**，因为异常在入队之前抛出。

---

## 八、状态卡顿的根源分析

"状态卡顿"不是指停留在 `pending`，而是指**状态与实际内容不一致**，或**状态机无法正常推进**。以下是几种典型场景：

### 8.1 场景 1：summarizationStatus 虚假成功

**表现：** `summarizationStatus = "success"` 但 `summary = null`

**原因：**
1. 资产类型书签的 summarize 任务被入队
2. `runSummarization()` 检测到非 LINK 类型，直接 `return`
3. Worker 任务标记为成功（不是失败）
4. `onComplete` 调用 `attemptMarkStatus(..., "success")`
5. `summarizationStatus` 被更新为 `"success"`
6. 但 `summary` 字段从未被写入

**排查难点：** 状态显示成功，容易误导排查者认为"已经处理过了"，但实际上什么都没做。

**可观察证据：**
- 日志（warn 级别）：`Bookmark ... (type: asset) is not a LINK or TEXT type ... Skipping summary.`

---

### 8.2 场景 2：taggingStatus 虚假成功（5 条子路径）

**表现：** `taggingStatus = "success"` 但 `tagsOnBookmarks` 表中没有新的 AI 标签

**对应第五章的 5 条路径：**

| 路径 | 排查关键词 | 日志级别 |
|------|-----------|---------|
| A. inferenceClient 未配置 | `No inference client configured` | debug |
| B. 全局开关关闭 | `disabled in the config` | debug |
| C. 用户开关关闭 | `user has disabled auto-tagging` | debug |
| D. 内容不足 | `due to missing content` / `because it's a GIF` | info |
| E. AI 返回空数组 | `and inferred: []` | info |

**排查难点：** 状态显示成功，但实际上没有标签产物。如果不查数据库，仅看状态字段会被误导。

---

### 8.3 场景 3：AI 返回的标签不符合 schema

**表现：** 任务失败重试，最终标记为 `failure`

**原因：**
```typescript
// tagging.ts:362-388
let tags = openAIResponseSchema.parse(
  parseJsonFromLLMResponse(response.response),
).tags;
...
} catch (e) {
  throw new Error(
    `[inference][${jobId}] The model ignored our prompt and didn't respond with the expected JSON: ...`,
  );
}
```
- AI 返回的 JSON 解析失败或不符合 schema 时抛异常
- 任务失败进入重试（最多 3 次）
- 重试耗尽后标记 `taggingStatus = "failure"`

这是唯一会标记为 `failure` 的场景。

**可观察证据：**
- 日志中有 `The model ignored our prompt and didn't respond with the expected JSON`

---

### 8.4 场景 4：预处理 Worker 永久失败

**表现：** `taggingStatus = null`，`summarizationStatus = null`

**原因：**
- 资产预处理 Worker 任务永久失败（如 PDF 文本为空）
- `onError` 回调将 `taggingStatus` 从 `"pending"` 改为 `null`
- 状态回到"未开始"，不会入队 AI 任务

**可观察证据：**
- assetPreprocessingWorker 错误日志中有 `PDF text is empty` 或其他异常

---

### 8.5 场景 5：真正的 pending 卡顿

**表现：** `taggingStatus = "pending"` 长时间（超过 10 分钟）不动

**这才是真正的"卡顿"，可能原因：**
1. `AssetPreprocessingQueue` 队列积压，Worker 处理不过来
2. `OpenAIQueue` 队列积压
3. inferenceWorker 进程挂了
4. Worker 内部死锁或无限等待

**可观察证据：**
- 检查队列长度：`getQueueClient().getQueue(OpenAIQueue).getJobCounts()`
- 检查 Worker 进程是否存活
- 查看是否有 `Starting an inference job` 日志但没有 `Completed successfully`

---

## 九、关键差异总结

### 9.1 两个 Worker 的失败处理对比

| 维度 | 资产预处理 Worker | AI 推理 Worker |
|------|------------------|----------------|
| 永久失败标记 | `null`（清空状态） | `"failure"`（标记失败） |
| 静默失败处理 | OCR 返回 null 时任务正常完成，仍入队 AI | 5 条路径下任务标记为 success 但实际无内容 |
| 重试次数 | 2 次 | 3 次 |
| summarizationStatus 初始值 | `null`（资产类型不参与自动摘要） | `null`（入队后会被改为 "success"，虚假） |

### 9.2 图片 vs PDF 的处理路径对比

| 维度 | 图片 | PDF |
|------|------|-----|
| OCR 失败是否抛异常 | ❌ 返回 null | ✅ 抛异常 |
| OCR 失败是否入队 AI | ✅ 仍入队 | ❌ 不会执行到入队 |
| AI 打标签是否依赖 OCR | ❌ 绕过，直接读图片 | ✅ 依赖提取的文本 |
| 是否生成截图 | ❌ | ✅ |
| 空内容是否导致永久失败 | ❌（tag 任务通常 success） | ✅ |

### 9.3 taggingStatus vs summarizationStatus 的状态对比

| 维度 | taggingStatus | summarizationStatus |
|------|--------------|---------------------|
| 资产类型初始值 | `"pending"` | `null` |
| success 但无产物路径 | 5 条 | 1 条（非 LINK 跳过） |
| 永久失败标记 | `"failure"` | `"failure"` |
| 无开关时状态 | 可能 success（路径 A/B/C） | 可能 success（虚假） |

---

## 十、排查指南

当遇到状态显示异常时，按以下步骤排查：

### 10.1 检查 `summarizationStatus = "success"` 但 `summary = null`

- 这是正常现象！资产类型目前不支持自动摘要
- 无需修复，是设计如此（但状态显示有误导性）
- 可观察：warn 日志中有 `Skipping summary`

### 10.2 检查 `taggingStatus = "success"` 但没有标签

按优先级排查：
1. **先查数据库**：`SELECT * FROM tagsOnBookmarks WHERE bookmarkId = ? AND attachedBy = 'ai'`
2. **再查日志**，搜索以下关键词（按出现频率排序）：
   - `due to missing content` → 路径 D（info 级别）
   - `because it's a GIF` → 路径 D-1（info 级别）
   - `and inferred: []` → 路径 E（info 级别）
   - `disabled in the config` → 路径 B（debug 级别）
   - `user has disabled auto-tagging` → 路径 C（debug 级别）
   - `No inference client configured` → 路径 A（debug 级别）

### 10.3 检查 `taggingStatus = "pending"` 长时间不动

- 检查队列是否有积压
- 检查 inferenceWorker 是否正常运行
- 检查是否有异常日志
- 这才是真正的"卡顿"，通常是队列或 Worker 问题

### 10.4 检查 `taggingStatus = null`

- 意味着预处理任务永久失败
- 查看 assetPreprocessingWorker 的错误日志
- 常见原因：PDF 文本为空、OCR 配置错误

### 10.5 检查 `taggingStatus = "failure"`

- 意味着 AI 推理任务永久失败
- 查看 inferenceWorker 的错误日志
- 常见原因：AI 返回格式不符合 schema、API 调用失败

---

## 十一、代码文件索引

| 文件路径 | 关键功能 |
|----------|---------|
| `packages/shared/assetdb.ts:51-62` | 两层类型白名单定义 |
| `packages/api/utils/upload.ts:42-143` | 文件上传（SUPPORTED_UPLOAD_ASSET_TYPES 校验） |
| `packages/trpc/routers/bookmarks.ts:314-360` | 创建资产书签（SUPPORTED_BOOKMARK_ASSET_TYPES 校验） |
| `packages/trpc/routers/bookmarks.ts:419-428` | 入队 AssetPreprocessingQueue（fixMode=false） |
| `packages/shared-server/src/queues.ts:238` | fixMode 默认值 false |
| `packages/db/schema.ts:205-210` | taggingStatus/summarizationStatus 字段默认值 |
| `packages/db/schema.ts:404-416` | bookmarkAssets 表定义（assetType: "image" \| "pdf"） |
| `packages/shared/types/bookmarks.ts:174-193` | 请求 schema 中 assetType 枚举约束 |
| `apps/workers/workers/assetPreprocessingWorker.ts:36-106` | Worker 构建 + onError 回调 |
| `apps/workers/workers/assetPreprocessingWorker.ts:266-323` | 图片 OCR 提取（失败不抛异常） |
| `apps/workers/workers/assetPreprocessingWorker.ts:325-360` | PDF 文本提取（空文本抛异常） |
| `apps/workers/workers/assetPreprocessingWorker.ts:175-264` | PDF 截图生成 |
| `apps/workers/workers/assetPreprocessingWorker.ts:372-482` | run() 主函数 + AI 入队条件 |
| `apps/workers/workers/assetPreprocessingWorker.ts:463` | `!isFixMode \|\| anythingChanged` 关键条件 |
| `apps/workers/workers/inference/inferenceWorker.ts:21-41` | AI 推理状态标记（attemptMarkStatus） |
| `apps/workers/workers/inference/inferenceWorker.ts:54-58` | onComplete → success |
| `apps/workers/workers/inference/inferenceWorker.ts:66-68` | onError → failure（重试耗尽） |
| `apps/workers/workers/inference/inferenceWorker.ts:86-92` | inferenceClient 未配置时静默跳过（路径 A） |
| `apps/workers/workers/inference/tagging.ts:133-173` | inferTagsFromImage（绕过 OCR，直接读图片） |
| `apps/workers/workers/inference/tagging.ts:152-157` | GIF 图片跳过打标签（路径 D-1） |
| `apps/workers/workers/inference/tagging.ts:237-262` | inferTagsFromPDF（依赖提取的文本） |
| `apps/workers/workers/inference/tagging.ts:325-351` | 资产类型 tagging 分派逻辑 |
| `apps/workers/workers/inference/tagging.ts:397-398` | connectTags 空数组返回（路径 E） |
| `apps/workers/workers/inference/tagging.ts:510-514` | 全局开关关闭（路径 B） |
| `apps/workers/workers/inference/tagging.ts:535-539` | 用户开关关闭（路径 C） |
| `apps/workers/workers/inference/tagging.ts:569-574` | inferTags 返回 null（路径 D） |
| `apps/workers/workers/inference/summarize.ts:20-45` | fetchBookmarkDetailsForSummary（不查询 asset） |
| `apps/workers/workers/inference/summarize.ts:97-126` | 非 LINK 类型跳过摘要 |
| `packages/shared-server/src/queues.ts:236-249` | AssetPreprocessingQueue 定义 |
