# 附件类资源处理流程梳理

## 概述

本文档梳理 Karakeep 项目中附件类资源从上传暂存到信息提取再回写到主记录的完整处理流程，重点回答：

1. 哪些上传类型能进入附件预处理队列
2. 哪些类型会在创建资产书签时被拒绝
3. 图片与 PDF 的后台处理分支分别执行了什么任务
4. 提取失败后状态字段如何回退

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

## 四、提取失败后状态字段的回退

### 4.1 初始状态

创建资产类型书签时，`bookmarks` 表的初始状态（`packages/trpc/routers/bookmarks.ts:242-258` + `packages/db/schema.ts:205-210`）：

| 字段 | 初始值 |
|------|--------|
| `taggingStatus` | `"pending"`（schema 默认值） |
| `summarizationStatus` | `null`（代码显式设置，仅 LINK 类型为 `"pending"`） |

**资产类型书签的 `summarizationStatus` 初始为 `null`，不是 `"pending"`。** 这是代码中的刻意设计（注释写明："Only links currently support summarization"）。

### 4.2 正常完成路径

`run()` 函数正常执行完毕后，在 `!isFixMode || anythingChanged` 条件下：

```typescript
// assetPreprocessingWorker.ts:463-481
if (!isFixMode || anythingChanged) {
  await OpenAIQueue.enqueue({ bookmarkId, type: "tag" }, enqueueOpts);
  await OpenAIQueue.enqueue({ bookmarkId, type: "summarize" }, enqueueOpts);
  await triggerSearchReindex(bookmarkId, enqueueOpts);
}
```

- **`anythingChanged = false`**（如图片 OCR 返回 null、PDF fixMode 跳过）：**不会入队 OpenAI 任务，也不会触发搜索索引更新**
- **`anythingChanged = true`**：入队 tag + summarize + 搜索索引

### 4.3 图片 OCR 提取返回 null（静默失败）

当 OCR 置信度不足、OCR 配置为空语言列表、或 LLM 返回空文本时，`extractAndSaveImageText()` 返回 `false`：

- `anythingChanged` 保持 `false`
- `bookmarkAssets.content` 保持 `null`（未更新）
- 不入队 OpenAI 任务
- **整个 Worker 任务正常完成（不触发 onError）**
- `taggingStatus` 保持 `"pending"`，`summarizationStatus` 保持 `null`

**这意味着：图片 OCR 静默失败后，`taggingStatus` 会永远停留在 `"pending"`，不会被清理。**

### 4.4 PDF 文本为空（抛异常）

```typescript
// assetPreprocessingWorker.ts:344-348
if (!pdfParse?.text) {
  throw new Error(
    `[assetPreprocessing][${jobId}] PDF text is empty. Please make sure that the PDF includes text and not just images.`,
  );
}
```

- 抛出异常 → Worker 任务失败 → 进入重试（最多 2 次）
- 重试耗尽后触发 `onError` 回调

### 4.5 Worker 任务永久失败后的回退（onError）

```typescript
// assetPreprocessingWorker.ts:67-93
if (bookmarkId && job.numRetriesLeft == 0) {
  await db.transaction(async (tx) => {
    await tx
      .update(bookmarks)
      .set({ taggingStatus: null })
      .where(
        and(
          eq(bookmarks.id, bookmarkId),
          eq(bookmarks.taggingStatus, "pending"),
        ),
      );
    await tx
      .update(bookmarks)
      .set({ summarizationStatus: null })
      .where(
        and(
          eq(bookmarks.id, bookmarkId),
          eq(bookmarks.summarizationStatus, "pending"),
        ),
      );
  });
}
```

**回退规则：**

| 条件 | 操作 |
|------|------|
| `taggingStatus = "pending"` | → 设为 `null` |
| `taggingStatus ≠ "pending"` | → 不修改（WHERE 条件不匹配） |
| `summarizationStatus = "pending"` | → 设为 `null`（对资产书签来说，初始就是 `null`，无实际变化） |
| `summarizationStatus ≠ "pending"` | → 不修改 |

**注意：这里不是设为 `"failure"`，而是清空为 `null`。** 这意味着永久失败后状态回到"未开始"而非"已失败"。

### 4.6 PDF 截图生成失败

截图失败不影响整体任务状态（不会导致 Worker 任务抛异常），有三种情况：

| 情况 | 返回值 | 对 anythingChanged 的影响 |
|------|--------|--------------------------|
| 存储配额超限 | `true` | 计为"有变更" |
| 渲染失败（buffer 为空或其他错误） | `false` | 不计入 |
| fixMode 跳过 | `false` | 不计入 |

截图失败不会导致任务重试，也不会触发 `onError`。

### 4.7 OpenAI Worker 的失败回退

```typescript
// apps/workers/workers/inference/inferenceWorker.ts:66-68
if (job.numRetriesLeft == 0) {
  workerStatsCounter.labels("inference", "failed_permanent").inc();
  await attemptMarkStatus(job?.data, "failure");
}
```

```typescript
// inferenceWorker.ts:21-41
async function attemptMarkStatus(jobData, status: "success" | "failure") {
  const request = zOpenAIRequestSchema.parse(jobData);
  await db
    .update(bookmarks)
    .set({
      ...(request.type === "summarize" ? { summarizationStatus: status } : {}),
      ...(request.type === "tag" ? { taggingStatus: status } : {}),
    })
    .where(eq(bookmarks.id, request.bookmarkId));
}
```

| 阶段 | 成功 | 永久失败 |
|------|------|---------|
| tag 任务完成 | `taggingStatus` → `"success"` | `taggingStatus` → `"failure"` |
| summarize 任务完成 | `summarizationStatus` → `"success"` | `summarizationStatus` → `"failure"` |

**与资产预处理 Worker 不同，AI 推理 Worker 永久失败时标记为 `"failure"` 而非 `null`。**

---

## 五、完整状态流转图

### 5.1 资产预处理阶段

```
                    ┌─────────────────────┐
                    │  创建资产书签         │
                    │  taggingStatus=pending│
                    │  summarizationStatus=null│
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │ AssetPreprocessing   │
                    │     Worker           │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼──────┐ ┌──────▼────────┐ ┌─────▼──────┐
     │ 图片: OCR成功  │ │ 图片: OCR null│ │ PDF: 异常   │
     │ changed=true  │ │ changed=false │ │ (文本为空)  │
     └────────┬──────┘ └──────┬────────┘ └─────┬──────┘
              │               │                │
     ┌────────▼──────┐ ┌─────▼──────┐  ┌──────▼──────┐
     │ 入队 OpenAI   │ │ 不入队     │  │ 重试(≤2次)  │
     │ (tag+summarize)│ │ taggingStatus│  │             │
     │               │ │ 保持pending │  │             │
     └───────────────┘ └────────────┘  │             │
                                        │     重试耗尽│
                                        └──────┬──────┘
                                               │
                                    ┌──────────▼──────────┐
                                    │ onError 回调         │
                                    │ taggingStatus:       │
                                    │   "pending" → null   │
                                    │ summarizationStatus: │
                                    │   null → null (无变化)│
                                    └─────────────────────┘
```

### 5.2 AI 推理阶段（仅当 anythingChanged=true 时触发）

```
┌─────────────────────┐
│ OpenAIQueue 入队     │
│ type: "tag"          │
│ type: "summarize"    │
└──────────┬──────────┘
           │
   ┌───────▼────────┐
   │  inferenceWorker │
   └───────┬────────┘
           │
    ┌──────┼──────┐
    │             │
 成功           永久失败(3次)
    │             │
 taggingStatus   taggingStatus
 → "success"     → "failure"
    │             │
 summarizationStatus  summarizationStatus
 → "success"     → "failure"
```

---

## 六、关键差异总结

| 维度 | 资产预处理 Worker | AI 推理 Worker |
|------|------------------|----------------|
| 永久失败标记 | `null`（清空状态） | `"failure"` |
| 静默失败处理 | OCR 返回 null 时任务正常完成，状态残留 `"pending"` | 不适用（要么成功要么异常） |
| 重试次数 | 2 次 | 3 次 |
| summarizationStatus 初始值 | `null`（资产类型不参与自动摘要） | `"pending"`（但资产类型入队时为 null，实际不会被处理） |

---

## 七、代码文件索引

| 文件路径 | 关键功能 |
|----------|---------|
| `packages/shared/assetdb.ts:51-62` | 两层类型白名单定义 |
| `packages/api/utils/upload.ts:42-143` | 文件上传（SUPPORTED_UPLOAD_ASSET_TYPES 校验） |
| `packages/trpc/routers/bookmarks.ts:314-360` | 创建资产书签（SUPPORTED_BOOKMARK_ASSET_TYPES 校验） |
| `packages/trpc/routers/bookmarks.ts:419-428` | 入队 AssetPreprocessingQueue |
| `packages/db/schema.ts:205-210` | taggingStatus/summarizationStatus 字段默认值 |
| `packages/db/schema.ts:404-416` | bookmarkAssets 表定义（assetType: "image" \| "pdf"） |
| `packages/shared/types/bookmarks.ts:174-193` | 请求 schema 中 assetType 枚举约束 |
| `apps/workers/workers/assetPreprocessingWorker.ts:36-106` | Worker 构建 + onError 回调 |
| `apps/workers/workers/assetPreprocessingWorker.ts:266-323` | 图片 OCR 提取 |
| `apps/workers/workers/assetPreprocessingWorker.ts:325-360` | PDF 文本提取 |
| `apps/workers/workers/assetPreprocessingWorker.ts:175-264` | PDF 截图生成 |
| `apps/workers/workers/assetPreprocessingWorker.ts:372-482` | run() 主函数 |
| `apps/workers/workers/inference/inferenceWorker.ts:21-41` | AI 推理状态标记 |
| `packages/shared-server/src/queues.ts:236-249` | AssetPreprocessingQueue 定义 |
