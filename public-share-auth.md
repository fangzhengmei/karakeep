# Karakeep 公开分享认证机制梳理

本文档从源码层面梳理公开分享（Public Share）在 Karakeep 中的完整运转路径，覆盖 Share Token 签发、校验、权限范围、撤销传播与预览缓存，重点标注越权与过期边界。

---

## 一、访问通道概览

Karakeep 对同一批列表/书签/资产存在 **五层视角**，每层通过不同的代码路径、不同的鉴权方式呈现不同的数据。下表从最高权限到最低权限排列：

| 层级 | 身份 | 代码入口 | userRole 枚举值 | 访问列表需要的条件 |
|------|------|----------|-----------------|--------------------|
| ① 管理员调试视图 | `users.role = "admin"` | `GET /trpc/admin.getBookmarkDebugInfo` | N/A（绕过列表权限） | 必须知道精确 bookmarkId，无列表级入口 |
| ② 已登录列表所有者 | session / API Key（lists scope） | `List.fromId(ctx, id)` 命中 `bookmarkLists.userId = ctx.user.id` | `"owner"` | 列表存在 + owner 身份 |
| ③ 已登录协作者 | 被邀请加入列表 | `List.fromId()` 命中 `listCollaborators` 表 | `"editor"` 或 `"viewer"` | 列表存在 + collaborator 角色 |
| ④ 公开列表页 | 匿名浏览器 | `GET /public/lists/[listId]` → tRPC `publicBookmarks.*` | `"public"`（impersonate 上下文内） | `bookmarkLists.public = true` （硬编码 token=null） |
| ⑤ RSS Feed 访问 | 匿名 RSS 阅读器 | `GET /api/v1/rss/lists/:listId?token=xxx` | `"public"`（impersonate 上下文内） | `public = true` **OR** `rssToken = token`（token 可选，省略时与 ④ 相同） |

后两层（④⑤）都通过 `List.getPublicList()` 的 `OR(public=true, rssToken=token)` 分支进入，差异只在输出格式（HTML vs RSS XML）以及 RSS 渲染代码自身的 bug。

资产（二进制文件）独立于列表权限体系，存在**三条 URL 生成路径**和**两条下载路由**，它们之间的映射关系非常容易误读：

| URL 生成方式 | 适用场景 | 指向的下载路由 | 是否带签名 | 过期时间 | 是否做隐私过滤 |
|-------------|----------|---------------|-----------|---------|--------------|
| `ZBookmark.assets[]` 仅含 `{id, assetType, fileName}`，**无 URL 字段** | Dashboard 已登录用户 | `/api/assets/:id`（需登录） | 否（用 session cookie） | N/A | 路由层 `Asset.canUserView()` 做权限判断 |
| `Asset.getPublicSignedAssetUrl()`（`getAlignedExpiry(3600, 900)`） | 公开书签 `asPublicBookmark()` 的 `content.assetUrl` / `bannerImageUrl` | `/public/assets/:id?token=xxx`（匿名） | 是（HMAC-SHA256） | 1 小时对齐 + 15 分钟宽限期 | **不过滤**，所有类型资产（含 USER_UPLOADED、BOOKMARK_ASSET）都签 |
| `Asset.getPublicSignedAssetUrl()`（`Date.now() + 10*60*1000`） | Admin debug `buildDebugInfo()` 的 `assets[].url` | `/public/assets/:id?token=xxx`（匿名） | 是（HMAC-SHA256） | 10 分钟固定 | **过滤** `PRIVACY_REDACTED_ASSET_TYPES`，USER_UPLOADED / BOOKMARK_ASSET 返回 `null` |

| 下载路由 | 鉴权方式 | 适配 URL 生成方式 |
|---------|----------|-----------------|
| `/api/assets/:id` | `authMiddleware`（session/API Key）→ `apiKeyScopeMiddleware("assets","read")` → `Asset.canUserView()` | Dashboard 里 `ZBookmark.assets[]` 的 `id`（无 URL，前端拼路径 + cookie 鉴权） |
| `/public/assets/:id?token=xxx` | `unauthedMiddleware` → `verifySignedToken()` → assetId 一致性 → DB 存在性 | 公开书签的签名 URL / Admin debug 的签名 URL |

---

## 二、数据库 Schema 层

**文件**: [schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/db/schema.ts#L475-L498)

```ts
export const bookmarkLists = sqliteTable("bookmarkLists", {
  // ...
  rssToken: text("rssToken"),                       // RSS 分享令牌，nullable
  public: integer("public", { mode: "boolean" })     // 公开开关，默认 false
    .notNull().default(false),
});
```

- `public` — 布尔字段，标记列表是否公开。创建时默认 `false`。
- `rssToken` — 可空字符串。存储由 `crypto.randomBytes(32).toString("hex")` 生成的随机令牌。**无过期时间字段**，令牌一旦生成即永久有效，直到被清除或重新生成。

安全注意点：`rssToken` 在所有面向认证用户的查询中均被 `columns: { rssToken: false }` 过滤，不会泄露给列表的 owner 以外的用户（参见 [lists.ts L95](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L93-L96)）。

---

## 三、五层访问视角的代码顺序对比

从高权限到低权限，每层都走独立的代码路径。下面按 tRPC/API 入口 → 中间件 → 模型鉴权 → 数据脱敏的顺序梳理。

### 3.1 五层入口与中间件校验顺序

| # | 视角 | tRPC/HTTP 入口 | 第一层中间件 | 第二层中间件 | 模型层入口 |
|---|------|----------------|--------------|--------------|------------|
| ① | Admin 调试视图 | `admin.getBookmarkDebugInfo(bookmarkId)` | `authedProcedure`（校验 `ctx.user` 存在） | `createAdminScopedProcedure("bookmarks")` → 校验 `ctx.user.role === "admin"`，且 API Key 需带 `admin:bookmarks:read` scope | `Bookmark.buildDebugInfo()` 内部再校验一次 admin 角色 |
| ② | List Owner | `lists.*` `bookmarks.*` 等 | `createScopedAuthedProcedure("lists"/"bookmarks")`（session 直接放行；API Key 校验对应 scope） | `ensureListAtLeastViewer` → `List.fromId(ctx, id)` 查 `bookmarkLists.userId = ctx.user.id` | `canUserManage() / canUserEdit() / canUserView()` |
| ③ | List Collaborator | `lists.*` `bookmarks.*` 等 | 同上 | `ensureListAtLeastViewer` → `List.fromId()` 回退查 `listCollaborators` 表，取 `role` 字段 | `canUserView()` 对 viewer/editor 返回 true |
| ④ | 公开列表页 | `publicBookmarks.getPublicListMetadata` / `getPublicBookmarksInList` | `publicProcedure`（仅限流，无 auth 校验） | 无 | `List.getPublicList(ctx, id, token=null)` → `WHERE ... AND public=true` |
| ⑤ | RSS 访问 | `GET /api/v1/rss/lists/:id?token=xxx` (Hono) | `unauthedMiddleware`（仅检查 ctx 存在，不校验 user） | 无 | `List.getPublicListContents(ctx, id, token)` → `WHERE ... AND (public=true OR rssToken=token)` |

**关键澄清（易误读点）**：

1. **RSS `token` 是可选参数**，不是必填。[rss.ts L17](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/rss.ts#L17) 定义了 `token: z.string().min(1).optional()`。当 token 省略时，`token ?? null`，走与层 ④ 相同的 `public=true` 分支。所以**不带 token 也可以 RSS 订阅公开列表**。

2. **OR 语义精确表达式**（[lists.ts L172-L179](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L172-L179)）：
   ```
   WHERE id = ? AND (
     (public = true) OR
     (token IS NOT NULL AND rssToken = token)
   )
   ```
   当 token 为 null 时，第二个条件退化为 `false`，只剩 `public=true`。

3. **Admin 调试视图必须知道 bookmarkId**：`admin.getBookmarkDebugInfo` 只接受单条 `bookmarkId` 作为输入，没有"列出所有用户书签"的接口。admin 必须通过其他渠道（如日志、support ticket）获知具体 bookmarkId 才能查询。

4. **Admin 角色双重校验**：除了 `createAdminScopedProcedure` 中间件，`Bookmark.buildDebugInfo()` 内部（[bookmarks.ts L278-L280](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L278-L280)）还有一次冗余校验 `if (ctx.user.role !== "admin") throw new TRPCError({ code: "FORBIDDEN" })`，防御中间件被绕过的深度攻击。

5. **层 ④ 和 层 ⑤ 使用完全相同的模型层函数** `getPublicList()` / `getPublicListContents()`，唯一差异是调用时 `token` 参数：层 ④ 硬编码传 `null`（仅 `public=true` 可过），层 ⑤ 传 URL 查询参数中的 token 值（`public=true` **或** `rssToken` 匹配都可过）。

### 3.2 List 角色矩阵与代码对应

**Schema**（[types/lists.ts L61](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/types/lists.ts#L61)）：

```ts
userRole: z.enum(["owner", "editor", "viewer", "public"]),
```

**模型层判断逻辑**（[lists.ts L397-L428](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L397-L428)）：

```ts
canUserView():  owner ✔  editor ✔  viewer ✔  public ✔
canUserEdit():  owner ✔  editor ✔  viewer ✗  public ✗
canUserManage(): owner ✔  editor ✗  viewer ✗  public ✗
```

- **owner**（层 ②）：`List.fromId()` 在 `bookmarkLists.userId === ctx.user.id` 时设置
- **editor/viewer**（层 ③）：`List.fromId()` 命中 `listCollaborators` 表时取其 `role` 字段
- **public**（层 ④/⑤）：`getPublicListContents()` 内部显式硬编码 `userRole: "public"` 构造 List 对象（[lists.ts L222-L230](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L222-L230)）

公开视角（④⑤）下 `List` 对象的 `ensureCanManage()` 会直接抛 `FORBIDDEN`，所以后续代码无法执行任何写入操作。

### 3.3 各视角可执行的操作对比

| 操作 | ① Admin | ② Owner | ③ Editor | ③ Viewer | ④ Public Web | ⑤ RSS Token |
|------|---------|---------|----------|----------|--------------|-------------|
| 查看列表元数据（名称/描述/图标） | ✔（debug view） | ✔ | ✔ | ✔ | ✔（限 public=true） | ✔（限 token 匹配 或 public=true） |
| 查看列表书签 | ✔（debug view） | ✔ | ✔ | ✔ | ✔（走 asPublicBookmark 脱敏） | ✔（同上，但 RSS 渲染代码再过滤一次） |
| 书签完整内容（htmlContent / 全文） | ✔（HTML 前 1000 字符 preview） | ✔ | ✔ | ✔ | ✗（LINK 只给 url，TEXT 给 text，ASSET 给 assetUrl） | ✗（同上，且 RSS 还会再丢字段） |
| 添加/移除书签 | — | ✔ | ✔ | ✗ | ✗ | ✗ |
| 编辑列表属性（改名/图标/描述） | — | ✔ | ✗ | ✗ | ✗ | ✗ |
| 管理协作者 | — | ✔ | ✗ | ✗ | ✗ | ✗ |
| 开启/关闭公开（`public` 字段） | — | ✔ | ✗ | ✗ | ✗ | ✗ |
| 生成/轮换/清除 RSS Token | — | ✔ | ✗ | ✗ | ✗ | ✗ |
| 删除列表 | — | ✔ | ✗ | ✗ | ✗ | ✗ |
| 查看协作者列表和邀请 | — | ✔（含 pending 邀请） | ✔（仅已接受） | ✔（仅已接受） | ✗（ownerName 可见，协作者隐藏） | ✗ |

**注意事项**：
- Admin ① 的 debug view **不是列表级入口**——`admin.getBookmarkDebugInfo` 只接受单条 `bookmarkId`，不走 List 角色体系，直接 `SELECT * FROM bookmarks WHERE id = ?`（[admin.ts L708-L787](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/admin.ts#L708-L787)）。没有"列出所有用户书签"接口，admin 必须通过其他渠道获知精确 bookmarkId 才能查询。
- Admin debug view 的 `assets[]` 有隐私过滤：`PRIVACY_REDACTED_ASSET_TYPES = { USER_UPLOADED, BOOKMARK_ASSET }` → `url: null`（[bookmarks.ts L285-L288](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L285-L288)），但 LINK_SCREENSHOT、LINK_BANNER_IMAGE、LINK_PDF 等衍生资产会以 10 分钟过期的签名 URL 返回。**这个过滤只在 Admin debug view 中生效**，公开书签 `asPublicBookmark()` 不过滤。
- Dashboard ②/③ 中 `ZBookmark.assets[]` 完全不含 URL 字段，只给 `{id, assetType, fileName}`，前端自行拼 `/api/assets/:id` 走已登录路由。

### 3.4 各视角书签数据字段输出对比

以同一条 LINK 书签为例，各层返回的字段差异：

| 字段 | ① Admin debug | ② Owner（ZBookmark） | ④/⑤ Public（ZPublicBookmark） | ⑤ RSS 实际输出 |
|------|---------------|----------------------|--------------------------------|----------------|
| id | ✔ | ✔ | ✔ | ✔（作为 guid） |
| createdAt | ✔ | ✔ | ✔ | ✔（作为 pubDate） |
| title | ✔ | ✔ | ✔ | ✔ |
| summary | ✔ | ✔ | —（schema 中无此字段） | — |
| description | ✗（debug view 无 description 字段） | ✔ | ✗（asPublicBookmark 未写入，schema 声明了但缺失） | ""（永远空字符串） |
| tags | ✔（含 id, name, attachedBy） | ✔（含 id, name, attachedBy） | ✔（仅 name 字符串数组） | ✔（作为 categories） |
| content.url | ✔（linkInfo.url） | ✔ | ✔ | ✔ |
| content.author | — | ✔ | ✗（schema 声明了但未写入 content） | undefined（永远缺失） |
| htmlContent | ✔（preview 前 1000 字符） | ✔（完整） | ✗ | ✗ |
| crawlStatus / crawledAt | ✔ | ✔ | ✗ | ✗ |
| bannerImageUrl | — | — | ✔（带签名） | ✗（RSS 未用 enclosure） |
| ASSET 书签的主资产 URL | ✔（10 分钟过期签名，USER_UPLOADED/BOOKMARK_ASSET → null） | ✗（ZBookmark content.assetUrl 无此字段；assets[] 仅含 id，前端拼 `/api/assets/:id` 走已登录路由） | ✔（1 小时对齐 + 15 分钟宽限期签名，**所有类型不过滤**） | ❌（误用 `/api/assets/:id` 需登录的 URL） |
| 其他衍生资产（screenshot/pdf/banner 等） | ✔（10 分钟过期签名） | ✗（仅 id + assetType + fileName，前端拼 `/api/assets/:id`） | bannerImageUrl 有签名；其他衍生资产 ID 在 content 中但不单独签 URL | ✗（未用 enclosure） |

### 3.5 各视角的资产下载路径与 URL 生成（按代码顺序）

**三条独立的 URL 生成路径，两个下载路由**：

| 视角 | 模型层如何返回资产信息 | 生成/使用的 URL 类型 | 指向的下载路由 | 代码位置 |
|------|------------------------|---------------------|---------------|---------|
| ②/③ 已登录 Dashboard | `Bookmark.toZodSchema()` 返回 `ZBookmark.assets[] = [{id, assetType, fileName}]`，**无 URL 字段** | 前端自行拼 `/api/assets/{id}`，用 session cookie 鉴权 | `/api/assets/:id`（authMiddleware） | [bookmarks.ts L224-L228](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L224-L228) + [zAssetSchema](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/types/bookmarks.ts#L34-L38) |
| ① Admin debug | `Bookmark.buildDebugInfo()` 对每个资产调 `Asset.getPublicSignedAssetUrl(id, userId, 10min)`，**过滤 `PRIVACY_REDACTED_ASSET_TYPES`**（USER_UPLOADED / BOOKMARK_ASSET → `url: null`） | 已签名 URL `/public/assets/{id}?token=xxx` | `/public/assets/:id?token=xxx`（unauthedMiddleware + 签名验证） | [bookmarks.ts L364-L379](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L364-L379) |
| ④ 公开网页（`asPublicBookmark`） | 对 ASSET 书签的主文件、LINK 书签的 banner image，调 `Asset.getPublicSignedAssetUrl(id, userId, 1h对齐+15min)`，**不过滤隐私类型**（所有资产均签） | 已签名 URL `/public/assets/{id}?token=xxx` | `/public/assets/:id?token=xxx`（unauthedMiddleware + 签名验证） | [bookmarks.ts L770-L837](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L770-L837) |
| ⑤ RSS（当前 bug） | RSS 渲染代码 `toRSS()` 不用 `asPublicBookmark()` 已签名的 `content.assetUrl`，改而用 `getAssetUrl(assetId)` 拼 `/api/assets/{id}` | 未签名 URL `/api/assets/{id}`（需登录） | `/api/assets/:id`（被 authMiddleware 阻断 → 401） | [rss/utils.ts L53-L54](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/utils/rss.ts#L53-L54) |

**两条下载路由的鉴权链（按代码执行顺序）**：

- **`GET /api/assets/:id`（已登录）**：
  1. `authMiddleware`（[auth.ts L25-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/middlewares/auth.ts#L25-L45)）：`ctx.user` 必须存在
  2. `apiKeyScopeMiddleware("assets", "read")`：如果是 API Key 调用，需带 `assets:read` scope；session 调用跳过
  3. `Asset.fromId(ctx, assetId).ensureCanView()` → `Asset.canUserView()`（[assets.ts L225-L251](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/assets.ts#L225-L251)）：
     - `asset.userId === ctx.user.id` → ✅ 放行（owner）
     - `assetType === "avatar"` → ✅ 放行（头像全局公开）
     - `asset.bookmarkId` 存在 → `BareBookmark.bareFromId()` → `isAllowedToAccessBookmark()` → 该书签属于当前用户，或在任一当前用户具有 view 权限的列表中 → ✅/❌
     - 其他 → ❌ 拒绝
  4. `serveAsset()` 流式响应

- **`GET /public/assets/:id?token=xxx`（匿名签名 URL）**：
  1. `unauthedMiddleware`（[auth.ts L9-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/middlewares/auth.ts#L9-L22)）：只检查 `ctx` 存在，不校验 user
  2. `verifySignedToken(token, NEXTAUTH_SECRET, zAssetSignedTokenSchema)`（[signedTokens.ts L79-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/signedTokens.ts#L79-L105)）：
     - Base64 解码 → JSON 解析 → HMAC 签名验证 → 过期检查 → Zod schema 校验
  3. `tokenPayload.assetId !== URL.assetId` → ❌ 403（防止 token 挪用于其他资产）
  4. DB `SELECT * FROM assets WHERE id = ? AND userId = ?` 找不到 → ❌ 404（防止越权访问他人资产）
  5. `serveAsset()` 流式响应

**隐私过滤的精确语义（易误读点）**：

- `PRIVACY_REDACTED_ASSET_TYPES = { USER_UPLOADED, BOOKMARK_ASSET }` 仅在 **Admin debug view** 中生效，目的是不让 admin 拿到用户直接上传的原始文件的签名 URL。
- 在 **公开书签 `asPublicBookmark()` 中不生效**——公开分享的 ASSET 书签本身就是用户主动选择公开的，USER_UPLOADED / BOOKMARK_ASSET 也会被正常签名。
- **`ZBookmark.assets[]`（Dashboard 视图）完全不过滤**，因为 `/api/assets/:id` 路由有 `Asset.canUserView()` 兜底，只有 owner 或协作者才能下载。

**因此 Viewer 可以下载列表中所有书签关联的资产**（Viewer 的 `List.canUserView() === true`，传递到 `BareBookmark.isAllowedToAccessBookmark()` → `Asset.canUserView()` 的第 3 条判断链）。

### 3.6 冒充上下文（impersonating context）的权限边界

层 ④/⑤ 在 `getPublicListContents()` 内部构造：

```ts
const authedCtx = await buildImpersonatingAuthedContext(listdb.userId);
const listObj = List.fromData(authedCtx, {
  ...listdb,
  userRole: "public",  // ← 强制 public 角色
  hasCollaborators: false,
}, null);
```

**文件**: [impersonate.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/lib/impersonate.ts#L1-L30)

该上下文特征：
- `ctx.user.id = listdb.userId`（以 owner 身份）
- `ctx.auth` 字段缺失（非 session、非 API Key）
- `ctx.req.ip = null`

后续只读操作链路：
1. `listObj.getBookmarkIds()` — 对 ManualList 查 `bookmarksInLists`，对 SmartList 走搜索 matcher，均不需要写权限
2. `Bookmark.loadMulti(authedCtx, ids)` — 内部走 `BareBookmark.isAllowedToAccessBookmark()`。由于 `ctx.user.id === bookmarkOwnerId`，**所有书签都会放行**
3. `bookmark.asPublicBookmark()` — 数据脱敏，输出字段已在 3.4 节列出

**越权防线**：`listObj` 的 `userRole: "public"` 导致 `ensureCanManage()` / `ensureCanEdit()` 在任何写操作被调用时立刻抛 `FORBIDDEN`。同时该上下文函数作用域内创建，不会泄漏到 HTTP handler 的返回值中。

---

## 四、RSS Token — 签发、校验、撤销

### 4.1 Token 签发

**文件**: [lists.ts L674-L677](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L674-L677)

```ts
async regenRssToken() {
  this.ensureCanManage();   // ← 仅 owner
  return await this.setRssToken(crypto.randomBytes(32).toString("hex"));
}
```

Token 为 64 字符十六进制字符串（32 字节随机数），写入 `bookmarkLists.rssToken` 列。

### 4.2 Token 校验

RSS Feed 路由通过 `List.getPublicList()` 中的 `eq(bookmarkLists.rssToken, token)` 进行精确匹配。

**注意**: RSS token **无过期机制**。一旦生成，只要不被清除或重新生成，就永久有效。

### 4.3 Token 撤销

两种撤销方式：

1. **regenRssToken** — 生成新 token，旧 token 立即失效（数据库行级替换）。
2. **clearRssToken** — 将 `rssToken` 设为 `null`，彻底关闭 RSS 访问。

两者都通过 `ensureCanManage()` 限制为 owner 操作。

### 4.4 撤销传播分析

**RSS Token 撤销是即时生效的**——因为每次 RSS 请求都实时查询数据库做 token 匹配，不存在缓存层。但以下场景需要注意：

- RSS 阅读器在撤销前已缓存了 feed 内容 → **服务端无法控制**。
- RSS 阅读器保存了旧 token URL → 下次轮询会得到 404（`List not found`）。
- `public = true` 的列表即使 RSS token 被清除，公开列表页面仍然可访问。

### 4.5 RSS Feed 附件资源鉴权问题（核心）

这是最容易混淆的一段脉络：**RSS 订阅输出中的 ASSET 类型书签 URL 与公开资产签名 Token 是两条完全脱节的路径**。

#### 4.5.1 双路由并行：`/assets` vs `/public/assets`

**文件**: [api/index.ts L90-L91](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/index.ts#L90-L91)

```ts
app
  .route("/assets", assets)        // ← 需要登录 authMiddleware
  .route("/public", publicRoute);  // ← /public/assets/* 需要签名 token
```

| 路由 | 路径 | 鉴权 | 用途 |
|------|------|------|------|
| assets route | `/api/assets/:assetId` | `authMiddleware` + `apiKeyScopeMiddleware("assets", "read")` | 已登录 Dashboard 用户下载 |
| public route | `/api/public/assets/:assetId?token=xxx` | `unauthedMiddleware` + HMAC 签名校验 | 匿名公开分享下载 |

**关键区别**: 两条路由最终都调用 `serveAsset()` 流式返回文件，但入口鉴权完全不同。

#### 4.5.2 RSS Feed 生成时的 URL 生成 —— 断点所在

**文件**: [rss.ts L33-L51](file:///d:/fz/0601-2\solo-dogfeeding\code\42-karakeep/packages/api/routes/rss.ts#L33-L51) 调用链：

```
rss.ts handler
  → List.getPublicListContents()          // 返回 ZPublicBookmark[]
    → asPublicBookmark()
      → getContent(ASSET)
        → assetUrl: getPublicSignedAssetUrl(assetId)  // ✅ 生成了签名 URL
  → toRSS(list, bookmarks)
```

**文件**: [rss/utils.ts L40-L42](file:///d:/fz/0601-2\solo-dogfeeding/code/42-karakeep/packages/api/utils/rss.ts#L40-L42)

```ts
feed.item({
  url:
    bookmark.content.type === BookmarkTypes.LINK
      ? bookmark.content.url
      : bookmark.content.type === BookmarkTypes.ASSET
        ? `${serverConfig.publicUrl}${getAssetUrl(bookmark.content.assetId)}`  // ❌ 丢弃已有的签名 URL，重新构造
        : "",
});
```

此处 `getAssetUrl()` 定义在 [assetUtils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/utils/assetUtils.ts#L1-L3):

```ts
export function getAssetUrl(assetId: string) {
  return `/api/assets/${assetId}`;  // ← 无 token，走需登录的路由
}
```

**实际结果**:

- `bookmark.content.assetUrl`（签名的 `/api/public/assets/:id?token=xxx`）**已生成但未使用**
- RSS 输出使用的是 `${publicUrl}/api/assets/:assetId`（**需要登录**），指向 `authMiddleware` 保护的路由
- RSS 订阅者点击 ASSET 链接后会被重定向到登录页 → **下载失败**

#### 4.5.3 ZPublicBookmark Schema 与实际输出的差异

**文件**: [zPublicBookmarkSchema](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/types/bookmarks.ts#L283-L310) 声明了以下字段：

```ts
export const zPublicBookmarkSchema = z.object({
  id, createdAt, modifiedAt, title, tags,
  description: z.string().nullish(),            // ← Schema 声明了
  bannerImageUrl: z.string().nullable(),
  content: z.discriminatedUnion("type", [
    z.object({ type: LINK,   url, author: z.string().nullish(), ... }),  // ← author 声明了
    z.object({ type: TEXT,   text, ... }),
    z.object({ type: ASSET,  assetType, assetId, assetUrl, fileName, sourceUrl, ... }),
  ]),
});
```

但 **`asPublicBookmark()` 实际返回**（[bookmarks.ts L852-L860](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L852-L860)）：

```ts
return {
  id, createdAt, modifiedAt, title, tags,
  // ❌ description 缺失
  content: getContent(content),
  bannerImageUrl: getBannerImageUrl(content),
};
```

并且 `getContent()` 的 LINK 分支（[bookmarks.ts L782-L786](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L782-L786)）：

```ts
case BookmarkTypes.LINK: {
  return {
    type: BookmarkTypes.LINK,
    url: content.url,
    // ❌ author 缺失
  };
}
```

**RSS Feed 中对应字段的表现**（[rss.ts L44-L49](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/utils/rss.ts#L44-L49)）：

```ts
author:
  bookmark.content.type === BookmarkTypes.LINK
    ? (bookmark.content.author ?? undefined)  // ← 永远 undefined
    : undefined,
categories: bookmark.tags,
description: bookmark.description ?? "",    // ← 永远空字符串 ""
```

结论：**RSS 输出中 description 永远为空，LINK 类型 author 永远缺失**。原因是 `asPublicBookmark()` 没有把这两个字段写入返回对象，而 Zod schema 的 `nullish` 对缺失值只是"容忍"，不会补默认值。

#### 4.5.4 ASSET 类型书签的 `<enclosure>` 元素缺失

RSS 2.0 标准中 `<enclosure>` 元素用于表示附件（图片/PDF等）。`toRSS()` 没有调用 `feed.item({ enclosure: {...} })`，导致：

1. ASSET 类书签在 RSS 客户端中不会以附件形式渲染（需要用户点击跳转）。
2. banner images（`bannerImageUrl` 字段中已经包含的签名图片 URL）不会作为缩略图在 RSS 阅读器中显示。

#### 4.5.5 RSS feedUrl 与 siteUrl 的精确含义（易误读点）

**文件**: [rss.ts L44-L48](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/rss.ts#L44-L48)

```ts
const rssFeed = toRSS(
  {
    title: `Bookmarks from ${list.icon} ${list.name}`,
    feedUrl: `${serverConfig.publicApiUrl}/v1/rss/lists/${listId}`,    // ← RSS 订阅 URL（正确）
    siteUrl: `${serverConfig.publicUrl}/dashboard/lists/${listId}`,    // ← 私有 Dashboard 链接（问题所在）
    description: list.description ?? undefined,
  },
  res.bookmarks,
);
```

**两个 URL 的精确含义**：

| 变量 | 生成规则 | RSS 元素 | 用途 | 是否正确 |
|------|----------|----------|------|----------|
| `feedUrl` | `NEXTAUTH_URL` + `/api/v1/rss/lists/{listId}` | `<atom:link rel="self" href="...">` | RSS 阅读器识别 feed 自身地址，用于去重和重新订阅 | ✅ 正确 |
| `siteUrl` | `NEXTAUTH_URL` + `/dashboard/lists/{listId}` | `<channel><link>` | RSS 阅读器点击"访问原网站"时跳转的页面 | ❌ 错误（指向私有 Dashboard） |

**配置来源**（[config.ts L264-L265](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/config.ts#L264-L265)）：
- `publicUrl: NEXTAUTH_URL`（用户部署时配置的域名，如 `https://karakeep.example.com`）
- `publicApiUrl: NEXTAUTH_URL + "/api"`

**修复方向**：`siteUrl` 应改为 `${serverConfig.publicUrl}/public/lists/${listId}`，指向公开列表页面而非私有 Dashboard。

特别注意：对于**仅 RSS Token 访问**的场景（`public=false` 但有正确的 `?token=xxx`），`/public/lists/${listId}` 公开页面本身无法打开（因为层 ④ 不走 OR 逻辑，只看 `public=true`）。此时需要考虑：
- 对于 token-only 访问，`siteUrl` 应该仍然是 RSS feed 本身（附带 token），或者考虑公开页面是否也接受 token 参数。
- 目前代码中，`/public/lists/:listId` 页面完全不认识 `?token=` 参数，也不会传递给 `publicBookmarks.*` tRPC 调用。


#### 4.5.6 类型过滤副作用：TEXT 书签在 RSS 中不可见

**文件**: [rss/utils.ts L28-L32](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/utils/rss.ts#L28-L32)

```ts
bookmarks
  .filter(
    (b) =>
      b.content.type === BookmarkTypes.LINK ||
      b.content.type === BookmarkTypes.ASSET,
  )
```

TEXT 类型书签在 RSS 中被**完全过滤掉**。虽然纯文本没有"链接地址"，但 RSS `<item>` 的 `<description>` 可以承载文本内容，所以理论上是可以输出的。

#### 4.5.7 RSS Feed 缓存头

**文件**: [rss.ts L53-L54](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/rss.ts#L53-L54)

```ts
c.header("Content-Type", "application/rss+xml");
return c.body(rssFeed);
```

RSS 响应**无任何 Cache-Control 头**，默认行为取决于 RSS 阅读器的实现。好处是任何改动（新增/删除书签、调整公开状态、轮换 token）立即生效；坏处是 RSS 阅读器的高频轮询（通常 10-30 分钟一次）会每次重新查询数据库 + 重新生成签名 token，带来一定的性能开销。

---

## 五、公开资产 — Signed Token 机制

### 5.1 Token 签发

**文件**: [assets.ts L266-L281](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/assets.ts#L266-L281)

```ts
static getPublicSignedAssetUrl(assetId: string, assetOwnerId: string, expireAt: number) {
  const payload: z.infer<typeof zAssetSignedTokenSchema> = { assetId, userId: assetOwnerId };
  const signedToken = createSignedToken(payload, serverConfig.signingSecret(), expireAt);
  return `${serverConfig.publicApiUrl}/public/assets/${assetId}?token=${signedToken}`;
}
```

**Payload 结构**（[zAssetSignedTokenSchema](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/types/assets.ts#L1-L6)）:

```ts
export const zAssetSignedTokenSchema = z.object({
  assetId: z.string(),   // 资产 ID
  userId: z.string(),    // 资产所有者 ID
});
```

**签名过程**（[signedTokens.ts L50-L74](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/signedTokens.ts#L50-L74)）:

1. 构造 `{ payload: { assetId, userId }, expiresAt }` JSON。
2. 使用 `HMAC-SHA256(NEXTAUTH_SECRET)` 对 JSON 字符串签名。
3. 将 `{ payload, signature }` 编码为 Base64 字符串作为 token。

**密钥来源**: `serverConfig.signingSecret()` 返回 `NEXTAUTH_SECRET` 环境变量（[config.ts L266-L271](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/config.ts#L266-L271)）。**与 NextAuth 共用同一密钥**。

### 5.2 Token 过期策略

**公开书签中的资产**（[bookmarks.ts L770-L777](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L770-L777)）:

```ts
const getPublicSignedAssetUrl = (assetId: string) => {
  // 1 小时过期，15 分钟宽限期
  return Asset.getPublicSignedAssetUrl(assetId, this.bookmark.userId, getAlignedExpiry(3600, 900));
};
```

`getAlignedExpiry`（[signedTokens.ts L24-L46](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/signedTokens.ts#L24-L46)）将过期时间对齐到固定间隔，减少因时钟偏移导致的边界问题：

- 间隔 3600 秒（1 小时）
- 宽限期 900 秒（15 分钟）
- 如果距离下一个间隔不足 15 分钟，跳到再下一个间隔

**认证用户书签中的资产**（[bookmarks.ts L328-L329](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L328-L329)）:

```ts
const expiresAt = Date.now() + 10 * 60 * 1000; // 10 分钟
```

认证用户的资产 token 使用固定 10 分钟过期，不做对齐。

### 5.3 Token 校验

**文件**: [public/assets.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/public/assets.ts#L1-L50)

```ts
.get("/:assetId", unauthedMiddleware, zValidator("query", z.object({ token: z.string() })), async (c) => {
  const assetId = c.req.param("assetId");
  const tokenPayload = verifySignedToken(
    c.req.valid("query").token,
    serverConfig.signingSecret(),
    zAssetSignedTokenSchema,
  );
  if (!tokenPayload) return c.json({ error: "Invalid or expired token" }, { status: 403 });
  if (tokenPayload.assetId !== assetId) return c.json({ error: "Invalid or expired token" }, { status: 403 });
  // ← 双重校验：token 中的 assetId 必须匹配 URL 中的 assetId

  const assetDb = await c.var.ctx.db.query.assets.findFirst({
    where: and(eq(assets.id, assetId), eq(assets.userId, tokenPayload.userId)),
  });
  if (!assetDb) return c.json({ error: "Asset not found" }, { status: 404 });
  return await serveAsset(c, assetId, tokenPayload.userId);
})
```

校验步骤（按顺序）：

1. **Base64 解码 + JSON 解析** — 格式错误返回 `null`。
2. **HMAC 签名验证** — 签名不匹配返回 `null`。
3. **Zod schema 验证** — payload 结构不符合 `zAssetSignedTokenSchema` 返回 `null`。
4. **过期检查** — `Date.now() > payload.expiresAt` 返回 `null`。
5. **assetId 一致性** — URL 路径中的 `assetId` 必须与 token 中的 `assetId` 一致（防止 token 挪用）。
6. **数据库存在性** — `assets.id = assetId AND assets.userId = userId` 必须能查到记录（防止越权访问他人资产）。

`unauthedMiddleware` 仅检查 `ctx` 是否存在，不校验用户登录状态（[auth.ts L9-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/middlewares/auth.ts#L9-L22)）。

### 5.4 隐私过滤与签名 URL 的关系（按代码顺序精确对应）

**文件**: [bookmarks.ts L285-L288](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L285-L288)

```ts
const PRIVACY_REDACTED_ASSET_TYPES = new Set<AssetTypes>([
  AssetTypes.USER_UPLOADED,
  AssetTypes.BOOKMARK_ASSET,
]);
```

**这个 Set 只在一个地方使用**——`Bookmark.buildDebugInfo()`（Admin 调试视图）。它控制的是**给 admin 看的 debug info 中，用户直接上传的原始文件要不要给签名 URL**。过滤逻辑在 [bookmarks.ts L369-L371](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L369-L371)：

```ts
const url = !PRIVACY_REDACTED_ASSET_TYPES.has(a.assetType)
  ? Asset.getPublicSignedAssetUrl(a.id, bookmark.userId, expiresAt) // 10 分钟
  : null;
```

**三条 URL 生成路径的精确对比（与 §1 概览、§3.5 一致）**：

| 路径 | 是否使用 PRIVACY_REDACTED_ASSET_TYPES | 过滤结果 | 生成的 URL 指向 |
|------|---------------------------------------|---------|---------------|
| ① Admin debug `buildDebugInfo().assets[]` | ✅ 使用 | USER_UPLOADED、BOOKMARK_ASSET → `url: null`；其他类型（screenshot、banner、pdf 等）→ 10 分钟签名 URL | `/public/assets/:id?token=xxx` |
| ②/③ Dashboard `ZBookmark.assets[]` | ❌ 不使用（完全无 URL 字段） | `{id, assetType, fileName}` 仅元数据，前端自行拼 `/api/assets/:id` + session cookie | `/api/assets/:id`（authMiddleware） |
| ④ 公开 `asPublicBookmark()` content.assetUrl / bannerImageUrl | ❌ 不使用 | 所有类型（含 USER_UPLOADED、BOOKMARK_ASSET）→ 1 小时对齐 + 15 分钟宽限期签名 URL | `/public/assets/:id?token=xxx` |

**为什么公开书签不过滤？** 设计上，公开分享的 ASSET 书签是用户主动把列表设为公开的，列表内的所有内容（包括用户上传的 PDF、图片）理应随列表一起公开。而 Admin debug view 是管理员排查问题用的，默认不让管理员获得用户原始文件的直接下载链接（除非用户明确通过公开分享暴露）。

**注意老版本描述的错误**：之前说"认证用户查看书签时，这两种类型的资产不生成签名 URL"——这不对。认证用户在 Dashboard 中拿到的是 `ZBookmark.assets[]`，它**根本就没有 URL 字段**，不是"不生成签名 URL"，而是"根本不生成 URL，前端走独立的 `/api/assets/:id` 已登录路由"。

---

## 六、预览缓存行为

### 6.1 资产文件缓存

**文件**: [utils/assets.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/utils/assets.ts#L28-L29)

```ts
c.header("Cache-Control", "private, max-age=31536000, immutable");
```

资产文件设置了 **1 年强缓存 + immutable** 标记。但 `private` 意味着 CDN/代理不应缓存。

**问题**: 资产 URL 包含带过期的签名 token，当 token 过期后，即使浏览器缓存了响应，新请求会因为 token 验证失败而被拒绝。但 **已缓存在浏览器中的响应不受服务端控制**——这是一个通用的签名 URL 缓存问题，不算漏洞。

### 6.2 公开列表页面缓存

**文件**: [page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/apps/web/app/public/lists/[listId]/page.tsx)

Next.js 页面使用 SSR（服务端渲染），无 `revalidate` 或 `dynamic` 配置，走 Next.js 默认缓存策略。`generateMetadata()` 在 SSR 时调用 `api.publicBookmarks.getPublicListMetadata()`，此调用走 tRPC 公开端点，不需要认证。

**注意**: 如果使用 Next.js ISR 或 Full Route Cache，列表被设为非公开后，缓存的页面可能在短时间内仍然可访问，直到缓存失效。

---

## 七、越权与过期边界总结

### 7.1 已识别的边界问题

| # | 问题 | 严重度 | 说明 |
|---|------|--------|------|
| 1 | RSS Token 无过期时间 | **中** | `rssToken` 在数据库中无 `expiresAt` 字段，一旦泄露无法自动失效。只能通过 `regenRssToken` 或 `clearRssToken` 手动撤销。 |
| 2 | 公开列表无访问审计 | **低** | `publicProcedure` 和 `unauthedMiddleware` 均不记录访问者信息（无 auth），无法追踪谁在何时访问了公开列表或 RSS。 |
| 3 | 资产签名 URL 浏览器缓存 | **低** | `Cache-Control: private, max-age=31536000, immutable` 意味着浏览器可能长期缓存资产。即使签名过期或列表被设为非公开，已缓存内容不受影响。 |
| 4 | 公开列表开关无确认步骤 | **低** | 从非公开切换为公开是即时生效的（`lists.edit` mutation → `ensureCanManage` → `UPDATE`），没有二次确认或"预览即将公开的内容"步骤。 |
| 5 | `public` 字段与 `rssToken` 独立 | **中** | 清除 RSS Token 不会关闭公开列表；关闭公开列表不会清除 RSS Token。两者是独立的访问通道，关闭其中一个不影响另一个。 |
| 6 | NEXTAUTH_SECRET 双重用途 | **中** | 签名 Token 与 NextAuth 共用 `NEXTAUTH_SECRET`。密钥轮换会使所有已签发的资产 token 立即失效，可能导致用户体验中断。 |
| 7 | **RSS ASSET 链接不可下载** | **高** | RSS Feed 中 ASSET 类书签的 URL 使用 `getAssetUrl()`（`/api/assets/:id`，需登录），**丢弃**了 `asPublicBookmark()` 中已生成的签名 `assetUrl`（`/api/public/assets/:id?token=xxx`）。RSS 订阅者点击会被弹到登录页。 |
| 8 | **RSS description 永远为空** | **中** | `zPublicBookmarkSchema` 声明了 `description` 字段，但 `asPublicBookmark()` 的返回对象中没有该字段。RSS 渲染 `bookmark.description ?? ""` → 永远空字符串。 |
| 9 | **RSS LINK author 永远缺失** | **低** | LINK 类型的公开 content schema 声明了 `author` 字段，但 `getContent(LINK)` 返回 `{type, url}` 不包含 author。RSS 的 `<author>` 元素永远不输出。 |
| 10 | **RSS channel siteUrl 指向私有 Dashboard** | **中** | `<channel><link>`（`siteUrl`）写死为 `NEXTAUTH_URL/dashboard/lists/:listId`（需登录的私有 Dashboard），而非公开列表页。`feedUrl`（`<atom:link rel="self">`）是正确的 RSS 订阅 URL。注意：即使把 siteUrl 改为 `/public/lists/:listId`，对**仅 RSS Token 访问**的场景（`public=false`，只靠 token 访问）也不生效——公开列表页完全不认识 `?token=` 参数，调用 `publicBookmarks.*` 时 token 传 null，仍会被 `public=true` 条件挡住。 |
| 11 | **RSS TEXT 类型书签被过滤** | **低** | TEXT 书签在 `toRSS()` 的 `.filter(b => LINK \|\| ASSET)` 中被完全过滤，RSS 客户端无法看到纯文本书签。 |
| 12 | **RSS 无 enclosure 元素** | **低** | ASSET 类书签和 banner image 未使用 RSS 2.0 `<enclosure>` 或 `<media:thumbnail>` 扩展，RSS 阅读器无法渲染附件和缩略图。 |
| 13 | **Admin debug view 可跨用户查看任何书签（但需精确 bookmarkId）** | **中** | `admin.getBookmarkDebugInfo` 走 `createAdminScopedProcedure("bookmarks")` 校验 `ctx.user.role === "admin"` + API Key `admin:bookmarks:read` scope，`Bookmark.buildDebugInfo()` 内部再冗余校验一次 admin 角色。核心限制：**必须知道精确 bookmarkId**——没有"列出所有用户书签"接口，无法枚举。但 admin 如果通过其他渠道（日志、support request）获得 bookmarkId，可直接 `SELECT * FROM bookmarks WHERE id = ?` 读取完整内容（含 htmlContent 前 1000 字符 preview），绕过 List 角色体系。**资产隐私过滤仅在此处生效**：USER_UPLOADED / BOOKMARK_ASSET 返回 `url: null`（10 分钟签名 URL 不签发），但截图、banner、PDF 等衍生资产正常签发。 |
| 14 | **Viewer 角色可下载列表中所有书签的关联资产** | **低** | `Asset.canUserView()` 通过 `BareBookmark.bareFromId` → `List.forBookmark` → `List.canUserView()` 级联判断。Viewer 对列表有 view 权限会自动传递到所有关联资产。符合预期，但与"viewer 不可写"的权限模型相比，资产下载算是 viewer 的隐藏能力。 |
| 15 | **公开视角 impersonating context 以 owner 身份读全量书签** | **低** | `buildImpersonatingAuthedContext(listdb.userId)` 在层 ④/⑤ 内部构造以 owner 身份的 ctx，`Bookmark.loadMulti` 会因为 `ctx.user.id === bookmarkOwnerId` 放行所有书签。依赖后续 `asPublicBookmark()` 做数据脱敏。安全依赖于脱敏函数的完整性——如果有字段被加入 ZBookmark 但忘记在 asPublicBookmark 中裁剪，会直接泄漏。`listObj` 的 `userRole: "public"` 保证 `ensureCanManage()` / `ensureCanEdit()` 会在任何写操作前立刻抛 `FORBIDDEN`，深度防御。 |
| 16 | **Token-only RSS 访问无法通过网页查看同一列表** | **低** | 对于 `public=false` 但持有正确 RSS Token 的用户，RSS feed 可以正常读取，但点击 `<channel><link>` 跳转的公开列表页 `/public/lists/:id` 无法识别 token，会报 `List not found`。两条通道的 token 体系没有打通。 |

### 7.2 越权防护验证（E2E 测试覆盖）

**文件**: [public.test.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/e2e_tests/tests/api/public.test.ts)

E2E 测试覆盖了以下越权场景：

- 无 token 访问公开资产 → 400
- 畸形 token → 403
- 无效 payload 结构 → 403
- 过期 token → 403
- 使用 A 资产的 token 访问 B 资产 → 403（assetId 不匹配）
- 使用 User2 的 userId 签发 token 访问 User1 的资产 → 404（DB 查询找不到）
- 访问非公开列表 → `List not found`
- 引用不存在的 assetId → 404

**当前 E2E 未覆盖**（需补充）：
- RSS endpoint 返回 404 对错误 token
- RSS endpoint 对无 token 请求返回 404 对非公开列表
- RSS ASSET URL 是否为签名 URL（当前会失败，因为 bug）
- RSS `feedUrl` 和 `siteUrl` 的正确性
- Admin debug view 越权访问非 admin 用户书签（需知道 bookmarkId）
- Viewer 无法 edit/manage 列表
- Collaborator 无法看到 rssToken（被 `columns: { rssToken: false }` 过滤）
- 公开列表页不接受 token 参数（层 ④ 与层 ⑤ 能力差异）

### 7.3 未覆盖的边界场景

| 场景 | 现状 |
|------|------|
| 列表从公开切为非公开后，已签发的资产 token 是否仍可访问资产？ | **可以**。资产 token 绑定的是 `assetId + userId`，不检查列表公开状态。即使列表变为非公开，只要 token 未过期，资产仍可下载。 |
| RSS Token 泄露后的窗口期 | **无限大**。除非 owner 手动轮换，泄露的 token 永久有效。 |
| 并发竞争：owner 正在关闭公开时，公开请求是否仍可通过？ | **可能**。数据库 UPDATE 和 SELECT 之间无事务隔离，存在极短的竞争窗口。 |
| 删除列表后 RSS Token 是否残留？ | **不残留**。列表删除时 `rssToken` 随行删除（列表整行删除）。 |
| RSS 中 ASSET 链接失效后，RSS 阅读器缓存的旧 XML 是否仍能访问？ | **链接会失效**，但已缓存的 XML 内容（不含二进制文件）阅读器仍会保留。 |
| NEXTAUTH_SECRET 轮换后，仍在 RSS 阅读器缓存中的旧签名 assetUrl 会怎样？ | **全部失效**，点击会返回 403。由于 RSS 本身无缓存头，阅读器轮询时会重新拉取 feed，但中间窗口期内的链接会中断。 |
| RSS 订阅时用的是 `?token=xxx`（RSS Token），但书签内的资产签名 Token 到期时间是 1 小时，会出现什么？ | feed 中每个 item 的资产 URL 在每次轮询时都会**重新签发**（对齐到小时+宽限期），所以 RSS 阅读器每次抓最新的 feed 时链接都是有效的。**但**如果阅读器缓存了 RSS item 的 URL（不重新拉取），1 小时后会失效。 |
| Admin 账号被攻破后的数据暴露面 | **受限于 bookmarkId 枚举难度**。`admin.getBookmarkDebugInfo` 需要知道精确 bookmarkId，且没有列出所有书签的接口。但攻击者如果掌握 admin session 并能够枚举有效的 bookmarkId（如通过日志泄漏、ID 预测、时序攻击等），可读取所有用户的书签内容（含 htmlContent preview）。注意：admin debug view 仍然有 PRIVACY_REDACTED_ASSET_TYPES 过滤，USER_UPLOADED 和 BOOKMARK_ASSET 类型资产不返回签名 URL。 |
| Viewer 移除后仍持有书签关联资产的旧签名 URL | **仍可下载**，直到签名过期（最长 1 小时 15 分钟）。资产签名 token 不校验协作者关系，只校验 assetId + userId。 |
| `public=false` + 持有 RSS Token 的用户，能否通过公开列表页查看？ | **不能**。公开列表页 `GET /public/lists/:id` 完全不识别 `?token=` 查询参数，调用 tRPC `publicBookmarks.*` 时 token 硬编码传 `null`，只走 `public=true` 分支。两条通道的 token 体系没有打通。 |

---

## 八、完整请求流转图

```
公开列表页面访问:
  浏览器 → GET /public/lists/[listId]
    → Next.js SSR → tRPC publicBookmarks.getPublicListMetadata(listId)
      → List.getPublicList(ctx, listId, null)
        → DB: SELECT * FROM bookmarkLists WHERE id=? AND (public=true)
    → Next.js SSR → tRPC publicBookmarks.getPublicBookmarksInList(listId)
      → List.getPublicList(ctx, listId, null)  ← 再次校验
        → buildImpersonatingAuthedContext(ownerId)
        → listObj.getBookmarkIds()
        → Bookmark.loadMulti(authedCtx, ids) → asPublicBookmark()
          → Asset.getPublicSignedAssetUrl(assetId, userId, alignedExpiry)
            → createSignedToken({assetId, userId}, NEXTAUTH_SECRET, expireAt)
              → HMAC-SHA256 签名 → Base64 编码
    → 返回 HTML + 带签名 token 的资产 URL

Admin 调试视图（单书签）:
  Admin 浏览器 → GET /trpc/admin.getBookmarkDebugInfo?input={bookmarkId:xxx}
    → authedProcedure (校验 ctx.user 存在)
    → createAdminScopedProcedure("bookmarks") → ctx.user.role === "admin" ? + API Key scope admin:bookmarks:read
    → Bookmark.buildDebugInfo(ctx, bookmarkId)
      → 内部二次校验 ctx.user.role !== "admin" → FORBIDDEN
      → SELECT * FROM bookmarks WHERE id=? （绕过 List 角色体系）
      → PRIVACY_REDACTED_ASSET_TYPES 过滤: USER_UPLOADED / BOOKMARK_ASSET → url: null
      → 其他资产: Asset.getPublicSignedAssetUrl(id, userId, Date.now()+10min)  (走 /public/assets/:id?token=xxx)
      → LINK: htmlContent preview 前 1000 字符

资产下载（正确路径）:
  浏览器 → GET /api/public/assets/{assetId}?token=xxx
    → unauthedMiddleware (仅检查 ctx 存在)
    → verifySignedToken(token, NEXTAUTH_SECRET, zAssetSignedTokenSchema)
      → Base64 解码 → JSON 解析 → HMAC 签名验证 → 过期检查 → Zod 验证
    → tokenPayload.assetId !== URL.assetId? → 403
    → DB: SELECT * FROM assets WHERE id=? AND userId=?  → 404 if not found
    → serveAsset() → Cache-Control: private, max-age=31536000, immutable

已登录 Dashboard 资产下载（/api/assets）:
  Dashboard 前端 → 拿到 ZBookmark.assets[] = [{id, assetType, fileName}] （无 URL 字段）
    → 浏览器发起 GET /api/assets/{assetId} （带 session cookie / Authorization 头）
      → authMiddleware (校验 ctx.user 存在)
      → apiKeyScopeMiddleware("assets", "read") （API Key 调用需 scope，session 调用跳过）
      → Asset.fromId(ctx, assetId).ensureCanView()
        → asset.userId === ctx.user.id ? ✅
        → assetType === "avatar" ? ✅
        → asset.bookmarkId 存在 ? BareBookmark.bareFromId → isAllowedToAccessBookmark → 检查该书签是否在 ctx.user 可 view 的列表中 ? ✅/❌
        → 其他 ❌
      → serveAsset()

RSS Feed 生成（token 可选）:
  RSS 阅读器 → GET /api/v1/rss/lists/:listId?token=xxx（token 可省略，省略时仅公开列表可访问）
    → unauthedMiddleware
    → List.getPublicListContents(ctx, listId, token ?? null)
      → List.getPublicList(ctx, listId, token ?? null)
        → DB: WHERE id=? AND (public=true OR (token IS NOT NULL AND rssToken=token))
        → 无 token 时退化为: WHERE id=? AND public=true（与公开列表页相同）
      → buildImpersonatingAuthedContext(ownerId)
      → listObj.getBookmarkIds()
      → Bookmark.loadMulti() → asPublicBookmark()
          → getContent(ASSET): assetUrl = 签名 URL (/api/public/assets/:id?token=xxx)  ✅
          → getContent(LINK):  只包含 url，不包含 author  ❌ (author 缺失)
          → 返回对象缺失 description  ❌
    → toRSS()
        → filter 掉 TEXT 类型  ❌ (TEXT 不可见)
        → LINK item.url = bookmark.content.url            ✅
        → ASSET item.url = publicUrl + /api/assets/:id    ❌ 误用未签名的需登录 URL！
            (丢弃了 bookmark.content.assetUrl 中的已签名 URL)
        → author = bookmark.content.author  (永远 undefined) ❌
        → description = bookmark.description  (永远 "")     ❌
        → 无 enclosure 元素  ❌
        → <channel><link> = /dashboard/lists/:id (需登录)  ❌
    → 返回 RSS XML (无 Cache-Control 头)

RSS 中 ASSET 链接的错误访问路径:
  RSS 阅读器用户点击 ASSET 链接
    → GET {publicUrl}/api/assets/:assetId (无 token)
    → authMiddleware (无登录) → HTTP 401 Unauthorized → 登录页 ✗ 失败

修复后 RSS 中 ASSET 链接的正确路径:
  RSS 阅读器用户点击 ASSET 链接
    → GET /api/public/assets/:assetId?token=xxx (使用 asPublicBookmark 已签发的 assetUrl)
    → 走上面"资产下载（正确路径）"流程 → 成功下载 ✓
```

---

## 九、关键文件索引

| 文件 | 职责 |
|------|------|
| [schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/db/schema.ts#L475-L498) | `bookmarkLists` 表定义，`public` 和 `rssToken` 字段 |
| [types/lists.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/types/lists.ts#L61) | `userRole: z.enum(["owner", "editor", "viewer", "public"])` 定义 |
| [signedTokens.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/signedTokens.ts) | `createSignedToken` / `verifySignedToken` / `getAlignedExpiry` |
| [zAssetSignedTokenSchema](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/types/assets.ts) | 资产签名 token 的 payload schema |
| [zPublicBookmarkSchema](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/types/bookmarks.ts#L283-L310) | 公开书签 schema（与实际返回存在字段差异） |
| [lists.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L85-L159) | `List.fromId()` — owner/collaborator 路由 + 角色判定入口 |
| [lists.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L161-L253) | `getPublicList()` / `getPublicListContents()` — 公开/RSS 共享入口 |
| [lists.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L397-L465) | `canUserView/Edit/Manage()` 角色矩阵 |
| [lists.ts (router)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/lists.ts#L26-L58) | `ensureListAtLeastViewer/Editor/Owner` tRPC 中间件 |
| [lists.ts (router)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/lists.ts#L204-L231) | RSS Token 签发/撤销/查询端点 |
| [bookmarks.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L100-L131) | `BareBookmark.bareFromId()` / `isAllowedToAccessBookmark()` — 书签访问控制 |
| [bookmarks.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L276-L395) | `buildDebugInfo()` — Admin 调试视图输出 |
| [bookmarks.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L769-L861) | `asPublicBookmark()` — 公开数据脱敏（含缺失字段问题） |
| [assets.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/assets.ts#L225-L260) | `Asset.canUserView()` / `ensureCanView()` — 已登录用户资产访问控制 |
| [assets.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/assets.ts#L266-L281) | `getPublicSignedAssetUrl()` — 签名 URL 生成 |
| [public/assets.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/public/assets.ts) | 公开资产下载端点 + token 校验（匿名） |
| [assets.ts (route)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/assets.ts) | 已登录资产下载端点（需 authMiddleware） |
| [api/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/index.ts#L86-L93) | 路由挂载点（/assets 与 /public/assets 并行） |
| [trpc/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/index.ts#L131-L227) | `authedProcedure` / `createScopedAuthedProcedure` / `createAdminScopedProcedure` 定义 |
| [admin.ts (router)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/admin.ts#L708-L787) | `admin.getBookmarkDebugInfo` 路由 |
| [impersonate.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/lib/impersonate.ts) | 冒充 owner 上下文构建 |
| [publicBookmarks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/publicBookmarks.ts) | tRPC 公开列表端点 |
| [rss.ts (handler)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/rss.ts) | RSS Feed HTTP 端点 |
| [rss.ts (utils)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/utils/rss.ts) | `toRSS()` — RSS XML 生成（含 URL 误用问题） |
| [assetUtils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/utils/assetUtils.ts) | `getAssetUrl()` 生成需登录的 URL（RSS 误用的函数） |
| [serveAsset](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/utils/assets.ts) | 资产响应 + Cache-Control 头 |
| [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/middlewares/auth.ts) | `unauthedMiddleware` / `authMiddleware` |
| [PublicListLink.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/apps/web/components/dashboard/lists/PublicListLink.tsx) | 前端公开开关 UI |
| [public list page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/apps/web/app/public/lists/[listId]/page.tsx) | 公开列表 SSR 页面 |
| [PublicBookmarkGrid.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/apps/web/components/public/lists/PublicBookmarkGrid.tsx) | 公开列表前端卡片渲染（LINK/TEXT/ASSET 三种分支） |
| [config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/config.ts#L266-L271) | `signingSecret()` → `NEXTAUTH_SECRET` |
| [public.test.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/e2e_tests/tests/api/public.test.ts) | 公开 API E2E 测试（当前未覆盖 RSS / Admin） |
