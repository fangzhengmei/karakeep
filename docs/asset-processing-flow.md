# 附件类资源处理流程梳理

## 概述

本文档梳理 Karakeep 项目中附件类资源从上传暂存到信息提取再回写到主记录的完整处理流程，重点回答：

1. 哪些上传类型能进入附件预处理队列
2. 哪些类型会在创建资产书签时被拒绝
3. 图片与 PDF 的后台处理分支分别执行了什么任务
4. "提取未落库但任务成功完成"的分支在什么情况下仍会入队 AI 任务
5. tagging 和 summarization 的状态如何变化
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

## 五、AI 任务入队后的处理流程

### 5.1 图片类型：打标签绕过 OCR 结果

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

即使 OCR 失败，只要图片文件存在，AI 打标签仍然可以工作。

### 5.2 PDF 类型：打标签依赖提取的文本

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

### 5.3 资产类型的 summarize 任务：会被静默跳过

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

---

## 六、完整状态流转

### 6.1 初始状态

创建资产类型书签时（`packages/trpc/routers/bookmarks.ts:242-258` + `packages/db/schema.ts:205-210`）：

| 字段 | 初始值 | 说明 |
|------|--------|------|
| `taggingStatus` | `"pending"` | schema 默认值 |
| `summarizationStatus` | `null` | 代码显式设置，仅 LINK 类型为 `"pending"` |

**资产类型书签的 `summarizationStatus` 初始为 `null`，不是 `"pending"`。**

### 6.2 图片 OCR 成功路径（anythingChanged = true）

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

### 6.3 图片 OCR 静默失败路径（anythingChanged = false）

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
  ├─ inferTagsFromImage() → 视觉模型打标签（绕过 OCR）
  ├─ 保存标签
  └─ onComplete → taggingStatus="success" ✅ （通常成功）
  ↓
inferenceWorker (summarize)
  ├─ 检测非 LINK → 跳过
  └─ onComplete → summarizationStatus="success"  ⚠️  虚假成功！
  ↓
最终状态：
  taggingStatus="success" ✅ （有真实标签）
  summarizationStatus="success" ⚠️  （但 summary=null）
  bookmarkAssets.content=null ⚠️ （OCR 失败，无提取文本）
```

**之前的错误结论：** 认为 OCR 失败后 taggingStatus 会卡在 pending。实际情况是：**taggingStatus 会变为 success，因为 inferTagsFromImage() 绕过了 OCR 结果。**

### 6.4 PDF 文本提取成功路径

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

### 6.5 PDF 文本提取失败路径

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

## 七、状态卡顿的根源分析

"状态卡顿"不是指停留在 `pending`，而是指**状态与实际内容不一致**，或**状态机无法正常推进**。以下是几种典型场景：

### 7.1 场景 1：summarizationStatus 虚假成功

**表现：** `summarizationStatus = "success"` 但 `summary = null`

**原因：**
1. 资产类型书签的 summarize 任务被入队
2. `runSummarization()` 检测到非 LINK 类型，直接 `return`
3. Worker 任务标记为成功（不是失败）
4. `onComplete` 调用 `attemptMarkStatus(..., "success")`
5. `summarizationStatus` 被更新为 `"success"`
6. 但 `summary` 字段从未被写入

**排查难点：** 状态显示成功，容易误导排查者认为"已经处理过了"，但实际上什么都没做。

### 7.2 场景 2：inferenceClient 未配置

**表现：** `taggingStatus = "success"` 但没有任何标签

**原因：**
```typescript
// inferenceWorker.ts:86-92
const inferenceClient = InferenceClientFactory.build();
if (!inferenceClient) {
  logger.debug(
    `[inference][${jobId}] No inference client configured, nothing to do now`,
  );
  return;  // 直接返回，不抛异常
}
```
- `inferenceClient` 为 null 时直接返回
- 任务标记为成功
- `onComplete` 标记 `taggingStatus = "success"`
- 但 `runTagging()` 从未执行，没有标签

### 7.3 场景 3：GIF 图片被跳过打标签

**表现：** `taggingStatus = "success"` 但没有任何标签

**原因：**
```typescript
// tagging.ts:152-157
if (metadata.contentType === ASSET_TYPES.IMAGE_GIF) {
  logger.info(
    `[inference][${jobId}] Skipping inference for bookmark with id "${bookmark.id}" because it's a GIF.`,
  );
  return null;  // 返回 null，不抛异常
}
```
- GIF 图片返回 null
- `runTagging()` 收到 null → 不调用 `connectTags()` → 没有标签
- 但任务成功完成 → `taggingStatus = "success"`

### 7.4 场景 4：AI 返回的标签不符合 schema

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

### 7.5 场景 5：预处理 Worker 永久失败

**表现：** `taggingStatus = null`，`summarizationStatus = null`

**原因：**
- 资产预处理 Worker 任务永久失败（如 PDF 文本为空）
- `onError` 回调将 `taggingStatus` 从 `"pending"` 改为 `null`
- 状态回到"未开始"，不会入队 AI 任务

---

## 八、关键差异总结

### 8.1 两个 Worker 的失败处理对比

| 维度 | 资产预处理 Worker | AI 推理 Worker |
|------|------------------|----------------|
| 永久失败标记 | `null`（清空状态） | `"failure"`（标记失败） |
| 静默失败处理 | OCR 返回 null 时任务正常完成，仍入队 AI | 多种场景下任务标记为成功但实际无内容 |
| 重试次数 | 2 次 | 3 次 |
| summarizationStatus 初始值 | `null`（资产类型不参与自动摘要） | `null`（入队后会被改为 "success"，虚假） |

### 8.2 图片 vs PDF 的处理路径对比

| 维度 | 图片 | PDF |
|------|------|-----|
| OCR 失败是否抛异常 | ❌ 返回 null | ✅ 抛异常 |
| OCR 失败是否入队 AI | ✅ 仍入队 | ❌ 不会执行到入队 |
| AI 打标签是否依赖 OCR | ❌ 绕过，直接读图片 | ✅ 依赖提取的文本 |
| 是否生成截图 | ❌ | ✅ |
| 空内容是否导致永久失败 | ❌（tag 任务通常成功） | ✅ |

---

## 九、排查指南

当遇到状态显示异常时，按以下步骤排查：

1. **检查 `summarizationStatus = "success"` 但 `summary = null`**
   - 这是正常现象！资产类型目前不支持自动摘要
   - 无需修复，是设计如此（但状态显示有误导性）

2. **检查 `taggingStatus = "success"` 但没有标签**
   - 查看 inferenceWorker 日志，搜索 "Skipping" 关键词
   - 可能原因：GIF 图片、`inferenceClient` 未配置、规则引擎过滤了所有标签

3. **检查 `taggingStatus = "pending"` 长时间不动**
   - 检查 AssetPreprocessingQueue 是否有积压
   - 检查 inferenceWorker 是否正常运行
   - 这才是真正的"卡顿"，通常是队列或 Worker 问题

4. **检查 `taggingStatus = null`**
   - 意味着预处理任务永久失败
   - 查看 assetPreprocessingWorker 的错误日志
   - 常见原因：PDF 文本为空、OCR 配置错误

5. **检查 `taggingStatus = "failure"`**
   - 意味着 AI 推理任务永久失败
   - 查看 inferenceWorker 的错误日志
   - 常见原因：AI 返回格式不符合 schema、API 调用失败

---

## 十、代码文件索引

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
| `apps/workers/workers/inference/inferenceWorker.ts:86-92` | inferenceClient 未配置时静默跳过 |
| `apps/workers/workers/inference/tagging.ts:133-173` | inferTagsFromImage（绕过 OCR，直接读图片） |
| `apps/workers/workers/inference/tagging.ts:152-157` | GIF 图片跳过打标签 |
| `apps/workers/workers/inference/tagging.ts:237-262` | inferTagsFromPDF（依赖提取的文本） |
| `apps/workers/workers/inference/tagging.ts:325-351` | 资产类型 tagging 分派逻辑 |
| `apps/workers/workers/inference/summarize.ts:20-45` | fetchBookmarkDetailsForSummary（不查询 asset） |
| `apps/workers/workers/inference/summarize.ts:97-126` | 非 LINK 类型跳过摘要 |
| `packages/shared-server/src/queues.ts:236-249` | AssetPreprocessingQueue 定义 |
