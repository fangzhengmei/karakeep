# Karakeep 公开分享认证机制梳理

本文档从源码层面梳理公开分享（Public Share）在 Karakeep 中的完整运转路径，覆盖 Share Token 签发、校验、权限范围、撤销传播与预览缓存，重点标注越权与过期边界。

---

## 一、两条公开分享通道概览

Karakeep 的公开分享分为 **两套独立通道**，它们使用不同的鉴权模型：

| 通道 | 入口 | 鉴权方式 | 访问粒度 |
|------|------|----------|----------|
| 公开列表（Public List） | `GET /public/lists/[listId]` (Next.js 页面) / tRPC `publicBookmarks` | `bookmarkLists.public = true` 布尔标记 | 整个列表只读 |
| RSS Feed | `GET /api/v1/rss/lists/:listId?token=xxx` | `bookmarkLists.rssToken` 字段匹配 | 整个列表只读 |
| 公开资产（Public Asset） | `GET /api/public/assets/:assetId?token=xxx` | HMAC-SHA256 签名 Token | 单个资产文件 |

关键区别：公开列表是"谁都能看"，RSS Token 是"有令牌才能看"，公开资产是"有时效签名才能下载"。

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

## 三、公开列表 — Token 签发与权限

### 3.1 开启公开分享

**前端**: [PublicListLink.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/apps/web/components/dashboard/lists/PublicListLink.tsx#L34-L44)

前端通过 `Switch` 组件调用 `editList({ listId, public: checked })`，触发 tRPC mutation。

**后端**: [lists.ts router](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/lists.ts#L86-L102)

```ts
edit: listsProcedure
  .input(zEditBookmarkListSchemaWithValidation)
  .use(ensureListAtLeastViewer)
  .use(ensureListAtLeastOwner)        // ← 仅 owner 可操作
  .mutation(async ({ input, ctx }) => {
    await ctx.list.update(input);     // 设置 public: true/false
  })
```

`List.update()` ([lists.ts L601-L625](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L601-L625)) 通过 `ensureCanManage()` 校验权限后执行 `UPDATE bookmarkLists SET public = ? WHERE id = ? AND userId = ?`。

**权限边界**:
- 只有列表 `owner`（`userRole === "owner"`）可以开关公开状态。
- `editor` / `viewer` / `public` 角色均被拒绝（`canUserManage()` 返回 false）。

### 3.2 公开列表内容读取

**tRPC 路由**: [publicBookmarks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/publicBookmarks.ts#L14-L70)

```ts
getPublicListMetadata: publicProcedure   // ← 无需登录
  .input(z.object({ listId: z.string() }))
  .query(async ({ input, ctx }) => {
    return await List.getPublicListMetadata(ctx, input.listId, /* token */ null);
  }),

getPublicBookmarksInList: publicProcedure  // ← 无需登录
  .query(async ({ input, ctx }) => {
    return await List.getPublicListContents(ctx, input.listId, /* token */ null, { ... });
  }),
```

两个端点都使用 `publicProcedure`（仅做限流，不做身份校验），**token 传 null**。

**核心鉴权**: [List.getPublicList()](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L161-L189)

```ts
private static async getPublicList(ctx: Context, listId: string, token: string | null) {
  const listdb = await ctx.db.query.bookmarkLists.findFirst({
    where: and(
      eq(bookmarkLists.id, listId),
      or(
        eq(bookmarkLists.public, true),                              // 条件1: 公开
        token !== null ? eq(bookmarkLists.rssToken, token) : undefined, // 条件2: RSS token 匹配
      ),
    ),
  });
  if (!listdb) throw new TRPCError({ code: "NOT_FOUND" });
  return listdb;
}
```

**鉴权逻辑**: `public = true` **或** `rssToken = token`，两者满足其一即可访问。当 token 为 null 时，只有 `public = true` 的列表可见。

### 3.3 数据脱敏 — asPublicBookmark()

**文件**: [bookmarks.ts L769-L861](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L769-L861)

公开书签通过 `asPublicBookmark()` 精心裁剪，**不使用 spread**：

```ts
return {
  id: this.bookmark.id,
  createdAt: this.bookmark.createdAt,
  modifiedAt: this.bookmark.modifiedAt,
  title: getBookmarkTitle(this.bookmark),
  tags: this.bookmark.tags.map((t) => t.name),   // 只暴露标签名，不暴露标签 ID
  content: getContent(this.bookmark.content),      // 裁剪后的内容
  bannerImageUrl: getBannerImageUrl(this.bookmark.content),
};
```

**不同类型的裁剪**:

| 类型 | 暴露字段 | 隐藏字段 |
|------|----------|----------|
| LINK | `url` | `title`, `description`, `htmlContent`, `author`, `publisher`, `crawledAt` 等元数据 |
| TEXT | `text` 完整文本 | `sourceUrl` |
| ASSET | `assetType`, `assetId`, `assetUrl`(签名), `fileName`, `sourceUrl` | 无额外隐藏 |

**关键注意**: LINK 类型在公开视图中只暴露原始 URL，不暴露爬取的 htmlContent、description 等字段。TEXT 类型会暴露完整文本。

### 3.4 冒充上下文 — buildImpersonatingAuthedContext

**文件**: [impersonate.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/lib/impersonate.ts#L1-L30)

公开列表读取时需要以列表 owner 身份查询书签：

```ts
const authedCtx = await buildImpersonatingAuthedContext(listdb.userId);
```

这创建了一个 **不带 auth 信息**（`auth` 字段缺失）、`req.ip = null` 的 AuthedContext。由于该上下文只在 `List.getPublicListContents()` 内部使用，不返回给外部，安全风险可控。

**越权边界**: 如果 impersonating context 被错误传递到其他需要真实身份的操作（如写入），可能导致以 owner 身份执行未授权操作。当前代码中，该上下文只用于 `Bookmark.loadMulti()` 和 `listObj.getBookmarkIds()`，均为只读查询。

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

### 5.4 隐私过滤 — PRIVACY_REDACTED_ASSET_TYPES

**文件**: [bookmarks.ts L285-L288](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L285-L288)

```ts
const PRIVACY_REDACTED_ASSET_TYPES = new Set<AssetTypes>([
  AssetTypes.USER_UPLOADED,
  AssetTypes.BOOKMARK_ASSET,
]);
```

认证用户查看书签时，这两种类型的资产 **不生成签名 URL**（`url: null`）。但在公开书签中，资产 URL 的生成逻辑不同——`asPublicBookmark()` 只对 ASSET 类型生成 `assetUrl`（用于 PDF/图片下载），banner image 的生成不受此过滤影响。

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
| 2 | 公开列表无访问审计 | **低** | `publicProcedure` 不记录访问者信息（无 auth），无法追踪谁在何时访问了公开列表。 |
| 3 | 资产签名 URL 浏览器缓存 | **低** | `Cache-Control: private, max-age=31536000, immutable` 意味着浏览器可能长期缓存资产。即使签名过期或列表被设为非公开，已缓存内容不受影响。 |
| 4 | 公开列表开关无确认步骤 | **低** | 从非公开切换为公开是即时生效的，没有二次确认或"预览即将公开的内容"步骤。 |
| 5 | `public` 字段与 `rssToken` 独立 | **中** | 清除 RSS Token 不会关闭公开列表；关闭公开列表不会清除 RSS Token。两者是独立的访问通道，可能造成混淆。 |
| 6 | NEXTAUTH_SECRET 双重用途 | **中** | 签名 Token 与 NextAuth 共用 `NEXTAUTH_SECRET`。如果该密钥被轮换，所有已签发的资产 token 立即失效（包括未过期的），可能导致用户体验中断。 |

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

### 7.3 未覆盖的边界场景

| 场景 | 现状 |
|------|------|
| 列表从公开切为非公开后，已签发的资产 token 是否仍可访问资产？ | **可以**。资产 token 绑定的是 `assetId + userId`，不检查列表公开状态。即使列表变为非公开，只要 token 未过期，资产仍可下载。 |
| RSS Token 泄露后的窗口期 | **无限大**。除非 owner 手动轮换，泄露的 token 永久有效。 |
| 并发竞争：owner 正在关闭公开时，公开请求是否仍可通过？ | **可能**。数据库 UPDATE 和 SELECT 之间无事务隔离，存在极短的竞争窗口。 |
| 删除列表后 RSS Token 是否残留？ | **不残留**。列表删除时 `rssToken` 随行删除（`onDelete: cascade` 作用于 userId 外键，列表整行删除）。 |

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

资产下载:
  浏览器 → GET /api/public/assets/{assetId}?token=xxx
    → unauthedMiddleware (仅检查 ctx 存在)
    → verifySignedToken(token, NEXTAUTH_SECRET, zAssetSignedTokenSchema)
      → Base64 解码 → JSON 解析 → HMAC 签名验证 → 过期检查 → Zod 验证
    → tokenPayload.assetId !== URL.assetId? → 403
    → DB: SELECT * FROM assets WHERE id=? AND userId=?  → 404 if not found
    → serveAsset() → Cache-Control: private, max-age=31536000, immutable

RSS Feed:
  RSS 阅读器 → GET /api/v1/rss/lists/:listId?token=xxx
    → unauthedMiddleware
    → List.getPublicListContents(ctx, listId, token)
      → List.getPublicList(ctx, listId, token)
        → DB: WHERE id=? AND (public=true OR rssToken=token)
      → → buildImpersonatingAuthedContext → asPublicBookmark()
    → toRSS() → RSS XML
```

---

## 九、关键文件索引

| 文件 | 职责 |
|------|------|
| [schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/db/schema.ts#L475-L498) | `bookmarkLists` 表定义，`public` 和 `rssToken` 字段 |
| [signedTokens.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/signedTokens.ts) | `createSignedToken` / `verifySignedToken` / `getAlignedExpiry` |
| [zAssetSignedTokenSchema](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/types/assets.ts) | 资产签名 token 的 payload schema |
| [lists.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/lists.ts#L161-L189) | `getPublicList()` 核心鉴权逻辑 |
| [lists.ts (router)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/lists.ts#L204-L231) | RSS Token 签发/撤销/查询端点 |
| [bookmarks.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/bookmarks.ts#L769-L861) | `asPublicBookmark()` 数据脱敏 |
| [assets.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/models/assets.ts#L266-L281) | `getPublicSignedAssetUrl()` 签名 URL 生成 |
| [public/assets.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/public/assets.ts) | 公开资产下载端点 + token 校验 |
| [impersonate.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/lib/impersonate.ts) | 冒充 owner 上下文构建 |
| [publicBookmarks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/trpc/routers/publicBookmarks.ts) | tRPC 公开列表端点 |
| [rss.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/routes/rss.ts) | RSS Feed 端点 |
| [serveAsset](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/utils/assets.ts) | 资产响应 + Cache-Control 头 |
| [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/api/middlewares/auth.ts) | `unauthedMiddleware` / `authMiddleware` |
| [PublicListLink.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/apps/web/components/dashboard/lists/PublicListLink.tsx) | 前端公开开关 UI |
| [public list page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/apps/web/app/public/lists/[listId]/page.tsx) | 公开列表 SSR 页面 |
| [config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/shared/config.ts#L266-L271) | `signingSecret()` → `NEXTAUTH_SECRET` |
| [public.test.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/42-karakeep/packages/e2e_tests/tests/api/public.test.ts) | 公开 API E2E 测试 |
