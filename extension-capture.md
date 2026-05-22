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
| 扩展图标点击 | 点击工具栏图标 | `manifest.json:11-13` → `index.html#save` | `apps/browser-extension/src/SavePage.tsx` |
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
Background ↔ Content Script 使用 `chrome.tabs.sendMessage` 通信：
- 请求：`{ type: "CAPTURE_PAGE", blockImages: boolean }`
- 响应：`{ success: boolean, html?: string, error?: string }`
- 超时：60 秒 (`CAPTURE_TIMEOUT_MS`)

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

// 3. HTTP 上传
const response = await fetch(`${settings.address}/api/assets`, {
  method: "POST",
  headers: { Authorization: `Bearer ${settings.apiKey}` },
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

## 七、关键代码位置索引

| 功能模块 | 主文件路径 | 关键函数/类 |
|---------|-----------|------------|
| 后台事件监听 | `apps/browser-extension/src/background/background.ts` | `handleContextMenuClick`, `addLinkToKarakeep`, `handleCommand` |
| 保存页面逻辑 | `apps/browser-extension/src/SavePage.tsx` | `saveBookmark`, `useEffect` 加载逻辑 |
| tRPC 客户端 | `apps/browser-extension/src/utils/trpc.ts` | `initializeClients`, `getApiClient`, `createTRPCClient` |
| SingleFile 集成 | `apps/browser-extension/src/utils/singlefile.ts` | `capturePageWithSingleFile`, `uploadSingleFileAsset`, `injectSingleFileContentScript` |
| 内容脚本 | `apps/browser-extension/src/content-scripts/singlefile-content-script.ts` | `captureCurrentPage`, `restoreOriginalImageUrls` |
| 设置存储 | `apps/browser-extension/src/utils/settings.ts` | `getPluginSettings`, `usePluginSettings`, `zSettingsSchema` |
| 登录页面 | `apps/browser-extension/src/SignInPage.tsx` | `api.apiKeys.exchange`, `api.apiKeys.validate` |
| API Key 生成 | `packages/trpc/auth.ts` | `generateApiKey`, `authenticateApiKey`, `parseApiKey` |
| API Key 路由 | `packages/trpc/routers/apiKeys.ts` | `apiKeysAppRouter.exchange`, `apiKeysAppRouter.create` |
| 书签创建 | `packages/trpc/routers/bookmarks.ts` | `createBookmark` mutation |
| 资产上传 | `packages/api/routes/assets.ts` | `POST /` endpoint |
| 上下文构建 | `apps/web/server/api/client.ts` | `createContextFromRequest` |
| 认证中间件 | `packages/api/middlewares/auth.ts` | `authMiddleware` |
| API 入口 | `packages/api/index.ts` | Hono app 路由配置 |
