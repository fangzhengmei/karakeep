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
| 视频处理 Worker | `apps/workers/workers/videoWorker.ts` | 视频下载（无转码） |
| 签名令牌 | `packages/shared/signedTokens.ts` | 公共资产访问的签名验证 |
| 鉴权中间件 | `packages/api/middlewares/auth.ts` | API 访问身份认证 |
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

## 4. 鉴权与访问控制

### 4.1 双轨访问机制

Karakeep 采用**受保护访问**和**签名访问**两种并行的资产访问机制：

```
                        资产访问请求
                             ↓
                ┌───────────────────────┐
                │ 判断访问路径          │
                └───────────────────────┘
                       ↓        ↓
           /api/assets/        /public/assets/
                ↓                      ↓
        authMiddleware         unauthedMiddleware
                ↓                      ↓
        用户已登录?            验证 signed token
            ↓   ↓                      ↓
           是   否              token 有效?
           ↓   ↓                   ↓   ↓
   权限检查   401                是   否
           ↓                       ↓   ↓
   serveAsset                serveAsset  403
```

### 4.2 受保护访问流程 (`/api/assets/{assetId}`)

**文件位置**: `packages/api/routes/assets.ts:13-50`

```typescript
// 中间件执行顺序
app.use(authMiddleware)  // 1. 强制用户认证
   .get("/:assetId", 
     apiKeyScopeMiddleware("assets", "read"),  // 2. API Key 权限检查
     async (c) => {
       const asset = await Asset.fromId(c.var.ctx, assetId);
       await asset.ensureCanView();  // 3. 业务权限检查
       return serveAsset(c, assetId, asset.asset.userId);
     }
   );
```

**权限检查层级** (`packages/trpc/models/assets.ts:225-260`):
1. **所有者检查**: `asset.userId === ctx.user.id` → 直接通过
2. **公开资产**: `assetType === "avatar"` → 直接通过
3. **关联书签权限**: 检查用户是否有权访问关联的 bookmark

### 4.3 签名访问流程 (`/public/assets/{assetId}`)

**文件位置**: `packages/api/routes/public/assets.ts:14-49`

用于公开分享场景，无需登录，但需要携带签名 token：

```typescript
// 1. 无需登录，但需要签名 token
app.get("/:assetId",
  unauthedMiddleware,  // 仅验证上下文存在，不要求用户登录
  zValidator("query", z.object({ token: z.string() })),
  async (c) => {
    // 2. 验证签名 token
    const tokenPayload = verifySignedToken(
      c.req.valid("query").token,
      serverConfig.signingSecret(),
      zAssetSignedTokenSchema,
    );
    
    // 3. token 绑定到特定 assetId，防止重用
    if (tokenPayload.assetId !== assetId) {
      return 403;
    }
    
    // 4. 验证资产存在性
    const assetDb = await db.query.assets.findFirst(...);
    
    return serveAsset(c, assetId, userId);
  }
);
```

### 4.4 签名令牌机制

**文件位置**: `packages/shared/signedTokens.ts`

```typescript
// Token 结构
{
  payload: {
    assetId: string,    // 绑定的资产 ID
    userId: string,     // 资产所有者
  },
  expiresAt: number,    // 过期时间戳
  signature: string,    // HMAC-SHA256 签名
}

// 过期策略
const expiresAt = getAlignedExpiry(
  3600,   // 1小时间隔对齐
  900,    // 15分钟宽限期
);
```

**签名 URL 生成** (`packages/trpc/models/assets.ts:266-281`):
```typescript
static getPublicSignedAssetUrl(assetId, assetOwnerId, expireAt) {
  const payload = { assetId, userId: assetOwnerId };
  const signedToken = createSignedToken(
    payload, 
    serverConfig.signingSecret(), 
    expireAt
  );
  return `${serverConfig.publicApiUrl}/public/assets/${assetId}?token=${signedToken}`;
}
```

### 4.5 访问路径选择逻辑

**签名 URL 生成时机** (`packages/trpc/models/bookmarks.ts:768-860`):

```
asPublicBookmark() 转换为公开视图时
    ↓
为每个资产生成带签名的 URL
    ↓
├─→ 资产内容 URL: assetUrl = getPublicSignedAssetUrl(assetId)
├─→ 横幅图片 URL: bannerImageUrl = getPublicSignedAssetUrl(bannerAssetId)
└─→ 截图 URL: screenshotUrl = getPublicSignedAssetUrl(screenshotAssetId)
```

**Token 有效期**:
- 普通列表查询: 10分钟 (`Date.now() + 10 * 60 * 1000`)
- 公开列表视图: 1小时 + 15分钟宽限期 (`getAlignedExpiry(3600, 900)`)

---

## 5. 视频处理流程（修正：仅下载，无转码）

**文件位置**: `apps/workers/workers/videoWorker.ts`

### 5.1 视频下载流程（无转码）

**重要修正**: 视频处理仅使用 `yt-dlp` 进行下载，**不执行任何格式转码或编码转换**。下载的视频格式由 yt-dlp 根据远程源自动选择。

```
视频链接
    ↓
URL 规范化 + 重定向验证
    ↓
yt-dlp 下载 (支持代理)
│  参数: -f "best[filesize<MAX_SIZE]M"
│        选择最佳可用格式，不进行转码
    ↓
临时文件: os.tmpdir()/video_downloads/{assetId}*
│  yt-dlp 自动添加文件扩展名（.mp4, .webm, .mkv 等）
    ↓
配额检查 → 保存到 AssetDB
│  contentType 硬编码为 "video/mp4"
│  （注意：实际格式可能不是 MP4，此处存在不匹配风险）
    ↓
更新数据库关联 → 删除旧资产
    ↓
清理临时文件
```

### 5.2 关键实现细节

```typescript
// packages/shared/assetdb.ts:37-41
// 支持的视频格式
export const VIDEO_ASSET_TYPES: Set<string> = new Set<string>([
  ASSET_TYPES.VIDEO_MP4,    // video/mp4
  ASSET_TYPES.VIDEO_WEBM,   // video/webm
  ASSET_TYPES.VIDEO_MKV,    // video/x-matroska
]);

// videoWorker.ts:206 - 硬编码为 MP4，但实际可能不是
metadata: { contentType: ASSET_TYPES.VIDEO_MP4 },
```

**潜在问题**: 
- 下载的视频格式取决于远程源，可能是 webm、mkv 等
- 但 contentType 被硬编码为 `video/mp4`
- 浏览器播放时依赖文件实际内容而非 Content-Type 头

### 5.3 资源释放关键点

1. **下载取消**: `execa("yt-dlp", { cancelSignal: job.abortSignal })` (L155-157)
2. **失败清理**: `deleteLeftOverAssetFile()` 在异常时调用 (L183, L234)
3. **旧资产清理**: `silentDeleteAsset(userId, oldVideoAssetId)` (L224)

---

## 6. 前端预览状态机

### 6.1 BookmarkPreview 主状态机

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

### 6.2 AssetContentSection 分支逻辑

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

### 6.3 LinkContentSection 多视图状态机

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

### 6.4 ReaderView 阅读状态机

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

## 7. HTTP 资源服务层

**文件位置**: `packages/api/utils/assets.ts:12-75`

### 7.1 服务流程

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

### 7.2 安全策略

**内容安全策略** (CSP):
```
sandbox                          ; 沙箱模式
default-src 'none'               ; 默认禁止所有
img-src https: data: blob:       ; 图片来源
style-src 'unsafe-inline' ...    ; 样式来源
media-src https: data: blob:     ; 媒体来源
```

---

## 8. 资源释放机制

### 8.1 服务器端资源释放

| 资源类型 | 释放位置 | 释放方式 |
|---------|---------|---------|
| Tesseract Worker | `assetPreprocessingWorker.ts:122` | `finally { worker.terminate() }` |
| 文件句柄 | `assetdb.ts:229-243` | `finally { fd.close() }` |
| 临时视频文件 | `videoWorker.ts:247-271` | `fs.promises.rm()` |
| 旧版本资产 | `videoWorker.ts:224` | `silentDeleteAsset()` |
| S3 连接 | S3Client 内部 | 连接池管理 |

### 8.2 客户端资源释放

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

## 9. 缓存层级设计

### 9.1 多层缓存策略

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

### 9.2 轮询触发机制与刷新节奏

#### 9.2.1 轮询触发位置

轮询仅在两个组件中通过 `refetchInterval` 动态计算激活：
1. `BookmarkPreview.tsx:148-154` - 预览弹窗/页面
2. `BookmarkCard.tsx:29-35` - 列表中的单个书签卡片

#### 9.2.2 刷新触发条件（三层判断）

**第一层: React Query 回调判断**
```typescript
// BookmarkPreview.tsx:148-154
refetchInterval: (query) => {
  const data = query.state.data;
  if (!data) {
    return false;  // 无数据时不轮询
  }
  return getBookmarkRefreshInterval(data);
}
```

**第二层: 加载状态总判断** (`packages/shared/utils/bookmarkUtils.ts:48-54`)
```typescript
function isBookmarkStillLoading(bookmark) {
  return (
    isBookmarkStillTagging(bookmark) ||      // taggingStatus == "pending"
    isBookmarkStillCrawling(bookmark) ||     // crawlStatus == "pending" 或 !crawledAt
    isBookmarkStillSummarizing(bookmark)     // summarizationStatus == "pending"
  );
}
```

**第三层: 各子状态详细判断**

| 条件 | 触发刷新 | 说明 |
|------|---------|------|
| `taggingStatus === "pending"` | ✅ 是 | 标签生成中 |
| `summarizationStatus === "pending"` | ✅ 是 | 摘要生成中 |
| `crawlStatus === "pending"` | ✅ 是 | 网页抓取中 |
| `!crawledAt && type === LINK` | ✅ 是 | 从未抓取过的链接 |
| 以上都不满足 | ❌ 否 | 所有处理完成，停止轮询 |

#### 9.2.3 渐进式退避刷新策略

**文件位置**: `packages/shared/utils/bookmarkUtils.ts:56-83`

```typescript
export function getBookmarkRefreshInterval(
  bookmark: ZBookmark,
): number | false {
  // 1. 非加载状态: 立即返回 false，停止刷新
  if (!isBookmarkStillLoading(bookmark)) {
    return false;
  }

  // 2. 前30秒: 每秒刷新一次（高实时性）
  if (Date.now() - bookmark.createdAt < 30 * 1000) {
    return 1000;  // 1秒
  }

  // 3. 30秒 ~ 10分钟: 每10秒刷新一次
  if (Date.now() - bookmark.createdAt < 10 * 60 * 1000) {
    return 10_000;  // 10秒
  }

  // 4. 10分钟 ~ 6小时: 每分钟刷新一次
  if (Date.now() - bookmark.createdAt < 6 * 60 * 60 * 1000) {
    return 60_000;  // 60秒
  }

  // 5. 超过6小时: 强制停止刷新
  return false;
}
```

#### 9.2.4 实际轮询节奏示例

```
用户创建书签 (t=0)
    ↓
t=0~30秒: 每秒刷新一次 (共约30次请求)
    ↓
t=30秒~10分钟: 每10秒刷新一次 (共约57次请求)
    ↓
t=10分钟~6小时: 每分钟刷新一次 (共约350次请求)
    ↓
t>6小时: 停止刷新
    ↓
总计: 约437次请求 / 书签 (完整加载周期)
```

**关键发现**:
- 时间基准是 `bookmark.createdAt`（书签创建时间），**不是**组件挂载时间
- 即使组件在 t=5分钟 时才挂载，也会立即进入 10秒 间隔的轮询阶段
- 书签创建超过6小时后，无论处理状态如何，**强制停止轮询**

#### 9.2.5 轮询生命周期边界

| 场景 | 轮询行为 |
|------|---------|
| 组件首次挂载 | 立即触发一次查询，然后按间隔轮询 |
| 组件卸载 | React Query 自动清理定时器 |
| 用户关闭预览弹窗 | Dialog 关闭 → 组件卸载 → 轮询停止 |
| 切换到其他书签 | bookmarkId 变化 → 旧查询取消 → 新查询开始 |
| 处理完成 (所有状态非 pending) | getBookmarkRefreshInterval 返回 false → 轮询停止 |
| 公开页面 PublicBookmarkGrid | 无轮询机制，数据静态 |

### 9.3 React Query 缓存配置

**文件位置**: `apps/web/lib/providers.tsx:23-33`

```typescript
// 全局默认配置
new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,  // 数据60秒后视为陈旧
    },
  },
});
```

**缓存层级总结**:
1. **React Query 内存缓存**: staleTime = 60秒
2. **轮询刷新**: 1秒 → 10秒 → 60秒 渐进退避（仅加载状态的组件）
3. **浏览器 HTTP 缓存**: Cache-Control: max-age=31536000 (1年)
4. **预处理结果缓存**: 数据库持久化，fixMode 时跳过重复处理

### 9.4 缓存失效机制

1. **浏览器缓存**: 基于 `immutable` 标志，资产 ID 变化时自动失效
2. **React Query**: 通过 `refetchInterval` 轮询更新（仅正在加载的 bookmark）
3. **预处理缓存**: 通过 `fixMode` 参数 + 数据库字段检查 (`alreadyHasText` / `alreadyHasScreenshot`)

---

## 10. LinkContentSection 视图恢复、路径界限与降级路径分析

### 10.1 查询参数恢复机制（仅受保护路径）

**文件位置**: `apps/web/components/dashboard/preview/LinkContentSection.tsx:126-130`

```typescript
const defaultSection =
  availableRenderers.length > 0 ? availableRenderers[0].id : "cached";
const [section, setSection] = useQueryState("section", {
  defaultValue: defaultSection,
});
```

**适用范围界限**:
| 路径类型 | URL 模式 | 是否支持 ?section= | 组件 |
|---------|---------|-------------------|------|
| ✅ 受保护预览页面 | `/dashboard/preview/[bookmarkId]` | 是 | `LinkContentSection` |
| ✅ 受保护预览弹窗 | `/dashboard/@modal/(.)preview/[bookmarkId]` | 是 | `LinkContentSection` |
| ❌ 受保护列表页面 | `/dashboard/bookmarks` | 否 | 无预览组件 |
| ❌ 公开列表页面 | `/public/lists/[listId]` | 否 | `PublicBookmarkGrid` (仅卡片) |
| ❌ 公开单个书签 | 不存在此路由 | - | - |

**视图恢复流程**:
```
页面加载 / 书签切换
    ↓
从 URL query 读取 ?section=xxx 参数
    ↓
参数存在且有效?
    ├─→ 是 → 使用该值作为当前 section
    └─→ 否 → 使用 defaultSection
              ├─→ 有自定义渲染器? → 第一个自定义渲染器
              └─→ 无 → "cached"
```

**nuqs useQueryState 关键行为**:
- 参数缺失时返回 `defaultValue`
- **不自动验证**参数值的有效性（如对应资产是否存在）
- `defaultValue` 不会同步回 URL
- 值变化时自动更新 URL query 参数

### 10.2 受保护路径 vs 公开路径：核心差异与界限

#### 10.2.1 数据结构与访问路径界限

| 维度 | 受保护路径 (`ZBookmark`) | 公开路径 (`ZPublicBookmark`) | 界限说明 |
|------|-------------------------|----------------------------|---------|
| 资产引用方式 | 存储 `assetId` (字符串) | 存储 `assetUrl` (完整URL) | 受保护路径前端拼接，公开路径后端生成 |
| 资产访问路径 | `/api/assets/{assetId}` | `/public/assets/{id}?token=xxx` | 两条完全独立的路由 |
| 鉴权方式 | Cookie + Session | Signed Token (HMAC) | 互斥，不能混用 |
| 多视图支持 | ✅ 6种视图切换 | ❌ 仅卡片展示 | 功能边界完全不同 |
| URL 参数恢复 | ✅ `?section=` 有效 | ❌ 参数被忽略 | 公开页面无解析逻辑 |
| 轮询刷新 | ✅ 渐进式退避 | ❌ 静态数据 | 公开页面无轮询机制 |

#### 10.2.2 资产 URL 生成路径界限

**受保护路径（前端拼接）** (`packages/shared/utils/assetUtils.ts:1-3`):
```typescript
export function getAssetUrl(assetId: string) {
  return `/api/assets/${assetId}`;  // 依赖 Cookie 鉴权，无签名
}
```
→ **界限**: 仅在登录用户的 dashboard 上下文中有效

**公开路径（服务端签名）** (`packages/trpc/models/bookmarks.ts:768-860`):
```typescript
// 服务端生成签名 URL，绑定 userId + assetId + 过期时间
const getPublicSignedAssetUrl = (assetId: string) => {
  return Asset.getPublicSignedAssetUrl(
    assetId,
    this.bookmark.userId,
    getAlignedExpiry(3600, 900),  // 1小时 + 15分钟宽限期
  );
};
// 返回格式: /public/assets/{assetId}?token={signed_jwt}
```
→ **界限**: 仅在服务端生成，前端直接使用，不能修改

#### 10.2.3 预览功能流程界限

**受保护预览完整流程**:
```
用户点击书签 (dashboard内)
    ↓
导航到 /dashboard/preview/[bookmarkId]
    ↓
BookmarkPreview 挂载
    ↓
useQuery 加载 bookmark (含轮询)
    ↓
LinkContentSection 渲染
    ↓
useQueryState 解析 ?section= 参数
    ↓
根据 section 选择子组件
    ├─→ cached → ReaderView (重新查询 htmlContent)
    ├─→ screenshot → ScreenshotSection (/api/assets/xxx)
    ├─→ archive → FullPageArchiveSection (/api/assets/xxx)
    ├─→ video → VideoSection (/api/assets/xxx)
    └─→ pdf → PDFSection (/api/assets/xxx)
    ↓
浏览器请求 /api/assets/xxx (带 Cookie)
    ↓
authMiddleware 验证 Session
    ↓
Asset.ensureCanView() 检查权限
    ↓
返回资产内容
```

**公开页面流程（无预览组件）**:
```
用户访问 /public/lists/[listId]
    ↓
服务端 RSC 获取 ZPublicBookmark 列表
    ↓
PublicBookmarkGrid 渲染卡片
    ↓
用户点击资产链接
    ↓
新标签页打开 assetUrl (/public/assets/xxx?token=xxx)
    ↓
浏览器请求（无 Cookie）
    ↓
verifySignedToken 验证签名
    ↓
检查 token.assetId === 路径 assetId
    ↓
返回资产内容
```

### 10.3 路径冲突点与失败降级路径

#### 10.3.1 冲突点1: LinkContentSection 硬编码受保护路径

**代码位置**: `apps/web/components/dashboard/preview/LinkContentSection.tsx:65-116`

```typescript
// 所有子组件全部硬编码 /api/assets/ 路径
function FullPageArchiveSection({ link }) {
  const archiveAssetId = link.fullPageArchiveAssetId ?? link.precrawledArchiveAssetId;
  return <iframe src={`/api/assets/${archiveAssetId}`} />;  // 受保护路径
}

function VideoSection({ link }) {
  return <video><source src={`/api/assets/${link.videoAssetId}`} /></video>;
}

function ScreenshotSection({ link }) {
  return <Image src={`/api/assets/${link.screenshotAssetId}`} />;
}

function PDFSection({ link }) {
  return <iframe src={`/api/assets/${link.pdfAssetId}`} />;
}
```

**冲突场景：在公开页面误用组件**
```
尝试在 /public/lists/[listId] 中使用 LinkContentSection
    ↓
组件渲染 VideoSection
    ↓
请求 /api/assets/xxx (无 Cookie)
    ↓
authMiddleware 返回 401 Unauthorized
    ↓
video 元素显示加载失败
    ↓
❌ 无降级处理，仅显示浏览器默认错误
```

**降级路径：无自动降级**
- 组件不会检测 401/403 错误
- 不会自动切换到其他视图
- 不会提示用户登录

#### 10.3.2 冲突点2: URL 参数恢复与资产存在性校验缺失

**问题**: `useQueryState` 直接使用 URL 参数值，不校验资产是否存在

```typescript
// LinkContentSection.tsx:128-130
const [section, setSection] = useQueryState("section", {
  defaultValue: defaultSection,
});
// ⚠️ 没有检查: section 对应的 assetId 是否存在
```

**失败场景：URL 指定无效视图**
```
用户访问 ?bookmarkId=123&section=video
    ↓
bookmark.videoAssetId = null (视频资产不存在)
    ↓
section = "video" (从 URL 恢复)
    ↓
SelectItem video 被 disabled (UI 禁用)
    ↓
SelectValue 仍显示 "video" (无效值)
    ↓
VideoSection 渲染
    ↓
<source src={`/api/assets/null`}>
    ↓
GET /api/assets/null → 404 Not Found
    ↓
❌ 无降级，video 显示空白
```

**降级路径：无自动回退**
- Select 组件显示无效值（Radix UI 行为未定义）
- 不会自动回退到 defaultSection
- 用户需要手动选择其他视图

#### 10.3.3 冲突点3: ReaderView 内容加载失败降级

**ReaderView 有降级处理** (`apps/web/components/dashboard/preview/ReaderView.tsx:112-133`):
```typescript
const { data: cachedContent, isPending: isCachedContentLoading } = useQuery(
  api.bookmarks.getBookmark.queryOptions(
    { bookmarkId, includeContent: true },
    { select: (data) => data.content.htmlContent ?? null }
  )
);

if (isCachedContentLoading) {
  content = <FullPageSpinner />;                    // 降级1: 加载中显示 spinner
} else if (!cachedContent) {
  content = <FileX 图标 + 错误提示>;               // 降级2: 无内容显示友好错误
} else {
  content = <BookmarkHTMLHighlighter />;           // 正常渲染
}
```
✅ **ReaderView 是唯一有完整降级路径的视图**

#### 10.3.4 冲突点4: 自定义渲染器 ErrorBoundary 降级

**自定义渲染器有降级** (`LinkContentSection.tsx:144-148`):
```typescript
<ErrorBoundary FallbackComponent={CustomRendererErrorFallback}>
  <RendererComponent bookmark={bookmark} />
</ErrorBoundary>
```

**降级组件** (`LinkContentSection.tsx:45-63`):
```typescript
function CustomRendererErrorFallback({ error }) {
  return (
    <Alert variant="destructive">
      <AlertTitle>Renderer Error</AlertTitle>
      <AlertDescription>
        Failed to load custom content renderer.
        <details><code>{error.message}</code></details>
      </AlertDescription>
    </Alert>
  );
}
```
✅ **自定义渲染器有 ErrorBoundary 降级**

#### 10.3.5 其他视图：无降级路径

| 视图组件 | 错误场景 | 降级行为 |
|---------|---------|---------|
| ScreenshotSection | assetId = null | 请求 `/api/assets/null` → 404 → Next.js Image 报错 |
| VideoSection | assetId = null | 请求 `/api/assets/null` → 404 → 显示 "Not supported" |
| FullPageArchiveSection | assetId = null | 请求 `/api/assets/null` → 404 → iframe 空白 |
| PDFSection | assetId = null | 请求 `/api/assets/null` → 404 → iframe 空白 |
| 以上所有 | 签名 token 过期 | 403 → 浏览器默认错误 |

### 10.4 禁用项与资产缺失检测

**文件位置**: `apps/web/components/dashboard/preview/LinkContentSection.tsx:206-240`

```typescript
<SelectItem
  value="screenshot"
  disabled={!bookmark.content.screenshotAssetId}
>
<SelectItem
  value="pdf"
  disabled={!bookmark.content.pdfAssetId}
>
<SelectItem
  value="archive"
  disabled={
    !bookmark.content.fullPageArchiveAssetId &&
    !bookmark.content.precrawledArchiveAssetId
  }
>
<SelectItem
  value="video"
  disabled={!bookmark.content.videoAssetId}
>
```

**禁用逻辑**:
| 视图 | 启用条件 |
|------|---------|
| screenshot | `screenshotAssetId != null` |
| pdf | `pdfAssetId != null` |
| archive | `fullPageArchiveAssetId != null || precrawledArchiveAssetId != null` |
| video | `videoAssetId != null` |
| cached | 始终启用（兜底视图） |
| 自定义渲染器 | 由渲染器注册表决定 |

### 10.3 无效参数场景：资产缺失

**场景**: URL 中 `?section=video` 但 `videoAssetId` 为 null

```
时间线:
t0: 用户访问 ?bookmarkId=123&section=video
    ↓
t1: useQueryState 读取 section = "video"
    ↓
t2: 渲染 Select 组件，value = "video"
    ↓
t3: SelectItem video 被 disabled (因 assetId 缺失)
    ↓
t4: SelectValue 显示 "video" (但选项不可选)
    ↓
t5: 渲染 VideoSection
    ↓
t6: <source src={`/api/assets/null`}> ❌ 无效请求
```

**问题**:
1. Select 显示无效值（已被 disabled 的选项）
2. Radix UI Select 对无效 value 的行为未定义
3. 可能导致渲染异常或静默失败

**资产访问路径 (缺失时)**:
```typescript
// FullPageArchiveSection - assetId 为 null 时
src={`/api/assets/${null}`}
// → GET /api/assets/null → 404 或 500

// VideoSection - assetId 为 null 时
<source src={`/api/assets/${null}`} />
// → GET /api/assets/null → 404

// ScreenshotSection - assetId 为 null 时
src={`/api/assets/${undefined}`}
// → Next.js Image 组件报错
```

### 10.4 公开视图场景：签名令牌过期

**文件位置**: `packages/trpc/models/bookmarks.ts:768-860`

公开列表页面的资产 URL 包含签名 token:
```typescript
const getPublicSignedAssetUrl = (assetId: string) => {
  return Asset.getPublicSignedAssetUrl(
    assetId,
    this.bookmark.userId,
    getAlignedExpiry(3600, 900),  // 1小时 + 15分钟宽限期
  );
};
```

**令牌过期流程**:
```
t0: 公开列表页面加载
    ↓
生成签名 URL: /public/assets/abc?token=xxx (有效期1小时)
    ↓
t1: 用户停留 70 分钟
    ↓
t2: 用户切换到 ?section=video
    ↓
t3: VideoSection 渲染
    ↓
t4: <source src="/public/assets/abc?token=xxx">
    ↓
t5: 浏览器发起请求
    ↓
t6: 服务器验证 token → 已过期 → 返回 403
    ↓
t7: video 元素显示 "Not supported by your browser" ❌ 误导性提示
```

**错误请求路径**:
```
GET /public/assets/{assetId}?token={expired_token}
    ↓
verifySignedToken() → null (过期)
    ↓
返回 403 { error: "Invalid or expired token" }
    ↓
浏览器: video 元素加载失败 → 显示备用文本
```

### 10.5 降级处理机制分析

**当前实现的降级策略**:

| 场景 | 降级行为 | 问题 |
|------|---------|------|
| 自定义渲染器出错 | ErrorBoundary → 显示错误提示 | ✅ 有处理 |
| ReaderView 无内容 | FileX 图标 + 错误文案 | ✅ 有处理 |
| 资产 ID 为 null | 直接请求 `/api/assets/null` | ❌ 无降级 |
| 签名 token 过期 | 资源 403 → 浏览器默认错误 | ❌ 无降级 |
| 资源加载失败 (iframe/image) | 浏览器默认错误状态 | ❌ 无降级 |

**自定义渲染器的 ErrorBoundary**:
```typescript
// LinkContentSection.tsx:144-148
<ErrorBoundary FallbackComponent={CustomRendererErrorFallback}>
  <RendererComponent bookmark={bookmark} />
</ErrorBoundary>
```

**ReaderView 的错误处理**:
```typescript
// ReaderView.tsx:114-133
if (!cachedContent) {
  content = (
    <div className="flex h-full w-full items-center justify-center p-4">
      <FileX className="h-8 w-8 text-muted-foreground" />
      <h3>{t("preview.fetch_error_title")}</h3>
      <p>{t("preview.fetch_error_description")}</p>
    </div>
  );
}
```

### 10.6 建议的降级改进方案

**方案1: 参数有效性校验 + 自动回退**
```typescript
// LinkContentSection 中添加
const availableSections = useMemo(() => {
  const sections: string[] = [...availableRenderers.map(r => r.id), "cached"];
  if (bookmark.content.screenshotAssetId) sections.push("screenshot");
  if (bookmark.content.pdfAssetId) sections.push("pdf");
  if (bookmark.content.fullPageArchiveAssetId || 
      bookmark.content.precrawledArchiveAssetId) sections.push("archive");
  if (bookmark.content.videoAssetId) sections.push("video");
  return sections;
}, [bookmark, availableRenderers]);

// 校验并修正 section
useEffect(() => {
  if (!availableSections.includes(section)) {
    setSection(defaultSection);  // 自动回退到默认视图
  }
}, [section, availableSections, defaultSection, setSection]);
```

**方案2: 资源加载错误捕获**
```typescript
// VideoSection 改进
function VideoSection({ link }: { link: ZBookmarkedLink }) {
  const [hasError, setHasError] = useState(false);
  
  if (hasError) {
    return (
      <div className="flex h-full items-center justify-center">
        <Alert variant="destructive">
          <AlertTitle>视频加载失败</AlertTitle>
          <AlertDescription>
            视频资源无法访问，请尝试其他视图
          </AlertDescription>
        </Alert>
      </div>
    );
  }
  
  return (
    <video controls onError={() => setHasError(true)}>
      <source src={`/api/assets/${link.videoAssetId}`} />
    </video>
  );
}
```

**方案3: 公开视图令牌刷新**
```typescript
// 检测到 403 时触发刷新获取新 token
const refreshPublicBookmark = useMutation({
  mutationFn: () => api.publicBookmarks.getPublicBookmark.refetch(),
  onSuccess: (data) => {
    // 更新 bookmark 数据，包含新的签名 URL
  },
});
```

---

## 11. 类型分支汇总

### 11.1 Bookmark 类型分支

| 类型 | 预览方式 | 预处理 |
|------|---------|--------|
| LINK | ReaderView / Screenshot / Archive / Video / PDF / 自定义渲染器 | 网页抓取 + 可选视频下载 |
| TEXT | TextContentSection (Markdown) | 无 |
| ASSET (image) | ImageContentSection | OCR 文本提取 |
| ASSET (pdf) | PDF iframe / Screenshot | 文本提取 + 首页截图 |

### 11.2 资产类型枚举

**文件位置**: `packages/shared/assetdb.ts:23-70`

- **图片**: `image/gif`, `image/jpeg`, `image/png`, `image/webp`
- **文档**: `application/pdf`
- **视频**: `video/mp4`, `video/webm`, `video/x-matroska`
- **网页**: `text/html`
- **归档**: `application/zip`

---

## 12. 关键代码路径

### 12.1 打开附件预览的完整调用链

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

### 12.2 预处理调用链

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

## 13. PDF 与截图视图状态一致性分析

### 13.1 初始状态选择逻辑

**文件位置**: `apps/web/components/dashboard/preview/AssetContentSection.tsx:20-41`

```typescript
const initialSection = useMemo(() => {
  const screenshot = bookmark.assets.find(
    (item) => item.assetType === "assetScreenshot",
  );
  const bigSize =
    bookmark.content.size && bookmark.content.size > BIG_FILE_SIZE;
  if (bigSize && screenshot) {
    return "screenshot";
  }
  return "pdf";
}, [bookmark]);

const [section, setSection] = useState(initialSection);
```

**状态初始化时机**:
- `useMemo` 仅在 `bookmark` 引用变化时重新计算
- `useState(initialSection)` 仅在组件**首次挂载**时使用初始值
- 后续 `bookmark` 数据更新**不会**自动改变 `section` 状态

### 13.2 数据刷新机制

**文件位置**: `apps/web/components/dashboard/preview/BookmarkPreview.tsx:141-156`

```typescript
const { data: bookmark } = useQuery(
  api.bookmarks.getBookmark.queryOptions(
    { bookmarkId },
    {
      initialData,
      refetchInterval: (query) => {
        const data = query.state.data;
        if (!data) return false;
        return getBookmarkRefreshInterval(data);
      },
    }
  ),
);
```

**刷新触发条件** (`packages/shared/utils/bookmarkUtils.ts`):
- 正在抓取的 bookmark: 2秒轮询
- 预处理中 (taggingStatus/summarizationStatus === "pending"): 2秒轮询
- 其他情况: 不自动刷新

### 13.3 状态不一致场景分析

#### 场景1: 截图生成完成后视图不自动切换

```
时间线:
t0: 用户打开 PDF (文件 > 20MB, 无截图)
    → initialSection = "pdf"
    → section = "pdf" (显示 iframe)

t1: 后台 Worker 完成 PDF 截图生成
    → bookmark.assets 新增 assetScreenshot

t2: React Query 轮询获取更新后的 bookmark
    → bookmark 引用变化
    → useMemo 重新计算: initialSection = "screenshot"
    → ⚠️  section 仍为 "pdf" (useState 不更新)
```

**问题**: 截图已生成，但用户仍看到 PDF iframe，不会自动切换到轻量的截图视图。

#### 场景2: 用户手动切换后数据刷新

```
t0: 大 PDF 已有截图
    → initialSection = "screenshot"
    → section = "screenshot"

t1: 用户手动切换到 "pdf" 视图
    → section = "pdf"

t2: 其他字段更新触发 bookmark 刷新
    → bookmark 引用变化
    → useMemo 重新计算: initialSection = "screenshot"
    → ⚠️  section 仍为 "pdf" (覆盖用户选择)
```

**问题**: 用户的手动选择在数据刷新后被保留，但初始逻辑意图可能已失效。

#### 场景3: 预处理中打开预览

```
t0: PDF 上传完成，开始预处理
    → bookmark.content.size 可能为 null
    → initialSection = "pdf" (因 size 判断失败)
    → section = "pdf"

t1: 预处理完成，size 更新为 25MB，截图生成
    → bookmark 刷新
    → initialSection = "screenshot"
    → ⚠️  section 仍为 "pdf"
```

**问题**: 预处理完成后，大小信息可用，但视图策略不会自动调整。

### 13.4 状态同步实现方案对比

| 方案 | 实现方式 | 优点 | 缺点 |
|------|---------|------|------|
| **当前实现** | `useState(initialSection)` + 手动 `setSection` | 尊重用户选择，不自动切换 | 新截图生成后不更新视图 |
| **useDerivedState** | `useEffect(() => setSection(initialSection), [initialSection])` | 数据变化时自动同步 | 可能覆盖用户手动选择 |
| **条件同步** | 仅在用户未手动切换时自动更新 | 平衡自动同步和用户选择 | 实现复杂度增加 |
| **key 重置** | 给组件添加 `key={bookmark.id}` | bookmark 变化时完全重置状态 | 会丢失所有本地状态 |

### 13.5 建议的改进方案

```typescript
// 方案: 跟踪用户是否手动切换过
const [section, setSection] = useState(initialSection);
const [userManuallyChanged, setUserManuallyChanged] = useState(false);

// 仅在用户未手动切换时，自动同步新的初始状态
useEffect(() => {
  if (!userManuallyChanged) {
    setSection(initialSection);
  }
}, [initialSection, userManuallyChanged]);

// 用户手动切换时标记
const handleSectionChange = (newSection: string) => {
  setUserManuallyChanged(true);
  setSection(newSection);
};
```

### 13.6 公开视图中的签名 URL 过期问题

**文件位置**: `packages/trpc/models/bookmarks.ts:768-776`

```typescript
const getPublicSignedAssetUrl = (assetId: string) => {
  return Asset.getPublicSignedAssetUrl(
    assetId,
    this.bookmark.userId,
    getAlignedExpiry(3600, 900),  // 1小时 + 15分钟宽限期
  );
};
```

**状态不一致风险**:
1. 公开页面加载时生成签名 URL（有效期1小时）
2. 用户停留超过1小时后切换到 PDF 视图
3. iframe 请求 `/public/assets/{id}?token=...`
4. token 已过期 → 403 错误
5. 页面无刷新机制 → URL 不会自动更新

**缓解方案**:
- 缩短过期时间但增加刷新频率
- 或在前端检测到 403 时触发数据刷新获取新 token

---

## 总结

### 设计优点

1. **双轨鉴权**: 受保护访问（登录用户）+ 签名访问（公开分享），灵活覆盖多种场景
2. **渐进式刷新**: 1秒 → 10秒 → 60秒 渐进退避策略，平衡实时性与性能
3. **分层缓存**: 浏览器 → React Query → 预处理结果 → 存储层，多级优化
4. **流式处理**: 大文件通过流传输，避免内存溢出
5. **队列解耦**: 耗时操作（OCR、视频下载）通过 Worker 队列异步处理
6. **安全可靠**: 严格的 CSP、沙箱 iframe、配额检查
7. **多后端支持**: 本地 FS 和 S3 统一接口
8. **签名令牌**: HMAC-SHA256 签名 + 过期时间对齐，安全且高效
9. **URL 状态持久化**: 使用 nuqs 将视图状态同步到 URL query 参数

### 关注点

1. **前端资源释放**: iframe/video 元素缺少显式清理
2. **视图状态一致性**:
   - PDF 与截图视图在数据刷新后不会自动同步
   - URL 参数恢复视图时不校验资产是否存在
3. **路径耦合界限**:
   - `LinkContentSection` 硬编码 `/api/assets/` 路径
   - 受保护/公开路径完全独立，组件不能跨路径复用
   - 误用会导致 401/403 错误，无降级处理
4. **状态复杂度**: 多视图切换 + 多类型分支，维护成本较高
5. **视频格式**: 视频下载无转码，ContentType 硬编码可能与实际格式不匹配
6. **降级处理不完善（分层）**:
   - ✅ ReaderView: 完整降级（加载中 → 无内容）
   - ✅ 自定义渲染器: ErrorBoundary 降级
   - ❌ Screenshot/Video/Archive/PDF: 无降级，直接请求无效 URL
   - ❌ 签名 token 过期: 显示浏览器默认错误
7. **内存占用**: 大 PDF iframe 可能导致浏览器内存累积
8. **签名 URL 过期**: 公开页面签名 URL 1小时过期，长时间停留后访问失败
9. **无效参数处理**: Select 可能显示已 disabled 的值，Radix UI 行为未定义
10. **轮询成本**:
    - 时间基准是 `bookmark.createdAt`（不是组件挂载时间）
    - 完整加载周期约 437 次请求/书签
    - 6小时后强制停止，即使处理未完成
11. **轮询触发条件**: 三层判断（数据存在 → 加载状态 → 时间退避）

### 关键文件速查表

| 文件 | 核心职责 |
|------|---------|
| `packages/shared/assetdb.ts` | 资产存储抽象 |
| `packages/shared/signedTokens.ts` | 签名令牌生成与验证 |
| `packages/shared/utils/assetUtils.ts` | 受保护资产 URL 生成 |
| `packages/shared/utils/bookmarkUtils.ts` | 刷新时间策略 + 加载状态判断 |
| `packages/api/middlewares/auth.ts` | 鉴权中间件 |
| `packages/api/routes/assets.ts` | 受保护资产路由 |
| `packages/api/routes/public/assets.ts` | 公开签名资产路由 |
| `packages/api/utils/assets.ts` | HTTP 资源服务 |
| `apps/workers/workers/assetPreprocessingWorker.ts` | OCR + PDF 处理 |
| `apps/workers/workers/videoWorker.ts` | 视频下载（无转码） |
| `apps/web/lib/providers.tsx` | React Query 全局配置 |
| `apps/web/components/dashboard/preview/BookmarkPreview.tsx` | 预览入口 + 轮询刷新 |
| `apps/web/components/dashboard/preview/AssetContentSection.tsx` | 资产类型分支 |
| `apps/web/components/dashboard/preview/LinkContentSection.tsx` | 链接多视图 + URL 参数恢复 |
| `apps/web/components/public/lists/PublicBookmarkGrid.tsx` | 公开列表展示（无预览） |
| `packages/trpc/models/assets.ts` | 资产模型与权限检查 |
| `packages/trpc/models/bookmarks.ts` | 书签模型与公开视图转换 |
