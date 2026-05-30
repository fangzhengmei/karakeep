# 附件类资源处理流程梳理

## 概述

本文档梳理了 Karakeep 项目中附件类资源从上传暂存到信息提取再回写到主记录的完整处理流程，解答了以下关键问题：

1. 解析操作是在前台还是后台执行
2. 不同类型资源处理管线差异
3. 字段抽取失败时的降级处理方式
4. 最终元信息如何与主记录关联

---

## 一、整体架构

### 1.1 执行位置：前台 vs 后台

**结论：解析操作完全在后台执行**

| 阶段 | 执行位置 | 说明 |
|------|-----------|------|
| 文件上传 | 前台 (API 层) | packages/api/utils/upload.ts |
| 数据库记录创建 | 前台 (API 层) | 同步创建 assets 表记录 |
| 信息解析/提取 | **后台 Worker | apps/workers/workers/assetPreprocessingWorker.ts |
| OCR 文本识别 | **后台 Worker** | 异步队列处理 |
| AI 标签/摘要 | **后台 Worker** | apps/workers/workers/inference/ |

**关键代码位置：

- **上传阶段**（前台）：`packages/api/utils/upload.ts:42-143`
  - 接收文件上传
  - 检测 MIME 类型
  - 存储到临时文件
  - 创建数据库记录（assetType = UNKNOWN）
  - 保存到资产存储（本地文件系统或 S3）

- **解析阶段**（后台）：`apps/workers/workers/assetPreprocessingWorker.ts
  - 从 AssetPreprocessingQueue 消费任务
  - 执行 OCR/文本提取
  - 触发 AI 标签和摘要生成

---

## 二、不同类型资源处理管线

### 2.1 支持的资源类型

| 类型 | MIME 类型 | 处理方式 |
|------|------------|----------|
| 图片 | image/gif, image/jpeg, image/png, image/webp | OCR 文本提取 |
| PDF | application/pdf | 文本提取 + 截图生成 |
| 视频 | video/mp4, video/webm, video/x-matroska | 仅存储，无内容提取 |
| HTML | text/html | （主要用于链接存档 |

### 2.2 图片类型处理管线 (`assetPreprocessingWorker.ts:420-431

```
上传 → AssetPreprocessingQueue
    ↓
读取资产文件
    ↓
OCR 文本提取（二选一）
    ├─ Tesseract OCR（默认）
    │   └─ 置信度阈值检查（serverConfig.ocr.confidenceThreshold
    │       ├─ 低于阈值 → 返回 null
    │       └─ 高于阈值 → 保存文本
    └─ LLM OCR（配置开启时）
        └─ 失败时回退到 Tesseract
    ↓
保存提取的文本 → bookmarkAssets.content
    ↓
触发 OpenAIQueue（标签+摘要）
```

**关键代码：

- `readImageText()`: `assetPreprocessingWorker.ts:108-124
- `readImageTextWithLLM()`: `assetPreprocessingWorker.ts:126-156
- `extractAndSaveImageText()`: `assetPreprocessingWorker.ts:266-323

### 2.3 PDF 类型处理管线 (`assetPreprocessingWorker.ts:432-447

```
上传 → AssetPreprocessingQueue
    ↓
读取资产文件
    ↓
├─ 文本提取（pdf2json）
│   └─ 保存到 bookmarkAssets.content + metadata
    ↓
└─ 生成首页截图（pdf2pic）
    └─ 存储为独立资产（ASSET_SCREENSHOT）
    ↓
触发 OpenAIQueue（标签+摘要）
```

**关键代码：

- `readPDFText()`: `assetPreprocessingWorker.ts:158-173
- `extractAndSavePDFText()`: `assetPreprocessingWorker.ts:325-360
- `extractAndSavePDFScreenshot()`: `assetPreprocessingWorker.ts:175-264

### 2.4 链接类型处理管线（通过 crawlerWorker）

```
上传 → LinkCrawlerQueue
    ↓
爬虫抓取页面内容
    ↓
提取标题、描述、作者等元数据
    ↓
保存到 bookmarkLinks 表
    ↓
触发 OpenAIQueue（标签+摘要）
```

### 2.5 文本类型处理管线

```
创建书签 → 直接保存文本内容
    ↓
触发 OpenAIQueue（标签+摘要）
```

---

## 三、字段抽取失败时的降级处理

### 3.1 OCR 文本提取降级策略

**3.1.1 Tesseract OCR 置信度阈值

```typescript
// assetPreprocessingWorker.ts:117-119
if (ret.data.confidence <= serverConfig.ocr.confidenceThreshold) {
  return null;
}
```

- 置信度低于阈值时，返回 `null`，不保存文本内容
- bookmarkAssets.content 保持为 `null`

**3.1.2 LLM OCR 失败回退

```typescript
// assetPreprocessingWorker.ts:131-135
if (!inferenceClient) {
  logger.warn("[assetPreprocessing] LLM OCR is enabled but no inference client is configured. Falling back to Tesseract.");
  return readImageText(buffer);
}
```

- LLM OCR 配置不可用时，自动回退到 Tesseract OCR
- LLM OCR 执行出错时，记录错误日志，但不自动回退

### 3.2 队列重试机制

| 队列 | 重试次数 | 超时时间 |
|------|-----------|----------|
| AssetPreprocessingQueue | 2次 | serverConfig.assetPreprocessing.jobTimeoutSec |
| OpenAIQueue | 3次 | serverConfig.inference.jobTimeoutSec |

**关键代码：`packages/shared-server/src/queues.ts:243-249

### 3.3 永久失败处理

当所有重试耗尽后：

```typescript
// assetPreprocessingWorker.ts:67-93
if (bookmarkId && job.numRetriesLeft == 0) {
  await db.transaction(async (tx) => {
    await tx
      .update(bookmarks)
      .set({
        taggingStatus: null,
      })
      .where(
        and(
          eq(bookmarks.id, bookmarkId),
          eq(bookmarks.taggingStatus, "pending"),
        ),
      );
    // ... summarizationStatus 同理
  });
}
```

- taggingStatus/summarizationStatus 从 "pending" → `null`
- 不标记为 "failure"，而是清空状态
- 保持 bookmarkAssets.content 保持为 `null`

### 3.4 PDF 截图生成失败降级

```typescript
// assetPreprocessingWorker.ts:252-263
catch (error) {
  if (error instanceof StorageQuotaError) {
    logger.warn(`[assetPreprocessing][${jobId}] Skipping PDF screenshot due to quota exceeded: ${error.message}");
    return true; // 返回 true 表示任务成功完成，只是跳过了截图
  }
  logger.error(`[assetPreprocessing][${jobId}] Failed to process PDF screenshot: ${error}");
  return false;
}
```

- 存储配额超限：静默跳过截图生成，任务标记为成功
- 其他错误：记录错误，返回 false

---

## 四、元信息与主记录关联

### 4.1 数据库表结构关系

```
bookmarks（主记录表）
    ├─ id (PK)
    ├─ type: "link" | "text" | "asset"
    ├─ title, summary, note
    ├─ taggingStatus: pending | failure | success | null
    └─ summarizationStatus: pending | failure | success | null
    │
    ├─ bookmarkLinks（链接类型子表）
    │   ├─ id (FK → bookmarks.id)
    │   └─ url, title, description, author, ...
    │
    ├─ bookmarkTexts（文本类型子表）
    │   ├─ id (FK → bookmarks.id)
    │   └─ text
    │
    └─ bookmarkAssets（资产类型子表）
        ├─ id (FK → bookmarks.id)
        ├─ assetType: "image" | "pdf"
        ├─ assetId (→ assets.id)
        ├─ content (提取的文本内容）
        └─ metadata (JSON 格式的元数据）
        │
        └─ assets（文件存储表）
            ├─ id (PK)
            ├─ assetType: 多种类型枚举
            ├─ bookmarkId (FK → bookmarks.id)
            ├─ userId
            ├─ contentType
            ├─ size
            └─ fileName
```

**Schema 定义位置：** `packages/db/schema.ts:295-333

### 4.2 关联建立流程

**4.2.1 上传时的初始状态

```typescript
// packages/api/utils/upload.ts:103-116
await db
  .insert(assets)
  .values({
    id: newAssetId(),
    assetType: AssetTypes.UNKNOWN,  // 初始状态
    bookmarkId: null,                // 尚未关联
    userId: user.id,
    contentType,
    size: data.size,
    fileName,
  })
  .returning();
```

- 上传时 assetType = UNKNOWN
- bookmarkId = null（尚未关联到主记录）

**4.2.2 创建书签时建立关联

```typescript
// packages/trpc/routers/bookmarks.ts:314-360
case BookmarkTypes.ASSET: {
  const [asset] = await tx
    .insert(bookmarkAssets)
    .values({
      id: bookmark.id,
      assetType: input.assetType,
      assetId: input.assetId,
      content: null,
      metadata: null,
      fileName: input.fileName ?? null,
      sourceUrl: input.sourceUrl ?? null,
    })
    .returning();
  // ...
  await tx
    .update(assets)
    .set({
      bookmarkId: bookmark.id,
      assetType: AssetTypes.BOOKMARK_ASSET,
    })
    .where(
      and(
        eq(assets.id, input.assetId),
        eq(assets.userId, ctx.user.id),
      ),
    );
}
```

- 创建 bookmarkAssets 记录，关联 bookmark.id
- 更新 assets 表的 bookmarkId 和 assetType

**4.2.3 信息提取后回写

```typescript
// assetPreprocessingWorker.ts:315-321
await db
  .update(bookmarkAssets)
  .set({
    content: imageText,  // 提取的文本
    metadata: null,
  })
  .where(eq(bookmarkAssets.id, bookmark.id));
```

- 提取的文本保存到 bookmarkAssets.content
- PDF 元数据保存到 bookmarkAssets.metadata（JSON 格式）

**4.2.4 PDF 截图作为独立资产

```typescript
// assetPreprocessingWorker.ts:238-246
await db.insert(assets).values({
  id: assetId,
  bookmarkId: bookmark.id,
  userId: bookmark.userId,
  assetType: AssetTypes.ASSET_SCREENSHOT,
  contentType,
  size: screenshot.buffer.byteLength,
  fileName,
});
```

- 截图作为独立的 assets 记录
- assetType = ASSET_SCREENSHOT
- 通过 bookmarkId 关联到主记录

### 4.3 AI 标签与主记录关联

```typescript
// apps/workers/workers/inference/tagging.ts:401-480
async function connectTags(bookmarkId: string, inferredTags: string[], userId: string) {
  // 匹配已有标签
  // 创建新标签
  // 删除旧的 AI 标签
  // 关联新标签
}
```

- tagsOnBookmarks 表存储标签与书签的关联
- attachedBy: "ai" | "human" 区分 AI 自动标签和人工标签

---

## 五、完整流程时序图

```
用户
  ↓ (上传文件)
  │
  ▼
API 层 (uploadAsset)
  ├─ 检测 MIME 类型
  ├─ 检查存储配额
  ├─ 保存到临时文件
  ├─ 插入 assets 表 (UNKNOWN, bookmarkId=null)
  └─ 返回 assetId
  │
  ▼
用户 (创建资产类型书签)
  │
  ▼
tRPC 层 (createBookmark)
  ├─ 插入 bookmarks 表
  ├─ 插入 bookmarkAssets 表
  ├─ 更新 assets 表 (BOOKMARK_ASSET, bookmarkId=xxx)
  └─ 入队 AssetPreprocessingQueue
  │
  ▼
Worker 层 (assetPreprocessingWorker)
  ├─ 读取资产文件
  ├─ 根据类型处理：
  │   ├─ 图片 → OCR 提取文本
  │   └─ PDF → 文本提取 + 截图生成
  ├─ 更新 bookmarkAssets.content/metadata
  └─ 入队 OpenAIQueue (tag + summarize)
  │
  ▼
Worker 层 (inferenceWorker)
  ├─ 读取 bookmark 内容
  ├─ 调用 AI 接口
  ├─ 解析返回结果
  ├─ 保存标签/摘要
  └─ 更新搜索索引
```

---

## 六、关键配置项

| 配置项 | 位置 | 说明 |
|--------|------|------|
| OCR 语言 | serverConfig.ocr.langs | Tesseract OCR 语言包 |
| OCR 置信度阈值 | serverConfig.ocr.confidenceThreshold | 低于此值不保存 |
| OCR 缓存目录 | serverConfig.ocr.cacheDir | Tesseract 缓存路径 |
| LLM OCR 开关 | serverConfig.ocr.useLLM | 是否使用 LLM 进行 OCR |
| 资产预处理并发数 | serverConfig.assetPreprocessing.numWorkers | Worker 数量 |
| 资产预处理超时 | serverConfig.assetPreprocessing.jobTimeoutSec | 单任务超时 |
| AI 推理并发数 | serverConfig.inference.numWorkers | Worker 数量 |
| AI 推理超时 | serverConfig.inference.jobTimeoutSec | 单任务超时 |
| 自动标签开关 | serverConfig.inference.enableAutoTagging | 全局开关 |
| 自动摘要开关 | serverConfig.inference.enableAutoSummarization | 全局开关 |

---

## 七、代码文件索引

| 文件路径 | 功能 |
|----------|------|
| `packages/api/utils/upload.ts` | 文件上传处理 |
| `packages/api/utils/assets.ts` | 资产文件服务 |
| `packages/shared/assetdb.ts` | 资产存储抽象（本地/S3） |
| `packages/db/schema.ts` | 数据库表结构 |
| `packages/trpc/routers/bookmarks.ts` | 书签创建与资产关联 |
| `packages/shared-server/src/queues.ts` | 任务队列定义 |
| `apps/workers/workers/assetPreprocessingWorker.ts` | 资产预处理 Worker |
| `apps/workers/workers/inference/inferenceWorker.ts` | AI 推理 Worker |
| `apps/workers/workers/inference/tagging.ts` | AI 标签生成 |
| `apps/workers/workers/inference/summarize.ts` | AI 摘要生成 |
