# 浏览器扩展书签与网页快照捕获链路分析

本文档详细分析 Karakeep 浏览器扩展如何捕获书签与网页快照，包括扩展进程与后台 API 的协作、抓取内容的序列化、以及登录凭据的复用机制。

---

## 一、整体架构概览

浏览器扩展采用 **Manifest V3** 架构，核心进程分工如下：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Browser Extension                           │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Background     │   Popup UI      │  Content Scripts            │
│  Service Worker │  (React SPA)    │  (SingleFile injection)    │
└────────┬────────┴────────┬────────┴──────────────┬──────────────┘
         │                 │                       │
         │ tRPC/Fetch      │ tRPC/Fetch            │ Chrome Message Passing
         ▼                 ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Backend API (Hono + tRPC)                   │
│  /api/trpc/*    /api/assets    /api/v1/*                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、捕获入口链路与事件监听

### 2.1 触发入口

扩展通过 **四个独立入口** 接收用户的捕获请求：

| 入口类型 | 触发方式 | 监听位置 | 代码路径 |
|---------|---------|---------|---------|
| 右键菜单 | 右键点击链接/页面/图片/选区 | `background.ts:262` | `apps/browser-extension/src/background/background.ts:104-138` |
| 键盘快捷键 | `Ctrl+Shift+E` | `background.ts:285` | `apps/browser-extension/src/background/background.ts:269-283` |
| 扩展图标点击 | 点击工具栏图标 | `manifest.json:11-13` → `index.html` → 路由 `/` | `apps/browser-extension/src/SavePage.tsx` |
| 命令调用 | `chrome.commands.onCommand` | `background.ts:285` | `apps/browser-extension/src/background/background.ts:269-283` |

### 2.2 上下文菜单注册逻辑

`background.ts:56-97` 中的 `registerContextMenus()` 函数根据配置动态注册菜单项：

```typescript
// 主要菜单项
chrome.contextMenus.create({
  id: ADD_LINK_TO_KARAKEEP_ID,
  title: "Add to Karakeep",
  contexts: ["link", "page", "selection", "image"],  // 支持4种上下文
});
```

### 2.3 事件处理流程

当用户触发捕获时，`handleContextMenuClick()` 或 `handleCommand()` 调用 `addLinkToKarakeep()`（`background.ts:144-184`）：

1.  **内容类型判断**：根据 `selectionText`/`srcUrl`/`linkUrl`/`pageUrl` 的存在性决定书签类型
    - `selectionText` 存在 → `BookmarkTypes.TEXT`
    - 否则使用 `srcUrl ?? linkUrl ?? pageUrl` → `BookmarkTypes.LINK`

2.  **会话存储传递**：将构造的 `ZNewBookmarkRequest` 对象存入 `chrome.storage.session`，键名为 `karakeep-new-bookmark`（`background.ts:180-182`）

3.  **弹出 UI**：调用 `chrome.action.openPopup()` 打开保存页面

### 2.4 Popup 页面加载流程

`SavePage.tsx:69-120` 的 `useEffect` 在组件挂载时：

1.  从 `chrome.storage.session` 读取待处理的书签请求
2.  如无待处理请求，使用当前活动标签页的 URL 和标题创建默认请求
3.  根据 `settings.autoSave` 决定是否自动保存或显示确认 UI

---

## 三、扩展与后台 API 协作机制

### 3.1 tRPC 客户端初始化

`apps/browser-extension/src/utils/trpc.ts` 实现了带缓存和设置变更检测的 tRPC 客户端：

#### 客户端创建 (`trpc.ts:98-111`)
```typescript
apiClient = createTRPCClient<AppRouter>({
  links: [
    httpBatchLink({
      url: `${address}/api/trpc`,
      headers() {
        return {
          Authorization: `Bearer ${apiKey}`,
          ...customHeaders,
        };
      },
      transformer: superjson,          // 类型序列化
    }),
  ],
});
```

#### 关键特性：
- **设置变更检测**：`initializeClients()` (`trpc.ts:23-113`) 会比较新旧设置，仅在地址/API Key/自定义头变更时重建客户端
- **缓存策略**：使用 `@tanstack/react-query-persist-client` 与 Chrome 存储持久化查询结果
- **缓存失效**：地址/API Key 变更时清除所有缓存

### 3.2 双协议通信架构

扩展与后台使用 **两种通信协议**：

#### 协议一：tRPC（主要用于结构化数据）
- **端点**：`POST /api/trpc/bookmarks.createBookmark`
- **用途**：创建书签、查询书签、用户操作
- **序列化**：`superjson` 处理 Date/Map 等复杂类型
- **代码路径**：`SavePage.tsx:45-67`

#### 协议二：REST Fetch（用于文件上传）
- **端点**：`POST /api/assets`
- **用途**：上传 SingleFile 捕获的网页快照
- **序列化**：`multipart/form-data` 封装 Blob
- **代码路径**：`apps/browser-extension/src/utils/singlefile.ts:101-139`

### 3.3 后端路由与处理

#### Hono 应用结构 (`packages/api/index.ts`)
```
/api
├── /trpc/*              → tRPC 适配器处理所有 procedure
├── /assets              → 文件上传/下载 (REST)
└── /v1/*                → REST API v1 (bookmarks, tags, lists 等)
```

#### tRPC 适配器 (`packages/api/middlewares/trpcAdapter.ts`)
将 tRPC 错误码映射为 HTTP 状态码：
- `BAD_REQUEST` → 400
- `UNAUTHORIZED` → 401
- `FORBIDDEN` → 403
- 内部错误在生产环境屏蔽详细信息

---

## 四、抓取内容序列化流程

### 4.1 条件触发

网页快照捕获仅在满足以下所有条件时执行（`SavePage.tsx:127-137`）：
```typescript
if (
  settings.useSingleFile &&              // 用户开启快照功能
  currentTabId !== undefined &&          // 有活动标签页
  bookmark.type === BookmarkTypes.LINK && // 仅链接类型
  !bookmark.precrawledArchiveId &&       // 无预先爬取的归档
  currentTabUrl !== undefined &&
  bookmark.url === currentTabUrl &&      // 书签 URL 匹配当前标签页
  (await hasHostPermission())            // 已授予 <all_urls> 权限
)
```

### 4.2 SingleFile 内容脚本注入

`apps/browser-extension/src/utils/singlefile.ts:59-96` 中的 `injectSingleFileContentScript()`：

1.  从 `manifest.json` 读取 content script 配置
2.  使用 `chrome.scripting.executeScript()` 动态注入
3.  由于 Vite 打包为 ES Module，使用动态 `import()` 而非直接加载文件
4.  重入保护：`window.__karakeepSingleFileLoaded__` 标志防止重复注入

### 4.3 页面捕获与序列化

#### Content Script 侧 (`singlefile-content-script.ts`)
```typescript
async function captureCurrentPage(opts: { blockImages: boolean }): Promise<string> {
  const pageData = await getPageData(
    {
      removeHiddenElements: true,        // 移除隐藏元素
      removeUnusedStyles: true,          // 清理未使用样式
      compressHTML: true,                // HTML 压缩
      blockScripts: true,                // 移除脚本
      blockImages: opts.blockImages,     // 是否阻塞图片
      saveOriginalURLs: opts.blockImages,// 保存原始 URL 以便恢复
      removeFrames: true,                // 移除 iframe
      maxResourceSize: 10,               // 单资源最大 10MB
    },
    {}, document, window
  );
  return opts.blockImages
    ? restoreOriginalImageUrls(pageData.content)  // 恢复 data-sf-original-* 属性
    : pageData.content;
}
```

**序列化优化**：当阻塞图片时，将 `data-sf-original-src` 恢复为 `src`，使图片在查看器中从源站加载。

#### 消息传递协议
Popup ↔ Content Script 使用 `chrome.tabs.sendMessage` 通信（**发起方是 Popup，不是 Background**）：
- 请求：`{ type: "CAPTURE_PAGE", blockImages: boolean }`
- 响应：`{ success: boolean, html?: string, error?: string }`
- 超时：60 秒 (`CAPTURE_TIMEOUT_MS`)

**调用链**：`SavePage.tsx:140` → `capturePageWithSingleFile(tabId)` → `sendCaptureMessage(tabId)` → `chrome.tabs.sendMessage(tabId, ...)`

### 4.4 捕获内容上传序列化

`singlefile.ts:101-139` 中的 `uploadSingleFileAsset()`：

```typescript
// 1. Blob 封装
const blob = new Blob([html], { type: "text/html" });
const filename = sanitizeFilename(title || "page") + ".html";
const file = new File([blob], filename, { type: "text/html" });

// 2. FormData 封装
const formData = new FormData();
formData.append("file", file);

// 3. HTTP 上传（复用 customHeaders）
const headers: HeadersInit = {
  Authorization: `Bearer ${settings.apiKey}`,
};
if (settings.customHeaders) {
  Object.entries(settings.customHeaders).forEach(([key, value]) => {
    headers[key] = value;
  });
}
const response = await fetch(apiUrl, {
  method: "POST",
  headers,
  body: formData,
});
```

### 4.5 服务端接收与存储

`packages/api/routes/assets.ts:15-41` 处理上传：
1.  Zod 验证请求体（`file` 或 `image` 字段）
2.  调用 `uploadAsset()` 工具函数处理存储
3.  返回 `{ assetId, contentType, size, fileName }`

上传成功后，`SavePage.tsx:147` 将 `assetId` 附加到书签请求的 `precrawledArchiveId` 字段。

### 4.6 失败降级策略

客户端捕获是 **尽力而为** 的：
```typescript
try {
  setIsCapturing(true);
  const html = await capturePageWithSingleFile(...);
  const precrawledArchiveId = await uploadSingleFileAsset(...);
  finalBookmark = { ...bookmark, precrawledArchiveId };
} catch (e) {
  console.warn("Client-side crawl failed, saving without archive:", e);
  // 不抛出错误，降级为普通书签，服务端后续仍会尝试爬取
} finally {
  setIsCapturing(false);
}
```

---

## 五、登录凭据复用机制

### 5.1 凭据类型与认证流程

扩展使用 **API Key** 作为认证凭据，有两种获取方式：

#### 方式一：用户名密码交换（"自制 OAuth"）

`SignInPage.tsx:26-33` 调用 `api.apiKeys.exchange.mutation()`：

```
[Extension]                          [Backend]
    |                                    |
    | POST /api/trpc/apiKeys.exchange    |
    | { email, password, keyName }      |
    |                                    |
    | 1. validatePassword()              |
    | 2. generateApiKey()                |
    |                                    |
    | { id, name, key, createdAt }       |
    |                                    |
    | 存储 apiKey 到 chrome.storage.sync |
```

**后端实现**（`packages/trpc/routers/apiKeys.ts:134-194`）：
- 速率限制：15 分钟 10 次请求（防止暴力破解）
- 支持可选的邮箱验证检查
- 生成的 API Key 默认具有 `full_access` 权限

#### 方式二：手动粘贴 API Key

用户从 Web 应用设置页面生成 API Key 后手动粘贴到扩展中。扩展调用 `api.apiKeys.validate()` 验证有效性。

### 5.2 API Key 格式与存储

#### 密钥生成 (`packages/trpc/auth.ts:52-82`)
```typescript
// 格式: ak2_{keyId}_{secret}
const plain = `${API_KEY_PREFIX_V2}_${keyId}_${secret}`;

// 存储: SHA-256 哈希（而非明文）
secretHash: createHash("sha256").update(secret).digest("base64")
```

- `keyId` (10 bytes hex)：用于数据库查找
- `secret` (16 bytes hex)：用于认证验证，**永不存储明文**
- 版本前缀 `ak2` 便于密钥版本迁移

#### 客户端存储 (`apps/browser-extension/src/utils/settings.ts`)
```typescript
const STORAGE = chrome.storage.sync;  // 跨设备同步存储

// 设置 schema 包含 apiKey 字段
const zSettingsSchema = z.object({
  apiKey: z.string(),
  apiKeyId: z.string().optional(),
  address: z.string().optional().default("https://cloud.karakeep.app"),
  // ... 其他设置
});
```

### 5.3 请求认证流程

#### 客户端请求注入 (`trpc.ts:102-106`)
```typescript
headers() {
  return {
    Authorization: `Bearer ${apiKey}`,
    ...customHeaders,  // 用户自定义头（用于反向代理认证）
  };
}
```

#### 服务端认证链

```
[Request]
    │
    ▼
createContextFromRequest()  [apps/web/server/api/client.ts:10-39]
    │
    ├─► 检查 Authorization: Bearer {token}
    │
    ├─► authenticateApiKey(token, db)  [packages/trpc/auth.ts:107-160]
    │    │
    │    ├─ 解析 keyId / keySecret
    │    ├─ 数据库查找 apiKey（通过 keyId）
    │    ├─ SHA-256 验证 secret
    │    └─ 返回 { user, apiKey } 或抛出错误
    │
    └─► 失败则回退到 Cookie Session 认证
```

#### API Key 验证细节 (`auth.ts:107-160`)
```typescript
export async function authenticateApiKey(key: string, database: Context["db"]) {
  const { version, keyId, keySecret } = parseApiKey(key);
  const apiKey = await database.query.apiKeys.findFirst({
    where: eq(k.keyId, keyId),
    with: { user: true },
  });

  // V2 使用 SHA-256 直接比较（性能更好）
  validation = createHash("sha256").update(keySecret).digest("base64") == hash;

  // 节流更新 lastUsedAt（每 10 分钟最多一次）
  if (!apiKey.lastUsedAt || apiKey.lastUsedAt < tenMinutesAgo) {
    database.update(apiKeys).set({ lastUsedAt: new Date() }).where(...);
  }

  return { user: apiKey.user, apiKey: { id, keyId, scopes } };
}
```

### 5.4 作用域权限控制

API Key 支持细粒度作用域（`packages/trpc/index.ts:177-195`）：

```typescript
export function createScopedAuthedProcedure(resource: ZApiKeyScopeResource) {
  return authedProcedure.use((opts) => {
    if (opts.ctx.auth?.type !== "apiKey") return opts.next();

    const access = opts.type === "query" ? "read" : "readwrite";
    const scope = getApiKeyScope(resource, access);

    if (hasRequiredApiKeyScopes(opts.ctx.auth.scopes, [scope])) {
      return opts.next();
    }
    throw new TRPCError({ code: "FORBIDDEN", message: `Missing scope: ${scope}` });
  });
}
```

**作用域示例**：
- `bookmarks:readwrite` → 允许创建/更新/删除书签
- `assets:read` → 仅允许下载资源
- `full_access` → 所有权限

---

## 六、完整调用链时序图

```
User Action (右键菜单 / 快捷键)
    │
    ▼
background.ts handleContextMenuClick()
    │
    ├─► 构造 ZNewBookmarkRequest
    ├─► chrome.storage.session.set()
    └─► chrome.action.openPopup()
    │
    ▼
SavePage.tsx 加载
    │
    ├─► 从 session storage 读取请求
    │
    ├─► [可选] SingleFile 捕获
    │    │
    │    ├─► injectSingleFileContentScript(tabId)
    │    ├─► chrome.tabs.sendMessage(tabId, {type: "CAPTURE_PAGE"})
    │    ├─► content-script 调用 single-file-core/getPageData()
    │    └─► 返回 HTML 字符串
    │
    ├─► [可选] uploadSingleFileAsset(html)
    │    │
    │    ├─► Blob + File + FormData 封装
    │    └─► POST /api/assets  (Authorization: Bearer {apiKey})
    │
    └─► createBookmark()
         │
         └─► tRPC mutation: bookmarks.createBookmark
              │
              ▼
[Backend]
packages/trpc/routers/bookmarks.ts createBookmark
    │
    ├─► 速率限制检查 (60s/30req)
    ├─► 去重检查 (相同 URL 是否已存在)
    ├─► 配额检查 (QuotaService.canCreateBookmark)
    ├─► 数据库事务插入 (bookmarks + bookmark_links)
    ├─► 更新 precrawled archive 关联
    ├─► 入队爬取任务 (LinkCrawlerQueue)
    ├─► 入队 AI 处理任务 (OpenAIQueue)
    └─► 触发搜索索引重建 + Webhook
```

---

## 八、关键事实澄清与调用路径分野

### 8.1 CAPTURE_PAGE 消息发起方：Popup，而非 Background

**澄清**：`CAPTURE_PAGE` 消息的发起方是 **Popup 页面**，不是 Background Service Worker。

**调用链证据**：
```
SavePage.tsx:140  saveBookmark()
    ↓
singlefile.ts:12  capturePageWithSingleFile(tabId, opts)
    ↓
singlefile.ts:19  sendCaptureMessage(tabId, blockImages)
    ↓
singlefile.ts:46  chrome.tabs.sendMessage(tabId, { type: "CAPTURE_PAGE", ... })
    ↓
singlefile-content-script.ts:23  onMessage 监听器接收
```

**设计意图**：页面捕获是用户在 Popup 中点击保存后的同步操作，由 Popup 上下文直接发起可以保持调用栈的完整性，便于错误处理和用户反馈（显示 "Capturing Page" 状态）。

---

### 8.2 默认入口路由为何不是 #save

**路由配置**（`main.tsx:21-39`）：
```typescript
<HashRouter>
  <Routes>
    <Route element={<Layout />}>
      <Route path="/" element={<SavePage />} />  {/* 默认路由 */}
      <Route path="/bookmark/:bookmarkId" element={<BookmarkSavedPage />} />
      ...
    </Route>
    <Route path="/notconfigured" element={<NotConfiguredPage />} />
    <Route path="/options" element={<OptionsPage />} />
    <Route path="/signin" element={<SignInPage />} />
  </Routes>
</HashRouter>
```

**事实澄清**：
- `manifest.json` 配置的 `default_popup: "index.html"` 没有 hash 后缀
- 使用 `HashRouter` 时，无 hash 的路径对应路由 `/`
- 路由 `/` 直接渲染 `<SavePage />` 组件，因此等效于 "保存页面"
- 无需 `#save` 是因为 SavePage 本身就是默认首页

**二次路由逻辑**（`Layout.tsx:14-17`）：
```typescript
if (!settings.apiKey || !settings.address) {
  navigate("/notconfigured");  // 未配置时跳转到配置页面
  return;
}
```
这意味着实际渲染路径可能是：`index.html` → `/` → 检查配置 → `/notconfigured`（如未登录）。

---

### 8.3 链接书签创建后队列分流机制

**双队列架构**（`packages/shared-server/src/queues.ts:89-109`）：

| 队列 | 队列名 | 优先级 | 用途 |
|------|--------|--------|------|
| `LinkCrawlerQueue` | `link_crawler_queue` | `QueuePriority.Default = 0` | 正常优先级爬取（用户交互创建） |
| `LowPriorityCrawlerQueue` | `low_priority_crawler_queue` | `QueuePriority.Low = 50` | 低优先级爬取（批量导入、高频用户） |

> 优先级数值越低，处理越早。

**分流决策逻辑**（`packages/trpc/routers/bookmarks.ts:382-406`）：
```typescript
// 1. 显式优先级：input.crawlPriority === "low"
// 2. 隐式优先级：速率限制触发（5分钟超过30次创建请求）
const forceLowPriority = await shouldUseLowPriorityQueues(ctx);
const shouldUseLowPriority = input.crawlPriority === "low" || forceLowPriority;

const crawlerQueue = shouldUseLowPriority
  ? LowPriorityCrawlerQueue
  : LinkCrawlerQueue;

await crawlerQueue.enqueue({ bookmarkId: bookmark.id }, {
  priority: shouldUseLowPriority ? QueuePriority.Low : QueuePriority.Default,
  groupId: ctx.user.id,  // 按用户分组，保证公平调度
});
```

**速率限制触发降级**（`bookmarks.ts:159-180`）：
```typescript
const highBookmarkCreationRateLimitConfig = {
  name: "bookmarks.createBookmark.highVolume",
  windowMs: 5 * 60 * 1000,   // 5分钟窗口
  maxRequests: 30,           // 超过30次触发降级
};
```

**其他类型队列分流**：
- `BookmarkTypes.LINK` → `LinkCrawlerQueue` / `LowPriorityCrawlerQueue`
- `BookmarkTypes.TEXT` → `OpenAIQueue`（打标签）
- `BookmarkTypes.ASSET` → `AssetPreprocessingQueue`

---

### 8.4 Popup 与 Background 两套 tRPC 客户端关系

**两套独立的 tRPC 客户端实例**，虽然配置同源，但实现和用途完全分离：

#### 客户端一：Background Service Worker 客户端
**位置**：`apps/browser-extension/src/utils/trpc.ts`

```typescript
// 模块级单例
let apiClient: ReturnType<typeof createTRPCClient<AppRouter>> | null = null;
let queryClient: QueryClient | null = null;

// 核心特性
// 1. 设置变更检测（地址/API Key/自定义头变更时重建）
// 2. 持久化缓存（@tanstack/react-query-persist-client + Chrome storage）
// 3. 缓存失效策略（地址/Key 变更时清空）
```

**使用场景**（仅 Background 内部调用）：
- `background.ts:315` → `getBadgeStatus()` → 查询 URL 是否已存档（badge 显示）
- `background.ts:315` → `checkAndUpdateIcon()` → 更新扩展图标 badge
- `badgeCache.ts:12` → `fetchBadgeStatus()` → 调用 `api.bookmarks.checkUrl.query()`

#### 客户端二：Popup React 上下文客户端
**位置**：`packages/shared-react/providers/trpc-provider.tsx`

```typescript
// React Context 管理
function getTRPCClient(settings: Settings) {
  return createTRPCClient<AppRouter>({
    links: [
      httpBatchLink({
        maxURLLength: TRPC_MAX_URL_LENGTH_EXTERNAL,
        fetch: (url, options) => {
          // 自定义 fetch：30秒超时 + AbortController 信号转发
          const controller = new AbortController();
          const timeout = setTimeout(() => controller.abort(), 30_000);
          // ... 信号转发逻辑
          return fetch(url, { ...options, signal: controller.signal });
        },
        headers() {
          return {
            Authorization: settings.apiKey ? `Bearer ${settings.apiKey}` : undefined,
            ...settings.customHeaders,
          };
        },
        transformer: superjson,
      }),
    ],
  });
}
```

**使用场景**（Popup UI 组件调用）：
- `SavePage.tsx:46` → `api.bookmarks.createBookmark.mutation()` → 创建书签
- `SignInPage.tsx:27` → `api.apiKeys.exchange.mutation()` → 登录交换密钥
- `SignInPage.tsx:40` → `api.apiKeys.validate.mutation()` → 验证 API Key

#### 两套客户端对比

| 维度 | Background 客户端 | Popup 客户端 |
|------|------------------|-------------|
| 生命周期 | 模块级单例，随 Service Worker 生命周期 | React 组件上下文，随 Popup 开关重建 |
| 配置来源 | `getPluginSettings()`（chrome.storage.sync） | `TRPCSettingsProvider` props |
| 缓存策略 | 持久化缓存（Chrome storage） | 内存缓存（React Query Provider） |
| 超时处理 | 使用 tRPC 默认 | 自定义 30s 超时 + AbortController |
| 设置变更检测 | 有（比较新旧设置） | 有（`useMemo` 依赖 `settings`） |
| 主要用途 | Badge 检查、后台静默查询 | 用户交互操作（保存、登录） |

**共同点**：都从 `settings` 读取 `apiKey` / `address` / `customHeaders`，都使用 `superjson` 序列化。

---

### 8.5 资产上传自定义请求头的复用关系

**三处独立实现，但复用同一配置源 `settings.customHeaders`**：

#### 实现一：Background tRPC 客户端
**位置**：`apps/browser-extension/src/utils/trpc.ts:102-106`
```typescript
headers() {
  return {
    Authorization: `Bearer ${apiKey}`,
    ...customHeaders,  // 直接展开
  };
}
```

#### 实现二：Popup tRPC 客户端
**位置**：`packages/shared-react/providers/trpc-provider.tsx:82-89`
```typescript
headers() {
  return {
    Authorization: settings.apiKey ? `Bearer ${settings.apiKey}` : undefined,
    ...settings.customHeaders,  // 直接展开
  };
}
```

#### 实现三：资产上传 Fetch 调用
**位置**：`apps/browser-extension/src/utils/singlefile.ts:116-124`
```typescript
const headers: HeadersInit = {
  Authorization: `Bearer ${settings.apiKey}`,
};
if (settings.customHeaders) {
  Object.entries(settings.customHeaders).forEach(([key, value]) => {
    headers[key] = value;  // 手动 forEach 添加
  });
}
```

**设计问题**：三处独立实现了相同的 headers 构造逻辑，没有共享的工具函数。当 `customHeaders` 为空或 undefined 时行为一致，但代码重复。

**自定义头用途**：支持用户在扩展中配置额外的 HTTP 请求头，用于通过反向代理的认证（如 Cloudflare Access、企业内网代理等）。

---

### 8.6 Popup 与 Background 调用路径分野总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User Action                                   │
│  右键菜单 / Ctrl+Shift+E / 点击扩展图标                              │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Background Service Worker                              │
│                                                                     │
│  监听事件 → 构造 ZNewBookmarkRequest → 存入 session storage         │
│  chrome.contextMenus.onClicked                                      │
│  chrome.commands.onCommand                                          │
│  chrome.tabs.onActivated → checkAndUpdateIcon() → getApiClient()   │
│  chrome.runtime.onMessage → BOOKMARK_REFRESH_BADGE                  │
│                                                                     │
│  【tRPC 客户端一】                                                  │
│  用途：静默 badge 查询，持久化缓存                                  │
│  调用：api.bookmarks.checkUrl.query()                               │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼  chrome.action.openPopup()
┌─────────────────────────────────────────────────────────────────────┐
│                      Popup UI (React SPA)                           │
│                                                                     │
│  SavePage.tsx → 读取 session storage → saveBookmark()               │
│  → [可选] capturePageWithSingleFile()                               │
│    → chrome.tabs.sendMessage(tabId, "CAPTURE_PAGE")                │
│  → [可选] uploadSingleFileAsset()                                   │
│    → fetch("/api/assets", { headers: { Authorization, ...custom } })│
│  → createBookmark()                                                 │
│                                                                     │
│  【tRPC 客户端二】                                                  │
│  用途：用户交互操作，30s 超时                                      │
│  调用：api.bookmarks.createBookmark.mutation()                      │
│        api.apiKeys.exchange.mutation()                              │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Backend API                                  │
│  /api/trpc/*    /api/assets                                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键代码位置索引

| 功能模块 | 主文件路径 | 关键函数/类 |
|---------|-----------|------------|
| 后台事件监听 | `apps/browser-extension/src/background/background.ts` | `handleContextMenuClick`, `addLinkToKarakeep`, `handleCommand`, `checkAndUpdateIcon` |
| 保存页面逻辑 | `apps/browser-extension/src/SavePage.tsx` | `saveBookmark`, `useEffect` 加载逻辑 |
| Background tRPC 客户端 | `apps/browser-extension/src/utils/trpc.ts` | `initializeClients`, `getApiClient`, `createTRPCClient` |
| Popup tRPC 客户端 | `packages/shared-react/providers/trpc-provider.tsx` | `TRPCSettingsProvider`, `getTRPCClient` |
| 路由配置 | `apps/browser-extension/src/main.tsx` | `HashRouter`, `Route path="/"` |
| Layout 路由守卫 | `apps/browser-extension/src/Layout.tsx` | 未配置检测 → `/notconfigured` |
| SingleFile 集成 | `apps/browser-extension/src/utils/singlefile.ts` | `capturePageWithSingleFile`, `uploadSingleFileAsset`, `injectSingleFileContentScript`, `sendCaptureMessage` |
| 内容脚本 | `apps/browser-extension/src/content-scripts/singlefile-content-script.ts` | `captureCurrentPage`, `restoreOriginalImageUrls`, `CAPTURE_PAGE` 监听器 |
| 队列定义 | `packages/shared-server/src/queues.ts` | `LinkCrawlerQueue`, `LowPriorityCrawlerQueue`, `QueuePriority` |
| 队列分流 | `packages/trpc/routers/bookmarks.ts` | `shouldUseLowPriorityQueues`, `createBookmark` 中的队列选择 |
| 设置存储 | `apps/browser-extension/src/utils/settings.ts` | `getPluginSettings`, `usePluginSettings`, `zSettingsSchema` |
| 登录页面 | `apps/browser-extension/src/SignInPage.tsx` | `api.apiKeys.exchange`, `api.apiKeys.validate` |
| API Key 生成 | `packages/trpc/auth.ts` | `generateApiKey`, `authenticateApiKey`, `parseApiKey` |
| API Key 路由 | `packages/trpc/routers/apiKeys.ts` | `apiKeysAppRouter.exchange`, `apiKeysAppRouter.create` |
| 书签创建 | `packages/trpc/routers/bookmarks.ts` | `createBookmark` mutation |
| 资产上传 | `packages/api/routes/assets.ts` | `POST /` endpoint |
| 上下文构建 | `apps/web/server/api/client.ts` | `createContextFromRequest` |
| 认证中间件 | `packages/api/middlewares/auth.ts` | `authMiddleware` |
| API 入口 | `packages/api/index.ts` | Hono app 路由配置 |
