# 媒体抓取与预览系统技术分析

本文档详细分析 Karakeep 项目中视频和嵌入媒体抓取的多层处理流程，涵盖 oEmbed、缩略图、metadata 标准化、失败重试、本地缓存与前端展示的协作机制。

---

## 1. 整体架构概览

媒体抓取与预览系统采用 **异步 Worker 队列 + 分层处理管道** 架构，从后端抓取到前端展示经历以下核心阶段：

```
用户提交书签
    │
    ▼
LinkCrawlerQueue (高优先级) / LowPriorityCrawlerQueue (低优先级)
    │
    ▼
CrawlerWorker ──────────────────────────────────────────┐
  ├─ 网络请求层 (network.ts) 代理/SSRF防护/DNS缓存      │
  ├─ 浏览器自动化 (Playwright) 页面渲染/截图/PDF         │
  ├─ 解析子进程 (parseHtmlSubprocess.ts)                │
  │   └─ metascraper + 插件 (oEmbed/OG/Reddit/Amazon)  │
  └─ 资产存储 (assetdb.ts) 图片/视频/PDF/HTML 归档       │
                                                        │
    ┌───────────────────────────────────────────────────┘
    ▼
下游队列级联触发:
  ├─ VideoWorkerQueue → VideoWorker (yt-dlp 视频下载)
  ├─ AssetPreprocessingQueue → AssetPreprocessingWorker (OCR/文本提取)
  ├─ EmbeddingsQueue / OpenAIQueue (AI 索引)
  ├─ SearchIndexingQueue (搜索索引)
  └─ WebhookQueue (Webhook 通知)
    │
    ▼
前端 BookmarkPreview 组件
  ├─ 轮询刷新机制 (渐进式回退策略)
  ├─ ContentRenderer 注册表 (YouTube/X/Amazon/Instagram/TikTok)
  └─ 多视图切换: Reader / Screenshot / PDF / Archive / Video
```

---

## 2. oEmbed 与平台特化元数据提取

Karakeep **不直接实现 oEmbed 协议客户端**，而是通过 `metascraper` 生态 + 自研插件实现等效的平台嵌入信息提取。

### 2.1 metascraper 插件栈

解析子进程 [parseHtmlSubprocess.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/scripts/parseHtmlSubprocess.ts#L36-L64) 中注册的完整插件链：

| 插件 | 作用 | 优先级 |
|------|------|--------|
| `metascraper-date` | 提取 `datePublished` / `dateModified` (支持 OG/JSON-LD) | 最早运行 |
| `metascraper-amazon-improved` (自研) | 修正 Amazon 商品标题/图片 | 先于官方插件 |
| `metascraper-amazon` | 官方 Amazon 商品数据提取 | - |
| `metascraper-youtube` | **YouTube oEmbed 等效**: 通过 `youtube.com/oembed` 端点提取标题、作者、缩略图 | 支持代理 |
| `metascraper-reddit` (自研) | Reddit JSON API + DOM 双路径提取 | 见 2.2 |
| `metascraper-author` | 从 `<meta name="author">`、JSON-LD 提取作者 | - |
| `metascraper-publisher` | 提取发布者 (og:site_name 等) | - |
| `metascraper-title` | 多级降级: og:title → twitter:title → `<title>` | - |
| `metascraper-description` | og:description → meta description | - |
| `metascraper-x` | X (Twitter) 卡片元数据提取 | - |
| `metascraper-image` | og:image → twitter:image → DOM 第一张图 | 见 2.3 |
| `metascraper-safe-favicon` (自研) | 安全的 favicon/logo 提取 | - |
| `metascraper-url` | 规范化 canonical URL | - |

### 2.2 Reddit 自研插件: 双层缓存 + API/DOM 双通道

[metascraper-reddit.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/metascraper-plugins/metascraper-reddit.ts#L83-L296) 是 oEmbed 思想的典型实现范例:

**内存级请求合并缓存** (`redditJsonCache`):
- TTL: 60 秒，防止短时间内重复请求 Reddit API
- 存储结构: `Map<url, { expiresAt, promise }>`，支持**在途请求合并** (避免缓存击穿)
- 每次访问前执行 `purgeExpiredCacheEntries` 惰性淘汰

**双通道提取策略**:
1. **API 通道** (优先): URL 追加 `.json` → `fetchWithProxy` → Zod Schema 解析
   - 提取字段: `title`、`preview.images[].source.url`、`media_metadata`、`author`、`created_utc`、`subreddit_name_prefixed`、`selftext_html`
2. **DOM 通道** (API 失败回退): Cheerio 直接解析 HTML
   - 图片: `img[src*="preview.redd.it"]` → `img[src*="i.redd.it"]`
   - 标题: `shreddit-title[title]` → `shreddit-post[post-title]`

### 2.3 YouTube 平台: 插件提取 + 前端嵌入的协作

**后端** [parseHtmlSubprocess.ts#L43-L54](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/scripts/parseHtmlSubprocess.ts#L43-L54):
- `metascraper-youtube` 内部调用 oEmbed 端点 `https://www.youtube.com/oembed?url=...&format=json`
- 支持 HTTP/HTTPS 代理 (防止地域限制 / 提高成功率)
- 产出标准化字段: `title`、`author`、`image` (缩略图 URL)、`publisher`

**前端** [YouTubeRenderer.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/content-renderers/YouTubeRenderer.tsx#L7-L57):
- 正则匹配 5 种 URL 格式提取 Video ID:
  - `youtube.com/watch?v=`
  - `youtu.be/`
  - `youtube.com/embed/`
  - `youtube.com/v/`
  - `youtube.com/shorts/`
- 直接渲染 `<iframe src="https://www.youtube.com/embed/${videoId}">`，**无需后端 oEmbed HTML**

---

## 3. 缩略图处理流程

缩略图存在三级来源和两层本地化策略:

### 3.1 卡片缩略图来源优先级 (由高到低)

在前端卡片展示时，[getBookmarkLinkAssetIdOrUrl](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/shared/utils/bookmarkUtils.ts#L4-L14) 定义选择顺序：

```
优先级从高到低:

1. imageAssetId (下载到本地的横幅图)
   └─ crawlerWorker.downloadAndStoreImage() → AssetTypes.LINK_BANNER_IMAGE
   └─ 对应 DB 字段: bookmarkLinks.imageAssetId

2. screenshotAssetId (浏览器截图)
   └─ Playwright screenshot → JPEG 80% 质量 → AssetTypes.LINK_SCREENSHOT
   └─ 对应 DB 字段: bookmarkLinks.screenshotAssetId

3. imageUrl (远程 URL)
   └─ metascraper-image 提取的 og:image / twitter:image
   └─ 对应 DB 字段: bookmarkLinks.imageUrl
   └─ data URI (base64) 会被过滤掉不存储
```

**注意**: 视频链接的卡片封面也走这条通用管道，没有独立的视频缩略图机制。详见 §3.4

### 3.2 横幅图 (Banner Image) 本地化流程

[crawlerWorker.ts#L1483-L1504](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/crawlerWorker.ts#L1483-L1504) 中 `downloadAndStoreImage`:

1. **配置开关**: `CRAWLER_DOWNLOAD_BANNER_IMAGE` (默认 `true`)
2. **代理下载**: 通过 `fetchWithProxy` 获取远程图片流
3. **大小硬限制**: `MAX_ASSET_SIZE_MB` (默认 50MB)，使用 `Transform stream` 边下边检，超限立即中止
4. **配额校验**: `QuotaService.checkStorageQuota` 检查用户存储余额
5. **存储**: `saveAssetFromFile` → AssetDB (本地文件系统或 S3)
6. **关联**: 写入 `assets` 表，类型 `LINK_BANNER_IMAGE`

### 3.3 浏览器截图生成

[crawlerWorker.ts#L923-L971](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/crawlerWorker.ts#L923-L971):

- **模式选择**:
  - `CRAWLER_FULL_PAGE_SCREENSHOT`: `false` (默认，首屏 1440×900) 或 `true` (全页长图)
  - `CRAWLER_STORE_SCREENSHOT`: 是否启用 (默认 `true`)
- **格式**: JPEG，质量 80，平衡清晰度与文件大小
- **超时**: `CRAWLER_SCREENSHOT_TIMEOUT_SEC` (默认 5 秒)
- **资源防护**: 自动屏蔽 `media` 类型请求 + 显式 video/audio Content-Type，避免页面加载大视频导致超时

### 3.4 视频缩略图: 走通用 og:image 管道

视频链接**没有独立的缩略图机制**，完全复用通用的卡片取图管道：

1. **后端提取**：由 `metascraper-image` / `metascraper-youtube` 等插件从 og:image、twitter:image 等元数据中提取视频封面 URL，写入 `bookmarkLinks.imageUrl`
2. **本地化**：`downloadAndStoreImage()` 将远程封面下载为 `LINK_BANNER_IMAGE` 类型资产，写入 `imageAssetId`
3. **前端展示**：卡片通过 `getBookmarkLinkImageUrl()` 按通用优先级选择图片，与普通网页无区别

**播放时的首帧**由浏览器 `<video>` 标签原生处理，不单独生成 poster 资产。

---

## 4. Metadata 标准化管道

所有来源各异的元数据最终被规整为统一的 `ZBookmarkedLink` 结构。

### 4.1 解析子进程架构

为防止 HTML 解析导致主 Worker 进程内存泄漏/OOM，[crawlerWorker.ts#L1163-L1255](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/crawlerWorker.ts#L1163-L1255) 将解析隔离到**独立子进程**：

```
主进程 (crawlerWorker)
  │  stdin: JSON({htmlContent, url, jobId})
  ▼
子进程 (parseHtmlSubprocess.ts)
  ├─ 内存隔离: --max-old-space-size=CRAWLER_PARSER_MEM_LIMIT_MB (默认 512MB)
  ├─ metascraper 提取 metadata
  ├─ @mozilla/readability 提取正文
  ├─ dompurify 安全消毒 XSS
  └─ stdout: JSON({metadata, readableContent})
     └─ 日志全部重定向到 stderr，避免污染 JSON 协议
```

**标准化 Schema** [parseHtmlSubprocessIpc.ts#L9-L23](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/utils/parseHtmlSubprocessIpc.ts#L9-L23):

```typescript
{
  title: string | null,           // 标题 (多级降级提取)
  description: string | null,     // 摘要
  image: string | null,           // 缩略图 URL (data URI 会被过滤)
  logo: string | null,            // 网站图标/favicon
  author: string | null,          // 作者
  publisher: string | null,       // 发布者
  datePublished: string | null,   // 发布时间 (ISO 字符串)
  dateModified: string | null     // 修改时间 (ISO 字符串)
}
```

### 4.2 懒加载图片规范化

[parseHtmlSubprocess.ts#L76-L110](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/scripts/parseHtmlSubprocess.ts#L76-L110) 在 Readability 提取正文前，修复大量网站使用的 `data-src` 等懒加载属性：

**扫描属性** (按顺序): `data-src` → `data-actualsrc` → `data-srv` → `data-original` → `data-lazy` → `data-lazyload` → `data-img-src` → `data-url`

**触发条件** (仅当 `src` 缺失或为占位图时替换):
- `src` 为空 / `#`
- `src` 为 `data:image/gif` (1x1 透明图)
- `src` 为 `data:image/png` (占位图)

### 4.3 两阶段写入: 快速反馈 + 慢速资产

[crawlerWorker.ts#L1958-L2098](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/crawlerWorker.ts#L1958-L2098) 采用**分阶段事务策略**优化用户体验：

| 阶段 | 写入内容 | 时机 | 目的 |
|------|---------|------|------|
| **Phase 1** | title, description, imageUrl, favicon, author, publisher, 日期 | 解析子进程完成后立即 | 让前端 1-2 秒内看到标题/卡片图 |
| **Phase 2** | 可读内容 (HTML内联或Asset引用)、截图、PDF、横幅图、替换旧资产 | 所有资源下载+存储完毕后 | 完整内容就绪 |

关键细节:
- **data URI 过滤**: `meta.image?.startsWith("data:")` 时置空，避免超大 base64 写入数据库
- **HTML 内联阈值**: `HTML_CONTENT_SIZE_INLINE_THRESHOLD_BYTES` (默认 5KB)，小于阈值存 `bookmarkLinks.htmlContent` 字段，大于则存为 `LINK_HTML_CONTENT` 类型资产
- **日期容错**: `parseDate()` 使用 try/catch，非法字符串静默降级为 `null`

---

## 5. 失败重试机制

重试系统采用 **Restate 持久化状态机 + Dispatcher/Runner 双层架构**。核心设计思想是：Runner 只负责执行（零重试），Dispatcher 只负责调度（集中重试逻辑）。

### 5.1 三层架构与双层重试保障

```
┌─────────────────────────────────────────────────────────────┐
│  Restate Service 层 (外层兜底)                               │
│  dispatcher.ts#L46-L54 retryPolicy:                         │
│    maxAttempts: NUM_RETRIES + 1 (含首次尝试)                 │
│    initialInterval: 5s, maxInterval: 1min                   │
│    仅在 Dispatcher 进程自身崩溃重启时生效                     │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Dispatcher 业务循环层 (实际重试逻辑)                         │
│  dispatcher.ts#L92-L222 while(runNumber <= NUM_RETRIES)     │
│    ├─ 信号量获取 semaphore.acquire()                         │
│    ├─ RPC 调用 runner.run(jobData)                           │
│    ├─ 三路分支判断: success / rate_limit / error             │
│    └─ 信号量释放 semaphore.release()                         │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Runner 执行层 (零重试)                                       │
│  runner.ts#L53-L55 retryPolicy: { maxAttempts: 1 }          │
│  runner.ts#L77-L122:                                         │
│    ├─ tryCatch(funcs.run()) 捕获业务异常                     │
│    ├─ QueueRetryAfterError → { type: "rate_limit" }          │
│    ├─ 普通 Error → { type: "error" }                         │
│    └─ 正常返回 → { type: "success" }                          │
└─────────────────────────────────────────────────────────────┘
```

**双重重试保障说明**:
- **内层 (while 循环)**: 业务代码实现的显式重试，控制三种分支的不同延迟策略
- **外层 (Restate retryPolicy)**: 仅当 Dispatcher 所在进程崩溃/重启时，Restate SDK 接管重放，防止极端情况下任务丢失
- **总尝试次数**: 内层 `NUM_RETRIES + 1` 次（runNumber 从 0 到 `NUM_RETRIES`，条件为 `<=`）

各队列默认重试次数（`numRetries`）:

| 队列 | 重试次数 | 总尝试次数 | 典型场景 |
|------|---------|-----------|---------|
| `LinkCrawlerQueue` / `LowPriorityCrawlerQueue` | 5 | 6 次 | 网页易被反爬封禁 |
| `VideoWorkerQueue` | 5 | 6 次 | 视频下载网络波动大 |
| `SearchIndexingQueue` | 5 | 6 次 | MeiliSearch 索引可能繁忙 |
| `OpenAIQueue` / `EmbeddingsQueue` | 3 | 4 次 | API 限流/计费保护 |
| `AssetPreprocessingQueue` | 2 | 3 次 | OCR 偶发超时 |
| `FeedQueue` / `RuleEngineQueue` | 1 | 2 次 | 周期性任务，失败下次补 |
| `AdminMaintenanceQueue` / `BackupQueue` | 1 | 2 次 | 维护任务，人工可重跑 |
| `WebhookQueue` | 3 | 4 次 | 外部 Webhook 服务不稳定 |

### 5.2 三种重试分支的完整代码链路

Runner 内部用 `tryCatch` 包装 `funcs.run()`，根据异常类型转换为三种结果类型，Dispatcher 再根据类型走三条不同的路径：

```
业务代码 (funcs.run)
    │
    ├─ 正常 return value
    │     └─ runner.ts#L103 → { type: "success", value }
    │           └─ dispatcher 分支: success (见 5.2.1)
    │
    ├─ throw new QueueRetryAfterError(msg, delayMs)
    │     └─ runner.ts#L92-L97 → { type: "rate_limit", delayMs }
    │           └─ dispatcher 分支: rate_limit (见 5.2.2)
    │
    └─ throw new Error(...) (包括 403/429/5xx 导致的 throw)
          └─ runner.ts#L98-L101 → { type: "error", error: serializeError(e) }
                └─ dispatcher 分支: error (见 5.2.3)
```

如果 Runner 服务本身不可用（RPC 连接失败），错误在 `tryCatch(runner.run(jobData))` 层被捕获，走 **RPC 错误分支**（见 5.2.4）。

#### 5.2.1 成功分支: success → 退出循环

[dispatcher.ts#L209-L222](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L209-L222):

```
流程顺序 (信号量一致性保证):
1. logDebug("Dispatcher completed ...")
2. runner.onCompleted({ job, result })   ← 先回调 (更新 DB 状态等)
3. semaphore.release(leaseId)             ← 后释放信号量
4. break;                                  ← 跳出 while 循环，任务结束
```

#### 5.2.2 限流不计数分支: rate_limit → 不消耗配额

**触发条件**: 业务代码显式抛出 `QueueRetryAfterError`，由 [runner.ts#L92-L97](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/plugins/queue-restate/src/runner.ts#L92-L97) 转换为 `{ type: "rate_limit", delayMs }`。

**典型场景**:
- 域名级限流 [crawlerWorker.ts#L2153-L2201](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/crawlerWorker.ts#L2153-L2201): `CRAWLER_DOMAIN_RATE_LIMIT_WINDOW_MS` + `CRAWLER_DOMAIN_RATE_LIMIT_MAX_REQUESTS`，限流时 **+40% 抖动** (`delayMs *= 0.7 + 0.6 * Math.random()`) 防止惊群
- 外部 API 429 响应（由业务代码显式 throw）

**执行流程** [dispatcher.ts#L179-L187](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L179-L187):
```
1. semaphore.release(leaseId)              ← 先释放信号量让其他任务运行
2. ctx.sleep(result.delayMs, "rate limit retry")  ← 精确睡眠调用方指定的时间
3. continue;                                ← 回到 while 开头，runNumber 不变
4. 不调用 onError，不产生 failed 指标
```

**关键特性**: **不消耗 `numRetries` 配额**，理论上可无限次重试，直到条件满足。

#### 5.2.3 业务错误分支: error → 固定 1 秒延迟，消耗配额

**触发条件**: Runner 执行业务逻辑时抛出普通错误（非 `QueueRetryAfterError`），由 [runner.ts#L98-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/plugins/queue-restate/src/runner.ts#L98-L101) 转换为 `{ type: "error" }`。

**典型场景**:
- 爬虫 HTTP 403/429/5xx 状态码（`shouldRetryCrawlStatusCode` 判定后 `throw Error`）
- 数据库事务失败、并发冲突
- 子进程解析异常（OOM、超时、JSON 解析失败）
- Playwright 浏览器自动化超时

**执行流程** [dispatcher.ts#L189-L207](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L189-L207):
```
1. runner.onError({ job, error })          ← 先调用错误回调 (更新 crawlStatus, 指标 +1)
2. semaphore.release(leaseId)              ← 再释放信号量
3. ctx.sleep(1000, "error retry")          ← 固定 1000ms 延迟
4. runNumber++                              ← 消耗重试配额
5. continue;                                ← 回到 while 开头
```

#### 5.2.4 RPC 错误分支: 指数退避 + 全抖动，消耗配额

**触发条件**: Dispatcher 调用 Runner 服务时 RPC 层失败（服务不可达、网络超时、Restate 内部状态异常），即 [dispatcher.ts#L138-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L138-L175) 的 `res.error` 分支。

**典型场景**:
- Runner 服务进程重启/崩溃导致的连接重置
- Restate 持久化状态机写盘失败
- Kubernetes 滚动部署时的短暂不可达
- 网络分区

**执行流程**:
```
1. semaphore.release(leaseId)              ← 先释放信号量
2. runner.onError({ job, error })          ← 调用错误回调
3. 计算延迟 (指数退避 + 全抖动):
   const baseMs = Math.min(5000 * 2 ** runNumber, 60000);
   // runNumber=0 → 5s, 1→10s, 2→20s, 3→40s, 4+→60s 封顶
   const delayMs = Math.floor(ctx.rand.random() * baseMs);
4. ctx.sleep(delayMs, "rpc error retry")   ← 睡眠随机值
5. runNumber++                              ← 消耗重试配额
6. continue;                                ← 回到 while 开头
```

**全抖动 (Full Jitter) 原理**: `[0, baseMs)` 均匀随机，避免大量任务同时失败后在同一时刻再次重试（惊群效应）。

### 5.3 可重试状态码判定

[crawlerWorker.ts#L137-L142](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/crawlerWorker.ts#L137-L142):

```typescript
function shouldRetryCrawlStatusCode(statusCode: number | null): boolean {
  return statusCode === 403       // 可能是临时反爬封禁 (CDN WAF 误封)
      || statusCode === 429       // Rate Limit
      || statusCode >= 500;       // 服务端错误 (500/502/503/504)
}
```

触发时 `throw Error` 抛出，走 **5.2.3 业务错误重试** 分支。

### 5.4 永久失败状态机

当 `runNumber > NUM_RETRIES` 且最后一次仍失败时，while 循环自然结束。[crawlerWorker.ts#L400-L452](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/crawlerWorker.ts#L400-L452) 在 `onError` 回调中，通过 `numRetriesLeft == 0` 判断是否为最后一次：

```
条件: job.numRetriesLeft == 0
  │
  ├─ bookmarkLinks.crawlStatus → "failure"
  ├─ bookmarks.taggingStatus → null (清空 pending，避免无限等待)
  ├─ bookmarks.summarizationStatus → null
  ├─ bookmarks.embeddingStatus → null
  └─ Prometheus: worker_stats_counter{crawler,failed_permanent} +1
```

---

## 6. 本地缓存机制

### 6.1 资产存储: 本地文件系统 / S3 双后端

[assetdb.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/shared/assetdb.ts) 实现 `AssetStore` 抽象:

#### 6.1.1 本地文件系统布局

```
ASSETS_DIR/
└─ {userId}/
   └─ {assetId}/
      ├─ asset.bin          // 原始二进制内容
      └─ metadata.json      // { contentType, fileName }
```

- **目录分片**: 两级目录 (userId → assetId)，避免单目录文件过多
- **原子写入**: `fs.promises.writeFile` + `Promise.all` 并行写入数据和元数据
- **流式支持**: `createAssetReadStream` 支持 HTTP Range 请求 (start/end)，用于视频拖动播放
- **配额校验**: 所有 `saveAsset` / `saveAssetFromFile` 必须传入 `QuotaApproved`，由 `QuotaService.checkStorageQuota` 预审批

#### 6.1.2 S3 对象存储

`ASSET_STORE_S3_ENDPOINT` 配置时自动启用:
- Key 结构: `{userId}/{assetId}` (与本地目录结构同构，便于迁移)
- Metadata 通过 `x-amz-meta-content-type` / `x-amz-meta-file-name` 传输
- 分块范围读取: S3 `Range: bytes=start-end` 原生支持
- 批量删除用户资产: `ListObjectsV2` + `DeleteObjects` (1000 条/批)

### 6.2 进程内缓存

| 缓存位置 | 类型 | TTL | 容量 | 作用 |
|---------|------|-----|------|------|
| [network.ts#L32-L36](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/network.ts#L32-L36) | LRUCache | 5 分钟 | 1000 条 | DNS 解析结果，加速 URL 合法性校验 |
| [metascraper-reddit.ts#L83-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/metascraper-plugins/metascraper-reddit.ts#L83-L98) | Map | 1 分钟 | 无限 (受 TTL 清理) | Reddit JSON API 响应，含在途 Promise 合并 |
| 浏览器实例 | 单例常驻 | - | 1 | Playwright Browser + AdBlocker 规则，避免冷启动 |

### 6.3 DNS 缓存详情

[network.ts#L38-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/network.ts#L38-L73):
- 并发执行 `resolve4` + `resolve6`，结果合并
- 自定义超时: `CRAWLER_IP_VALIDATION_DNS_RESOLVER_TIMEOUT_SEC` (默认 1s)
- SSRF 防护: 返回所有 IP 后逐一校验 `ipaddr.js` 的 range，存在任一禁用 IP 即整体拒绝

### 6.4 资产软引用与垃圾回收

**资产更新模式** (crawlerWorker 中 `updateAsset` 模式):
1. 写入新资产 → 获取新 `assetId`
2. 数据库事务内更新 `bookmarkAssets` / `bookmarkLinks` 关联
3. **事务提交后** 异步 `silentDeleteAsset(userId, oldAssetId)`
4. `silentDeleteAsset` 吞掉所有异常 (`catch(() => ({}))`)，避免删除失败影响主流程

---

## 7. 前端展示与协作机制

### 7.1 渐进式轮询刷新

前端预览的核心机制是 [getBookmarkRefreshInterval](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/shared/utils/bookmarkUtils.ts#L56-L83) 实现的**四段式衰减轮询**，配合 `@tanstack/react-query`:

| 书签创建后时间 | 刷新间隔 | 总刷新次数 (约) |
|--------------|---------|----------------|
| 0 ~ 30 秒 | 1 秒 | ~30 次 (保证立即可见) |
| 30 秒 ~ 10 分钟 | 10 秒 | ~57 次 |
| 10 分钟 ~ 6 小时 | 1 分钟 | ~350 次 |
| 超过 6 小时 | 停止 | - |

**停止条件** (`isBookmarkStillLoading` 为 false):
- 抓取完成: `crawlStatus != 'pending'` 且 `crawledAt != null`
- 标签完成: `taggingStatus != 'pending'`
- 摘要完成: `summarizationStatus != 'pending'`

### 7.2 ContentRenderer 插件化注册表

前端预览支持平台特化渲染器，架构完全可扩展:

**注册入口** [content-renderers/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/content-renderers/index.ts#L1-L14):

```typescript
contentRendererRegistry.register(youTubeRenderer);   // priority: 10
contentRendererRegistry.register(xRenderer);
contentRendererRegistry.register(amazonRenderer);
contentRendererRegistry.register(tikTokRenderer);
contentRendererRegistry.register(instagramRenderer);
```

**匹配算法** [registry.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/content-renderers/registry.ts#L5-L23):
- 遍历所有注册 renderer，执行 `canRender(bookmark)`
- 按 `priority` 降序排序 (高优先级优先展示)
- `Select` 下拉框中自定义 renderer 排在前面，后跟内置视图

### 7.3 多视图切换系统

[LinkContentSection.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/LinkContentSection.tsx#L118-L291) 统一调度 6 种视图:

| 视图 Key | 渲染组件 | 启用条件 |
|---------|---------|---------|
| `{customRendererId}` | ContentRenderer.component | `canRender(bookmark) === true` |
| `cached` | ReaderView + Readability 正文 | 默认视图 |
| `screenshot` | `<Image>` 加载截图资源 | `screenshotAssetId != null` |
| `pdf` | `<iframe>` 内嵌 PDF | `pdfAssetId != null` |
| `archive` | `<iframe sandbox>` 加载 Monolith 归档 | `fullPageArchiveAssetId` 或 `precrawledArchiveAssetId` |
| `video` | `<video controls>` 本地视频 | `videoAssetId != null` |

**渲染安全**:
- 自定义 Renderer 包裹 `ErrorBoundary`，错误降级为技术详情 Alert
- Archive 视图使用 `<iframe sandbox="">` (空策略)，禁止脚本/表单/弹窗/同域访问
- ReaderView 内容经过后端 DOMPurify 消毒

### 7.4 视频前端渲染的三种模式

1. **本地下载模式** (`video` 视图):
   - 后端 VideoWorker 通过 yt-dlp 下载 → 存为 `LINK_VIDEO` 类型资产
   - 前端 `<video><source src="/api/assets/{videoAssetId}">`
   - 支持流式 Range 请求，拖动进度条无需全量加载

2. **平台嵌入模式** (如 YouTubeRenderer):
   - 无需下载视频文件，直接 iframe embed
   - 节省存储配额 + 避免版权风险

3. **原始链接模式** (外链打开):
   - 右上角 "View Original" 链接，跳转原始 URL

### 7.5 抓取中状态的视觉反馈

[BookmarkPreview.tsx#L47-L57](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/BookmarkPreview.tsx#L47-L57) 当 `isBookmarkStillCrawling(bookmark)` 为 true 时:

- 内容区域替换为居中动画: `lucide-react Globe` icon + `animate-bounce` + 多语言文案 "Crawling in progress"
- 侧边栏 (details) 正常显示创建时间、标签、笔记等 (这些不受抓取状态影响)

---

## 8. 关键配置参考

所有开关在 [config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/shared/config.ts#L341-L378) 的 `crawler` 命名空间下:

| 环境变量 | 默认值 | 作用 |
|---------|-------|------|
| `CRAWLER_DOWNLOAD_BANNER_IMAGE` | true | 是否本地化 og:image |
| `CRAWLER_STORE_SCREENSHOT` | true | 是否生成页面截图 |
| `CRAWLER_FULL_PAGE_SCREENSHOT` | false | 全页截图 vs 首屏 |
| `CRAWLER_STORE_PDF` | false | 是否生成 PDF 快照 |
| `CRAWLER_FULL_PAGE_ARCHIVE` | false | 是否 Monolith 单文件归档 |
| `CRAWLER_VIDEO_DOWNLOAD` | false | 是否启用 yt-dlp 下载视频 |
| `CRAWLER_VIDEO_DOWNLOAD_MAX_SIZE` | 50 | 最大视频体积 (MB) |
| `CRAWLER_NUM_WORKERS` | 1 | 爬虫并发数 |
| `CRAWLER_JOB_TIMEOUT_SEC` | 60 | 单次抓取总超时 |
| `CRAWLER_ENABLE_ADBLOCKER` | true | Playwright 加载 Ghostery 广告过滤 |
| `MAX_ASSET_SIZE_MB` | 50 | 单资产硬体积上限 |
| `ASSET_STORE_S3_ENDPOINT` | - | 配置后启用 S3 存储 |

---

## 9. 端到端数据流向 (YouTube 视频示例)

以用户保存 `https://www.youtube.com/watch?v=dQw4w9WgXcQ` 为例，完整链路:

```
时间线:
T+0s    用户保存链接
        └─ LinkCrawlerQueue.enqueue({bookmarkId}, {priority:0})

T+1~5s  CrawlerWorker.runCrawler 执行
        ├─ checkDomainRateLimit → 无限制
        ├─ selectRunProxies → 选定代理
        ├─ getContentType → text/html
        └─ crawlAndParseUrl 执行
            ├─ Playwright.goto → 等待 networkidle
            ├─ capture screenshot (JPEG) + PDF (如启用)
            ├─ 子进程 parseHtmlSubprocess
            │   ├─ metascraper-youtube → 调 oembed 端点
            │   │   └─ {title:"...", author:"Rick Astley", image:"i.ytimg.com/vi/...", publisher:"YouTube"}
            │   └─ readability 提取正文
            ├─ Phase 1 DB update: title/imageUrl/author/publisher → 立即可见
            ├─ downloadAndStoreImage(ytimg.com 缩略图) → LINK_BANNER_IMAGE
            └─ Phase 2 DB update: htmlContent + screenshotAssetId + imageAssetId
                └─ bookmarkLinks.crawlStatus = success

T+5s    (配置允许时)
        ├─ VideoWorkerQueue.enqueue → yt-dlp 下载 (10min 超时)
        ├─ EmbeddingsQueue / OpenAIQueue → AI 标签+摘要
        ├─ SearchIndexingQueue → 搜索同步
        └─ WebhookQueue → 外部通知

T+0~6h  前端 BookmarkPreview
        ├─ react-query 按 1s→10s→60s 衰减轮询
        ├─ 轮询期间: 优先 YouTubeRenderer 嵌入 iframe
        ├─ 视频下载完成后: 可选切换到本地 video 标签播放
        └─ 所有状态 ready: 停止轮询
```

---

## 10. 核心代码位置速查表

| 模块 | 文件路径 |
|------|---------|
| 爬虫主逻辑 | [crawlerWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/crawlerWorker.ts) |
| 网络层/代理/SSRF | [network.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/network.ts) |
| HTML 解析子进程 | [parseHtmlSubprocess.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/scripts/parseHtmlSubprocess.ts) |
| Reddit 插件 | [metascraper-reddit.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/metascraper-plugins/metascraper-reddit.ts) |
| 视频 Worker | [videoWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/workers/workers/videoWorker.ts) |
| 资产存储 | [assetdb.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/shared/assetdb.ts) |
| 队列定义与重试次数 | [queues.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/shared-server/src/queues.ts) |
| Restate Dispatcher (重试核心) | [dispatcher.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/plugins/queue-restate/src/dispatcher.ts) |
| Restate Runner (执行核心) | [runner.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/plugins/queue-restate/src/runner.ts) |
| 前端预览根组件 | [BookmarkPreview.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/BookmarkPreview.tsx) |
| 链接内容多视图 | [LinkContentSection.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/LinkContentSection.tsx) |
| 渲染器注册表 | [registry.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/content-renderers/registry.ts) |
| YouTube 渲染器 | [YouTubeRenderer.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/apps/web/components/dashboard/preview/content-renderers/YouTubeRenderer.tsx) |
| 轮询刷新策略 | [bookmarkUtils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/shared/utils/bookmarkUtils.ts) |
| 配置总入口 | [config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/47-karakeep/packages/shared/config.ts) |
