# 手机端书签详情数据流分析

## 一、整体架构概述

Karakeep 采用现代化的全栈架构，前端使用 React + React Native，后端使用 tRPC + Drizzle ORM，实现端到端的类型安全。

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐     │
│  │ React Page  │───▶│ React Query │───▶│ tRPC Client     │     │
│  │ (Next.js)   │    │ (TanStack)  │    │ (Type-Safe)     │     │
│  └─────────────┘    └─────────────┘    └─────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼ HTTP
┌─────────────────────────────────────────────────────────────────┐
│                         服务端层                                   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │ tRPC Procedure  │───▶│ Bookmark Model  │───▶│ Drizzle ORM │ │
│  │ (Middleware)    │    │ (Business Logic)│    │ (Database)  │ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、数据流详细追踪

### 2.1 路由入口与参数解析

**文件位置**: `apps/mobile/app/dashboard/bookmarks/[slug]/index.tsx`

```typescript
// 1. 从路由获取 bookmarkId
const { slug } = useLocalSearchParams();

// 2. 初始化 tRPC API 客户端
const api = useTRPC();

// 3. 发起查询（注意：includeContent = false，先只获取元数据）
const { data: bookmark, error, refetch } = useQuery(
  api.bookmarks.getBookmark.queryOptions({
    bookmarkId: slug,
    includeContent: false,
  }),
);
```

**关键设计决策**:
- 使用 `includeContent: false` 实现渐进式加载，先展示基本信息
- 结合 TanStack Query 的缓存和重取机制

---

### 2.2 前端状态管理层

#### 2.2.1 TRPC Provider 配置

**文件位置**: `packages/shared-react/trpc.ts` + `apps/mobile/lib/providers.tsx`

```typescript
// 创建 TRPC 上下文
export const { TRPCProvider, useTRPC } = createTRPCContext<AppRouter>();

// 在 mobile 端的 providers 中配置
<TRPCSettingsProvider settings={settings}>
  <ReaderSettingsProvider>
    {children}
  </ReaderSettingsProvider>
</TRPCSettingsProvider>
```

#### 2.2.2 Query Hook 封装

**文件位置**: `packages/shared-react/hooks/bookmarks.ts`

```typescript
export function useAutoRefreshingBookmarkQuery(input) {
  const api = useTRPC();
  return useQuery(
    api.bookmarks.getBookmark.queryOptions(input, {
      refetchInterval: (query) => {
        const data = query.state.data;
        if (!data) return false;
        return getBookmarkRefreshInterval(data); // 根据书签状态动态决定刷新间隔
      },
    }),
  );
}
```

**智能刷新策略**:
- 爬虫进行中：短间隔刷新
- 爬虫完成/失败：停止刷新
- 内容未处理：按状态决定

---

### 2.3 服务端 tRPC 层

#### 2.3.1 Procedure 定义与中间件

**文件位置**: `packages/trpc/routers/bookmarks.ts`

```typescript
export const bookmarksAppRouter = router({
  getBookmark: bookmarksProcedure
    .use(createBookmarksQueriedMiddleware())  // 统计查询次数
    .input(z.object({
      bookmarkId: z.string(),
      includeContent: z.boolean().optional().default(false),
    }))
    .output(zBookmarkSchema)
    .use(ensureBookmarkAccess)  // 权限检查中间件
    .query(async ({ input, ctx }) => {
      return (
        await Bookmark.fromId(ctx, input.bookmarkId, input.includeContent)
      ).asZBookmark();
    }),
});
```

#### 2.3.2 权限检查中间件

```typescript
export const ensureBookmarkAccess = experimental_trpcMiddleware<{
  ctx: AuthedContext;
  input: { bookmarkId: string };
}>().create(async (opts) => {
  // 1. 首先验证书签存在性
  const bookmark = await BareBookmark.bareFromId(
    opts.ctx,
    opts.input.bookmarkId,
  );
  
  // 2. 检查访问权限（所有者或列表协作者）
  // BareBookmark.isAllowedToAccessBookmark() 内部处理
  
  return opts.next({
    ctx: { ...opts.ctx, bookmark },
  });
});
```

---

### 2.4 领域模型层 (Bookmark Model)

**文件位置**: `packages/trpc/models/bookmarks.ts`

#### 2.4.1 类结构设计

```typescript
// 轻量级基类 - 仅用于权限检查
export class BareBookmark {
  static async bareFromId(ctx: AuthedContext, bookmarkId: string);
  ensureOwnership();
  protected static async isAllowedToAccessBookmark(...);
}

// 完整书签类 - 包含完整业务逻辑
export class Bookmark extends BareBookmark {
  static async fromId(ctx, bookmarkId, includeContent);
  static async loadMulti(ctx, input);       // 批量加载
  static async getBookmarkHtmlContent(...); // 获取HTML内容
  static async getBookmarkPlainTextContent(...);
  asZBookmark();     // 转换为 Zod schema
  asPublicBookmark(); // 转换为公开API格式
  async delete();    // 删除书签
}
```

#### 2.4.2 单条书签查询实现

```typescript
static async fromId(ctx, bookmarkId, includeContent) {
  // 1. Drizzle 查询 - 使用关联加载
  const bookmark = await ctx.db.query.bookmarks.findFirst({
    where: eq(bookmarks.id, bookmarkId),
    with: {
      tagsOnBookmarks: { with: { tag: true } },
      link: true,
      text: true,
      asset: true,
      assets: true,
    },
  });

  // 2. 权限验证
  if (!(await BareBookmark.isAllowedToAccessBookmark(ctx, bookmark))) {
    throw new TRPCError({ code: "NOT_FOUND" });
  }

  // 3. 转换为 Zod Schema（处理内容加载）
  return Bookmark.fromData(
    ctx,
    await Bookmark.toZodSchema(bookmark, includeContent),
  );
}
```

#### 2.4.3 内容类型转换逻辑

```typescript
private static async toZodSchema(bookmark, includeContent): Promise<ZBookmark> {
  let content: ZBookmarkContent;
  
  if (bookmark.link) {
    content = {
      type: BookmarkTypes.LINK,
      url: link.url,
      title: link.title,
      htmlContent: includeContent
        ? await Bookmark.getBookmarkHtmlContent(link, bookmark.userId)
        : null,
      // ... 其他字段
    };
  } else if (bookmark.text) {
    content = { type: BookmarkTypes.TEXT, text: text.text };
  } else if (bookmark.asset) {
    content = { type: BookmarkTypes.ASSET, ... };
  }

  return { tags: [...], content, assets: [...], ...rest };
}
```

---

### 2.5 数据库层 (Drizzle ORM)

#### 2.5.1 表结构设计要点

```typescript
// bookmarks 主表
export const bookmarks = pgTable("bookmarks", {
  id: text("id").primaryKey(),
  userId: text("user_id").notNull(),
  type: bookmarkType("type").notNull(),
  title: text("title"),
  favourited: boolean("favourited").default(false),
  archived: boolean("archived").default(false),
  note: text("note"),
  summarizationStatus: summarizationStatus("summarization_status"),
  // ...
});

// 各类型内容分表（1:1 关系）
export const bookmarkLinks = pgTable("bookmark_links", { ... });   // 链接类型
export const bookmarkTexts = pgTable("bookmark_texts", { ... });   // 文本类型
export const bookmarkAssets = pgTable("bookmark_assets", { ... }); // 资产类型

// 多对多关系表
export const tagsOnBookmarks = pgTable("tags_on_bookmarks", { ... });
export const bookmarksInLists = pgTable("bookmarks_in_lists", { ... });
```

#### 2.5.2 批量加载优化策略 (loadMulti)

**文件位置**: `packages/trpc/models/bookmarks.ts:400-751`

```typescript
static async loadMulti(ctx, input) {
  // 1. 根据过滤条件选择查询策略
  // - 按列表过滤: bookmarksInLists JOIN bookmarks
  // - 按标签过滤: tagsOnBookmarks JOIN bookmarks
  // - 按 RSS Feed 过滤: rssFeedImportsTable JOIN bookmarks
  // - 无过滤: 直接查询 bookmarks 表

  // 2. 使用 CTE (Common Table Expression) 优化
  const sq = ctx.db.$with("bookmarksSq").as(...);

  // 3. 单次查询 + JOIN 加载所有关联数据
  const results = await ctx.db
    .with(sq)
    .select()
    .from(sq)
    .leftJoin(tagsOnBookmarks, ...)
    .leftJoin(bookmarkTags, ...)
    .leftJoin(bookmarkLinks, ...)
    .leftJoin(bookmarkTexts, ...)
    .leftJoin(bookmarkAssets, ...)
    .leftJoin(assets, ...);

  // 4. 客户端聚合（Reduce）
  const bookmarksRes = results.reduce((acc, row) => {
    // 合并重复行（因 JOIN 产生的笛卡尔积）
    // 聚合 tags 和 assets
    // 处理大内容的延迟加载
  }, {});

  // 5. 异步加载大内容（HTML from assets）
  if (input.includeContent) {
    await Promise.all(bookmarksArr.map(async (bookmark) => {
      if (bookmark.content.contentAssetId) {
        const asset = await readAsset({ userId, assetId });
        bookmark.content.htmlContent = asset.asset.toString("utf8");
      }
    }));
  }
}
```

---

### 2.6 前端渲染层

#### 2.6.1 主页面渲染流程

```typescript
// apps/mobile/app/dashboard/bookmarks/[slug]/index.tsx

export default function BookmarkView() {
  // 1. 加载状态 → 显示 Spinner
  if (!bookmark) return <FullPageSpinner />;
  
  // 2. 错误状态 → 显示错误页 + 重试按钮
  if (error) return <FullPageError error={error.message} onRetry={refetch} />;

  // 3. 根据内容类型分发渲染
  let comp;
  switch (bookmark.content.type) {
    case BookmarkTypes.LINK:
      comp = <BookmarkLinkView bookmark={bookmark} bookmarkPreviewType={...} />;
      break;
    case BookmarkTypes.TEXT:
      comp = <BookmarkTextView bookmark={bookmark} />;
      break;
    case BookmarkTypes.ASSET:
      comp = <BookmarkAssetView bookmark={bookmark} />;
      break;
  }

  return (
    <KeyboardAvoidingView>
      <Stack.Screen options={...} />  // 导航栏配置
      {comp}                          // 内容渲染
      <BottomActions bookmark={bookmark} />  // 底部操作栏
    </KeyboardAvoidingView>
  );
}
```

#### 2.6.2 链接类型书签渲染分发

```typescript
// apps/mobile/components/bookmarks/BookmarkLinkView.tsx

export default function BookmarkLinkView({ bookmark, bookmarkPreviewType }) {
  switch (bookmarkPreviewType) {
    case "browser":    // 内置浏览器
      return <BookmarkLinkBrowserPreview bookmark={bookmark} />;
    case "reader":     // 阅读器模式
      return <BookmarkLinkReaderPreview bookmark={bookmark} />;
    case "screenshot": // 截图预览
      return <BookmarkLinkScreenshotPreview bookmark={bookmark} />;
    case "archive":    // 归档页面
      return <BookmarkLinkArchivePreview bookmark={bookmark} />;
    case "pdf":        // PDF 预览
      return <BookmarkLinkPdfPreview bookmark={bookmark} />;
  }
}
```

#### 2.6.3 底部操作栏 (BottomActions)

**文件位置**: `apps/mobile/components/bookmarks/BottomActions.tsx`

**核心功能**:
- 使用 `useToolbarActions` hook 管理所有操作
- 支持可配置的工具栏（用户自定义显示哪些按钮）
- 集成乐观更新：点击收藏/归档后立即更新 UI，然后后台同步
- 提供玻璃态效果（iOS 18+ GlassView）
- 菜单项包括：列表管理、标签管理、信息查看、收藏/取消收藏、归档、浏览器打开、分享、删除

---

## 三、完整数据流时序图

```
用户点击书签卡片
        │
        ▼
  ┌─────────────┐
  │ Expo Router │
  │ /[slug]     │
  └──────┬──────┘
         │ bookmarkId = slug
         ▼
  ┌─────────────────┐
  │ useQuery()      │  ◄─── TanStack Query
  │ (React-Native)  │       - 缓存管理
  └──────┬──────────┘       - 重试逻辑
         │
         ▼ HTTP Request
  ┌─────────────────────────────────────┐
  │ tRPC Client                         │
  │ @karakeep/shared-react/trpc.ts      │
  └───────────────┬─────────────────────┘
                  │
                  ▼
  ┌─────────────────────────────────────┐
  │ tRPC Server Procedure               │
  │ packages/trpc/routers/bookmarks.ts  │
  └───────────────┬─────────────────────┘
                  │
     ┌────────────┴────────────┐
     │  ensureBookmarkAccess   │  权限验证
     │  Middleware             │
     └────────────┬────────────┘
                  │
                  ▼
  ┌─────────────────────────────────────┐
  │ Bookmark Model Layer                │
  │ packages/trpc/models/bookmarks.ts   │
  │  - fromId()                         │
  │  - toZodSchema()                    │
  │  - getBookmarkHtmlContent()         │
  └───────────────┬─────────────────────┘
                  │
                  ▼
  ┌─────────────────────────────────────┐
  │ Drizzle ORM                         │
  │  - query.bookmarks.findFirst()      │
  │  - with relational loading          │
  └───────────────┬─────────────────────┘
                  │
                  ▼ Database
     ┌─────────────────────────────┐
     │ PostgreSQL / SQLite         │
     │  - bookmarks                │
     │  - bookmark_links/texts/assets │
     │  - tags_on_bookmarks        │
     │  - assets                   │
     └─────────────────────────────┘
                  │
     ┌────────────┘
     ▼
  ┌─────────────────────────────────────┐
  │ Optional: readAsset()               │
  │  (for large HTML content stored as  │
  │   asset files, not inline)          │
  └───────────────┬─────────────────────┘
                  │
                  ▼ Response
  ┌─────────────────────────────────────┐
  │ ZBookmark Schema                    │
  │  - type-safe validation             │
  │  - polymorphic content field        │
  └───────────────┬─────────────────────┘
                  │
                  ▼
  ┌─────────────────────────────────────┐
  │ Frontend Rendering                  │
  │  - BookmarkLinkView / TextView      │
  │  - AssetView                        │
  │  - BottomActions                    │
  │  - Reader / Browser modes           │
  └─────────────────────────────────────┘
```

---

## 四、关键设计模式与技术亮点

### 4.1 领域驱动设计 (DDD)

**模式**: Rich Domain Model
- `Bookmark` 类封装业务逻辑，不仅是数据容器
- 方法如 `delete()`, `asPublicBookmark()`, `getBookmarkHtmlContent()`
- 继承关系 `BareBookmark → Bookmark` 体现职责分层

**位置**: `packages/trpc/models/bookmarks.ts`

### 4.2 中间件组合模式

**模式**: tRPC Middleware Pipeline
```
bookmarksProcedure
  ├─ createRateLimitMiddleware()     // 限流
  ├─ createEventLogMiddleware()      // 事件日志
  ├─ createBookmarksQueriedMiddleware() // 统计
  └─ ensureBookmarkAccess / ensureBookmarkOwnership  // 权限
```

### 4.3 渐进式内容加载

**模式**: Lazy Loading + Content Negotiation
- `includeContent: boolean` 参数控制是否加载大内容
- `contentAssetId` vs inline `htmlContent` 两种存储方式
- 小内容内联，大内容存 asset 表延迟加载

### 4.4 响应式查询优化

**模式**: Smart Refetch + Caching
```typescript
refetchInterval: (query) => {
  const data = query.state.data;
  if (!data) return false;
  return getBookmarkRefreshInterval(data);  // 根据状态动态调整
};
```

- 爬虫进行中：频繁刷新
- 爬虫完成：停止刷新，缓存有效
- 错误状态：指数退避

### 4.5 乐观更新策略

**位置**: `packages/shared-react/hooks/bookmarks.ts`

在 `useUpdateBookmark`, `useDeleteBookmark` 等 hooks 中：
1. 立即更新本地缓存（queryClient.setQueryData）
2. 后台发起 API 请求
3. 成功：触发相关查询失效（invalidateQueries）
4. 失败：回滚本地状态 + 提示错误

---

## 五、权限模型详解

### 5.1 访问层级

```
┌─────────────────────────────────────────────────────────┐
│  Owner (所有者)                                          │
│  ├─ 可以执行所有操作                                      │
│  ├─ 查看 favourites/archived 状态                         │
│  └─ 查看个人 note                                         │
└─────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│  Collaborator (协作者)                                   │
│  ├─ 通过列表共享获得访问                                   │
│  ├─ 无法查看 favourites/archived（始终为 false）            │
│  └─ 无法查看个人 note（始终为 null）                        │
└─────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│  Public (公开访问)                                       │
│  └─ asPublicBookmark() 进一步过滤字段                      │
└─────────────────────────────────────────────────────────┘
```

### 5.2 权限过滤实现

```typescript
asZBookmark(): ZBookmark {
  if (this.bookmark.userId === this.ctx.user.id) {
    return this.bookmark;  // 所有者：完整数据
  }
  
  // 协作者：隐藏敏感字段
  return {
    ...this.bookmark,
    archived: false,    // 始终 false
    favourited: false,  // 始终 false
    note: null,         // 始终 null
  };
}
```

---

## 六、性能优化点分析

| 优化策略 | 实现位置 | 说明 |
|---------|---------|------|
| **CTE + 单次 JOIN 查询** | `Bookmark.loadMulti()` | 避免 N+1 查询，单次获取所有关联数据 |
| **内容分表存储** | `bookmark_links.contentAssetId` | HTML 超过阈值存 asset 表，减少主表体积 |
| **includeContent 参数** | `getBookmark` procedure | 按需加载，列表页不加载完整内容 |
| **React Query 缓存** | 前端 hooks | 相同 bookmarkId 共享缓存，避免重复请求 |
| **智能刷新间隔** | `useAutoRefreshingBookmarkQuery` | 只在需要时刷新（如爬虫进行中） |
| **数据库索引设计** | Drizzle schema | userId + createdAt 复合索引，支持分页排序 |
| **客户端 Reduce 聚合** | `loadMulti()` | 利用数据库 JOIN，客户端单次循环聚合结果 |

---

## 七、代码质量与可维护性

### 7.1 类型安全贯穿全链路

```
Zod Schema 定义
    │
    ▼
tRPC Procedure Input/Output
    │
    ▼
Drizzle Query 类型推断
    │
    ▼
Model 层 TypeScript 类型
    │
    ▼
前端 Hook 返回类型
    │
    ▼
React Props 类型检查
```

### 7.2 代码组织原则

- **按功能分包**：`packages/trpc/routers/` vs `packages/trpc/models/`
- **共享代码提升复用**：`@karakeep/shared*` 包系列
- **关注点分离**：API 路由（Procedure）≠ 业务逻辑（Model）≠ 数据库操作（ORM）

### 7.3 可测试性

- 纯函数优先：`toZodSchema`, `getBookmarkRefreshInterval`
- 依赖注入：`ctx` 参数传入数据库、用户信息等
- 中间件可单独测试：`ensureBookmarkAccess`

---

## 八、相关文件索引

| 层级 | 文件路径 | 核心职责 |
|------|---------|---------|
| **前端页面** | `apps/mobile/app/dashboard/bookmarks/[slug]/index.tsx` | 详情页入口，路由参数，状态管理 |
| **前端组件** | `apps/mobile/components/bookmarks/*.tsx` | 各类型书签渲染，BottomActions |
| **前端 Hooks** | `packages/shared-react/hooks/bookmarks.ts` | 封装 React Query，乐观更新 |
| **TRPC Client** | `packages/shared-react/trpc.ts` | tRPC 上下文创建 |
| **TRPC Router** | `packages/trpc/routers/bookmarks.ts` | API 端点定义，中间件 |
| **领域模型** | `packages/trpc/models/bookmarks.ts` | Bookmark/BareBookmark 类，业务逻辑 |
| **数据库 Schema** | `packages/db/schema/*` | 表结构定义，关系映射 |
| **类型定义** | `packages/shared/types/bookmarks.ts` | Zod Schemas，共享类型 |

---

## 九、总结

Karakeep 书签详情数据流展现了一个精心设计的现代化全栈应用架构：

1. **端到端类型安全**：从数据库到前端 UI，类型全程贯穿
2. **分层架构清晰**：路由 → Query → tRPC → Model → DB，职责分明
3. **性能考虑周全**：渐进式加载、批量优化、智能缓存
4. **权限设计严谨**：Owner/Collaborator/Public 三层访问控制
5. **用户体验优先**：乐观更新、加载状态、错误处理、智能刷新

该架构既保证了开发效率（类型安全、代码复用），又兼顾了运行时性能和可维护性，是 React + tRPC + Drizzle 技术栈的优秀实践范例。
