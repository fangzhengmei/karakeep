# Karakeep 附件预览流程分析

## 概述

本文档分析 Karakeep 项目中打开收藏附件进行预览的完整流程，包括缓存机制、转码处理、状态切换和资源释放。

---

## 1. 系统架构概览

### 1.1 核心模块

| 模块 | 位置 | 职责 |
|------|------|------|
| 资产数据库 | `packages/shared/assetdb.ts` | 资产存储、读取、缓存管理 |
| 资产预处理 Worker | `apps/workers/workers/assetPreprocessingWorker.ts` | OCR 识别、PDF 截图生成、文本提取 |
| 视频处理 Worker | `apps/workers/workers/videoWorker.ts` | 视频下载与转码 |
| 资产服务 API | `packages/api/utils/assets.ts` | HTTP 资源服务、流式传输 |
| 前端预览组件 | `apps/web/components/dashboard/preview/` | UI 渲染、状态管理 |

---

## 2. 资产存储层 (AssetDB)

### 2.1 存储策略

**文件位置**: `packages/shared/assetdb.ts:134-652`

系统支持两种存储后端：

1. **本地文件系统存储** (`LocalFileSystemAssetStore`)
   - 目录结构: `{rootPath}/{userId}/{assetId}/`
   - 数据文件: `asset.bin` (二进制内容)
   - 元数据: `metadata.json` (contentType, fileName)

2. **S3 兼容存储** (`S3AssetStore`)
   - 对象键: `{userId}/{assetId}`
   - 元数据通过 S3 对象 metadata 存储

### 2.2 读取优化机制

```typescript
// 支持范围请求 (Range Requests)
async readAsset({ userId, assetId, start, end }) {
  // 本地文件系统: 使用文件句柄分块读取
  // S3: 使用 HTTP Range 头
}

// 创建可读流 - 用于大文件流式传输
async createAssetReadStream({ userId, assetId, start, end })
```

**关键设计点**:
- **范围请求支持**: 支持 `bytes=start-end` 格式的范围请求，用于视频播放和 PDF 按需加载
- **流式传输**: 大文件不一次性加载到内存，通过流处理
- **缓存头**: `Cache-Control: private, max-age=31536000, immutable` - 浏览器端永久缓存

---

## 3. 资产预处理流程 (转码/提取)

### 3.1 预处理队列系统

**文件位置**: `apps/workers/workers/assetPreprocessingWorker.ts:36-106`

预处理 Worker 基于队列系统：

```
用户上传资产
    ↓
入队 AssetPreprocessingQueue
    ↓
Worker 消费任务 (并发数可配置)
    ↓
根据资产类型分支处理
```

### 3.2 图片处理流程

**文件位置**: `apps/workers/workers/assetPreprocessingWorker.ts:266-323`

```
图片资产
    ↓
├─→ OCR 文本提取 (Tesseract.js 或 LLM)
│    ├─→ 置信度检查 (≥ 配置阈值)
│    └─→ 保存提取文本到 bookmarkAssets.content
    ↓
└─→ 触发后续处理: 标签生成、摘要、搜索索引
```

**资源释放**:
- Tesseract Worker 使用 `finally` 块确保终止: `await worker.terminate()` (L122)
- 缓冲区在函数返回后自动 GC

### 3.3 PDF 处理流程

**文件位置**: `apps/workers/workers/assetPreprocessingWorker.ts:325-360`

```
PDF 资产
    ├─→ 文本提取 (pdf2json)
    │    └─→ 保存 content + metadata
    └─→ 截图生成 (pdf2pic → GraphicsMagick + Ghostscript)
         ├─→ 检查是否已有截图 (fixMode 时跳过)
         ├─→ 配额检查 (StorageQuota)
         ├─→ 生成 PNG 截图 (第一页)
         └─→ 保存为独立资产 (assetType: assetScreenshot)
```

**关键分支逻辑** (`AssetContentSection.tsx:20-40`):
```typescript
const BIG_FILE_SIZE = 20 * 1024 * 1024; // 20MB

// 初始视图策略:
if (文件 > 20MB && 存在截图) {
  默认显示截图视图
} else {
  默认显示 PDF iframe
}
```

---

## 4. 视频处理流程

**文件位置**: `apps/workers/workers/videoWorker.ts`

### 4.1 视频下载转码流程

```
视频链接
    ↓
URL 规范化 + 重定向验证
    ↓
yt-dlp 下载 (支持代理)
    ↓
临时文件: os.tmpdir()/video_downloads/{assetId}*
    ↓
配额检查 → 保存到 AssetDB
    ↓
更新数据库关联 → 删除旧资产
    ↓
清理临时文件
```

### 4.2 资源释放关键点

1. **下载取消**: `execa("yt-dlp", { cancelSignal: job.abortSignal })` (L155-157)
2. **失败清理**: `deleteLeftOverAssetFile()` 在异常时调用 (L183, L234)
3. **旧资产清理**: `silentDeleteAsset(userId, oldVideoAssetId)` (L224)

---

## 5. 前端预览状态机

### 5.1 BookmarkPreview 主状态机

**文件位置**: `apps/web/components/dashboard/preview/BookmarkPreview.tsx`

```
初始化
    ↓
┌─────────────────────────────────┐
│ 加载 bookmark 数据 (useQuery)   │
└─────────────────────────────────┘
    ↓
内容类型判断 (bookmark.content.type)
    ├─→ LINK → LinkContentSection
    ├─→ TEXT → TextContentSection
    └─→ ASSET → AssetContentSection
```

### 5.2 AssetContentSection 分支逻辑

**文件位置**: `apps/web/components/dashboard/preview/AssetContentSection.tsx:107-118`

```
ASSET 类型
    ├─→ image → ImageContentSection (Next.js Image)
    └─→ pdf → PDFContentSection
              ├─→ state: "screenshot" → <Image />
              └─→ state: "pdf" → <iframe />
```

**状态切换**:
- 使用 `useState(initialSection)` 管理当前视图
- 用户通过 `<Select>` 手动切换
- 初始状态由文件大小 + 截图存在性决定

### 5.3 LinkContentSection 多视图状态机

**文件位置**: `apps/web/components/dashboard/preview/LinkContentSection.tsx:118-291`

```
LINK 类型
    │
    ├─→ 自定义渲染器 (AmazonRenderer, TikTokRenderer, etc.)
    │
    ├─→ "cached" → ReaderView (HTML 高亮阅读器)
    │    └─→ useQuery 加载 htmlContent → 滚动跟踪 → 高亮交互
    │
    ├─→ "screenshot" → <Image />
    ├─→ "archive" → <iframe sandbox /> (完整页面存档)
    ├─→ "video" → <video controls />
    └─→ "pdf" → <iframe />
```

**状态管理**:
- 使用 `nuqs` 的 `useQueryState` 持久化到 URL query 参数
- 默认视图: 优先自定义渲染器，其次 "cached"

### 5.4 ReaderView 阅读状态机

**文件位置**: `apps/web/components/dashboard/preview/ReaderView.tsx`

```
ReaderView
    ├─→ 加载状态 → FullPageSpinner
    ├─→ 无内容 → 错误提示 (FileX 图标)
    └─→ 正常阅读
         ├─→ ScrollProgressTracker (进度跟踪)
         ├─→ BookmarkHTMLHighlighter (高亮交互)
         └─→ ReadingProgressBanner (继续阅读提示)
```

---

## 6. HTTP 资源服务层

**文件位置**: `packages/api/utils/assets.ts:12-75`

### 6.1 服务流程

```
GET /api/assets/{assetId}
    ↓
并行读取 metadata + size
    ↓
设置安全头:
  - Content-Type
  - X-Content-Type-Options: nosniff
  - Cache-Control: private, max-age=31536000, immutable
  - CSP: sandbox + 严格来源限制
    ↓
Range 头检测?
    ├─→ 是 → 206 Partial Content + Content-Range
    └─→ 否 → 200 OK + Content-Length
    ↓
流式传输 (hono/streaming)
```

### 6.2 安全策略

**内容安全策略** (CSP):
```
sandbox                          ; 沙箱模式
default-src 'none'               ; 默认禁止所有
img-src https: data: blob:       ; 图片来源
style-src 'unsafe-inline' ...    ; 样式来源
media-src https: data: blob:     ; 媒体来源
```

---

## 7. 资源释放机制

### 7.1 服务器端资源释放

| 资源类型 | 释放位置 | 释放方式 |
|---------|---------|---------|
| Tesseract Worker | `assetPreprocessingWorker.ts:122` | `finally { worker.terminate() }` |
| 文件句柄 | `assetdb.ts:229-243` | `finally { fd.close() }` |
| 临时视频文件 | `videoWorker.ts:247-271` | `fs.promises.rm()` |
| 旧版本资产 | `videoWorker.ts:224` | `silentDeleteAsset()` |
| S3 连接 | S3Client 内部 | 连接池管理 |

### 7.2 客户端资源释放

**注意**: 当前前端代码未显式实现组件卸载时的资源清理：

1. **iframe 资源**: 组件卸载时浏览器自动回收，但大 PDF 可能保留内存
2. **video 元素**: 组件卸载时应调用 `video.pause()` 并清除 src
3. **React Query 缓存**: 由 TanStack Query 自动管理（可配置 staleTime）

**潜在改进点**:
```tsx
// 建议添加的清理逻辑
useEffect(() => {
  return () => {
    // 组件卸载时清理 video 元素
    const video = document.querySelector('video');
    if (video) {
      video.pause();
      video.src = '';
      video.load();
    }
  };
}, []);
```

---

## 8. 缓存层级设计

### 8.1 多层缓存策略

```
┌─────────────────────────────────┐
│ 浏览器 HTTP 缓存                │
│ Cache-Control: 1 年 immutable   │
└─────────────────────────────────┘
              ↓
┌─────────────────────────────────┐
│ React Query 客户端缓存          │
│ bookmark 数据 + htmlContent     │
└─────────────────────────────────┘
              ↓
┌─────────────────────────────────┐
│ 资产预处理结果缓存              │
│ - OCR 文本                      │
│ - PDF 截图                      │
│ (fixMode 时跳过重复处理)        │
└─────────────────────────────────┘
              ↓
┌─────────────────────────────────┐
│ 存储层 (本地 FS / S3)           │
│ 原始资产数据                    │
└─────────────────────────────────┘
```

### 8.2 缓存失效机制

1. **浏览器缓存**: 基于 `immutable` 标志，资产 ID 变化时自动失效
2. **React Query**: 通过 `refetchInterval` 轮询更新（正在抓取的 bookmark）
3. **预处理缓存**: 通过 `fixMode` 参数 + 数据库字段检查 (`alreadyHasText` / `alreadyHasScreenshot`)

---

## 9. 类型分支汇总

### 9.1 Bookmark 类型分支

| 类型 | 预览方式 | 预处理 |
|------|---------|--------|
| LINK | ReaderView / Screenshot / Archive / Video / PDF / 自定义渲染器 | 网页抓取 + 可选视频下载 |
| TEXT | TextContentSection (Markdown) | 无 |
| ASSET (image) | ImageContentSection | OCR 文本提取 |
| ASSET (pdf) | PDF iframe / Screenshot | 文本提取 + 首页截图 |

### 9.2 资产类型枚举

**文件位置**: `packages/shared/assetdb.ts:23-70`

- **图片**: `image/gif`, `image/jpeg`, `image/png`, `image/webp`
- **文档**: `application/pdf`
- **视频**: `video/mp4`, `video/webm`, `video/x-matroska`
- **网页**: `text/html`
- **归档**: `application/zip`

---

## 10. 关键代码路径

### 10.1 打开附件预览的完整调用链

```
用户点击书签
    ↓
BookmarkPreview.tsx
  ↳ useQuery(api.bookmarks.getBookmark)
    ↳ tRPC → packages/trpc/routers/bookmarks.ts
      ↳ DB 查询
    ↓
  ↳ 根据 content.type 分支
    ↓
    ├─→ AssetContentSection.tsx
    │    ↳ PDFContentSection / ImageContentSection
    │         ↳ getAssetUrl() → /api/assets/{id}
    │              ↳ packages/api/utils/assets.ts#serveAsset
    │                   ↳ assetdb.createAssetReadStream
    └─→ LinkContentSection.tsx
         ↳ ReaderView (htmlContent)
         ↳ 或其他视图
```

### 10.2 预处理调用链

```
资产创建
    ↓
入队 AssetPreprocessingQueue
    ↓
assetPreprocessingWorker.run()
  ↳ readAsset()
  ↳ 类型分支 (image/pdf)
    ├─→ extractAndSaveImageText()
    │    ↳ readImageText() / readImageTextWithLLM()
    │    └─→ UPDATE bookmarkAssets
    └─→ extractAndSavePDFText() + extractAndSavePDFScreenshot()
         ↳ readPDFText() / fromBuffer(asset)
         └─→ saveAsset() + INSERT assets
    ↓
  入队 OpenAIQueue (tag + summarize)
  触发搜索索引重建
```

---

## 总结

### 设计优点

1. **分层缓存**: 浏览器 → React Query → 预处理结果 → 存储层，多级优化
2. **流式处理**: 大文件通过流传输，避免内存溢出
3. **队列解耦**: 耗时操作（OCR、视频下载）通过 Worker 队列异步处理
4. **安全可靠**: 严格的 CSP、沙箱 iframe、配额检查
5. **多后端支持**: 本地 FS 和 S3 统一接口

### 关注点

1. **前端资源释放**: iframe/video 元素缺少显式清理
2. **状态复杂度**: 多视图切换 + 多类型分支，维护成本较高
3. **错误边界**: 自定义渲染器有 ErrorBoundary，但基础视图缺少
4. **内存占用**: 大 PDF iframe 可能导致浏览器内存累积

### 关键文件速查表

| 文件 | 核心职责 |
|------|---------|
| `packages/shared/assetdb.ts` | 资产存储抽象 |
| `packages/api/utils/assets.ts` | HTTP 资源服务 |
| `apps/workers/workers/assetPreprocessingWorker.ts` | OCR + PDF 处理 |
| `apps/workers/workers/videoWorker.ts` | 视频下载 |
| `apps/web/components/dashboard/preview/BookmarkPreview.tsx` | 预览入口 |
| `apps/web/components/dashboard/preview/AssetContentSection.tsx` | 资产类型分支 |
| `apps/web/components/dashboard/preview/LinkContentSection.tsx` | 链接多视图 |
