# 手机端书签详情数据流分析

## 一、整体架构概述

Karakeep 采用现代化的全栈架构，移动端使用 React Native + Expo Router，后端使用 tRPC + Drizzle ORM，实现端到端的类型安全。

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层（React Native）                   │
│  ┌────────────────┐    ┌─────────────┐    ┌─────────────────┐   │
│  │ Expo Router    │───▶│ React Query │───▶│ tRPC Client     │   │
│  │ (File-based    │    │ (TanStack)  │    │ (Type-Safe)     │   │
│  │  Routing)      │    │             │    │                 │   │
│  └────────────────┘    └─────────────┘    └─────────────────┘   │
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

## 二、三段核心流程详细分析

### 流程一：从列表点击进入详情（含默认外部浏览器分支）

#### 1.1 触发条件
- **触发时机**：用户在书签列表页点击任意 BookmarkCard
- **触发位置**：`apps/mobile/components/bookmarks/BookmarkCard.tsx:461-479`

#### 1.2 关键代码
```typescript
// BookmarkCard.tsx:461-479
const onOpenBookmark = (bookmark: ZBookmark) => {
  // 分支判断：LINK 类型且默认视图为外部浏览器
  if (
    bookmark.content.type === BookmarkTypes.LINK &&
    settings.defaultBookmarkView === "externalBrowser"
  ) {
    // 分支A：直接调用系统浏览器打开（React Native Linking API）
    void Linking.openURL(bookmark.content.url).catch(() => {
      // 兜底：外部浏览器打开失败时，仍然跳转详情页
      toast({ message: "Failed to open link", variant: "destructive" });
      router.push(`/dashboard/bookmarks/${bookmark.id}`);
    });
    return;
  }

  // 分支B：正常跳转到应用内详情页（Expo Router 文件路由）
  router.push(`/dashboard/bookmarks/${bookmark.id}`);
};
```

#### 1.3 配置来源（defaultBookmarkView）
**文件位置**：`apps/mobile/lib/settings.ts:37-63`

```typescript
// 设置 Schema 定义（Zod + SecureStore 持久化）
const zSettingsSchema = z.object({
  defaultBookmarkView: z
    .enum(["reader", "browser", "externalBrowser"])
    .optional()
    .default("reader"),  // 默认为阅读模式
  // ... 其他设置
});
```

#### 1.4 数据流分支图
```
用户点击 BookmarkCard
        │
        ▼
┌─────────────────────────────────────────┐
│ 判断 bookmark.type + defaultBookmarkView │
└───────────────┬─────────────────────────┘
                │
        ┌───────┴───────┐
        ▼               ▼
┌───────────────┐  ┌──────────────────┐
│ LINK 类型？   │  │ 其他类型（TEXT/  │
│ 且外部浏览器? │  │ ASSET）           │
└───────┬───────┘  └─────────┬────────┘
        │                    │
        ▼                    ▼
┌───────────────┐  ┌──────────────────┐
│ Linking.openURL│  │ Expo Router      │
│ (系统浏览器)   │  │ router.push(详情页) │
└───────┬───────┘  └──────────────────┘
        │
        ▼ 失败兜底
    ┌──────────┐
    │ 仍然跳转 │
    │ 详情页   │
    └──────────┘
```

#### 1.5 关键参数与渲染落点

| 参数 | 来源 | 说明 | 渲染落点 |
|------|------|------|----------|
| `bookmark.content.type` | 书签数据 | 书签类型枚举 | 分支判断条件 |
| `settings.defaultBookmarkView` | Expo SecureStore 本地存储 | 用户偏好设置 | 视图模式选择 |
| `bookmark.content.url` | 书签内容 | 网页 URL | React Native Linking API 参数 |
| `bookmark.id` | 书签元数据 | 书签唯一标识 | Expo Router 路由参数 |

---

### 流程二：详情页首次请求（获取元数据）

#### 2.1 触发条件
- **触发时机**：进入 `apps/mobile/app/dashboard/bookmarks/[slug]/index.tsx` 页面时（Expo Router 文件路由）
- **触发位置**：`index.tsx:42-57`

#### 2.2 请求参数详解
```typescript
// 详情页首次请求（React Native 页面初始化）
const { slug } = useLocalSearchParams(); // Expo Router 路由参数 Hook
const api = useTRPC();

const { data: bookmark, error, refetch } = useQuery(
  api.bookmarks.getBookmark.queryOptions({
    bookmarkId: slug,          // 路由参数：书签ID
    includeContent: false,     // ⚠️ 关键：不包含HTML内容！
  }),
);
```

#### 2.3 服务端处理链路

**步骤1：Procedure 入口**
```typescript
// packages/trpc/routers/bookmarks.ts:789-803
getBookmark: bookmarksProcedure
  .use(createBookmarksQueriedMiddleware())  // 统计查询次数
  .input(z.object({
    bookmarkId: z.string(),
    includeContent: z.boolean().optional().default(false),
  }))
  .output(zBookmarkSchema)
  .use(ensureBookmarkAccess)  // 权限验证中间件
  .query(async ({ input, ctx }) => {
    return (
      await Bookmark.fromId(ctx, input.bookmarkId, input.includeContent)
    ).asZBookmark();
  }),
```

**步骤2：权限验证中间件**
```typescript
// packages/trpc/routers/bookmarks.ts:90-106
export const ensureBookmarkAccess = experimental_trpcMiddleware()
  .create(async (opts) => {
    // 验证书签存在性
    const bookmark = await BareBookmark.bareFromId(
      opts.ctx,
      opts.input.bookmarkId,
    );
    // 验证用户权限（所有者/协作者/公开）
    // ... 内部逻辑
    return opts.next({ ctx: { ...opts.ctx, bookmark } });
  });
```

**步骤3：Bookmark Model 数据查询**
```typescript
// packages/trpc/models/bookmarks.ts:233-270
static async fromId(ctx, bookmarkId, includeContent) {
  // Drizzle ORM 关联查询
  const bookmark = await ctx.db.query.bookmarks.findFirst({
    where: eq(bookmarks.id, bookmarkId),
    with: {
      tagsOnBookmarks: { with: { tag: true } },  // 标签
      link: true,      // 链接元数据（不包含内容）
      text: true,      // 文本内容（小型）
      asset: true,     // 资产元数据
      assets: true,    // 关联资产列表
    },
  });

  // 权限验证
  if (!(await BareBookmark.isAllowedToAccessBookmark(ctx, bookmark))) {
    throw new TRPCError({ code: "NOT_FOUND" });
  }

  // 转换为 Zod Schema
  return Bookmark.fromData(
    ctx,
    await Bookmark.toZodSchema(bookmark, includeContent),
  );
}
```

#### 2.4 首次请求返回的数据结构
```typescript
// includeContent: false 时返回的元数据
{
  id: string,
  userId: string,
  type: "link" | "text" | "asset",
  title: string | null,
  favourited: boolean,
  archived: boolean,
  note: string | null,
  createdAt: Date,
  modifiedAt: Date,
  tags: [{ id, name, attachedBy, ... }],
  assets: [{ id, assetType, fileName, ... }],
  content: {
    type: "link",
    url: string,          // 网页URL
    title: string | null, // 页面标题
    description: string | null,
    favicon: string | null,
    imageUrl: string | null,
    crawledAt: Date | null,
    crawlStatus: "success" | "failure" | "pending",
    htmlContent: null,    // ⚠️ 不包含HTML内容！
    // ... 其他元数据
  }
}
```

#### 2.5 首次请求渲染落点

```typescript
// index.tsx:67-141
// 根据书签类型分发渲染组件（React Native Native Components）
switch (bookmark.content.type) {
  case BookmarkTypes.LINK:
    comp = (
      <BookmarkLinkView
        bookmark={bookmark}                  // 只包含元数据
        bookmarkPreviewType={bookmarkLinkType}  // reader/browser
      />
    );
    break;
  case BookmarkTypes.TEXT:
    comp = <BookmarkTextView bookmark={bookmark} />;
    break;
  case BookmarkTypes.ASSET:
    comp = <BookmarkAssetView bookmark={bookmark} />;
    break;
}
```

**底部操作栏渲染**：
- 所有模式下都立即渲染 BottomActions（React Native View 组件）
- 包含收藏、归档、标签管理、列表管理、分享等操作按钮
- 不依赖 HTML 内容

#### 2.6 首次请求数据流图
```
router.push("/dashboard/bookmarks/[slug]")
        │
        ▼
┌─────────────────────────────────────┐
│ Expo Router 页面初始化               │
│ useLocalSearchParams()              │
│ slug = bookmarkId                   │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ React Query: getBookmark            │
│ bookmarkId: slug                    │
│ includeContent: false               │  ⚠️ 关键！
└─────────────┬───────────────────────┘
              │
              ▼ HTTP
┌─────────────────────────────────────┐
│ tRPC Server: bookmarks.getBookmark  │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ ensureBookmarkAccess Middleware     │
│ - 验证书签存在性                     │
│ - 验证用户访问权限                   │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ Bookmark.fromId()                   │
│ Drizzle ORM 关联查询                 │
│ with: { tags, link, assets, ... }   │
│ includeContent: false               │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ 返回元数据（无HTML）                 │
│ + tags + assets + link metadata     │
└─────────────┬───────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│ React Native 渲染页面框架            │
│ - Stack Navigation 导航栏           │
│   (标题、Stack.Screen options)      │
│ - BookmarkLinkView                  │
│   → BookmarkLinkReaderPreview (待加载) │
│ - BottomActions (收藏/归档等按钮)    │
└─────────────────────────────────────┘
```

---

### 流程三：阅读模式二次请求（获取内容 + 高亮数据）

#### 3.1 触发条件
- **触发时机**：`BookmarkLinkReaderPreview` 组件挂载时（React Native 组件生命周期）
- **前置条件**：
  1. 首次请求已完成，获取了书签元数据
  2. `bookmarkLinkType === "reader"`（用户选择阅读模式，或默认）
- **触发位置**：`apps/mobile/components/bookmarks/BookmarkLinkPreview.tsx:103-246`

#### 3.2 二次请求的两个并行调用

```typescript
// BookmarkLinkPreview.tsx:112-128
export function BookmarkLinkReaderPreview({ bookmark }) {
  const api = useTRPC();

  // 调用A：获取完整HTML内容（第二次 getBookmark 调用）
  const {
    data: bookmarkWithContent,
    error,
    isLoading,
    refetch,
  } = useQuery(
    api.bookmarks.getBookmark.queryOptions({
      bookmarkId: bookmark.id,
      includeContent: true,    // ⚠️ 关键！这次要获取完整内容
    }),
  );

  // 调用B：获取书签的所有高亮数据（独立接口）
  const { data: highlights } = useQuery(
    api.highlights.getForBookmark.queryOptions({
      bookmarkId: bookmark.id,
    }),
  );

  // ... 渲染逻辑
}
```

#### 3.3 调用A - 完整内容获取（includeContent: true）

**服务端处理差异**：
```typescript
// packages/trpc/models/bookmarks.ts:151-231
private static async toZodSchema(bookmark, includeContent) {
  let content: ZBookmarkContent;
  
  if (bookmark.link) {
    content = {
      type: BookmarkTypes.LINK,
      url: link.url,
      title: link.title,
      // ... 元数据
      htmlContent: includeContent
        ? await Bookmark.getBookmarkHtmlContent(link, bookmark.userId)
        : null,  // 首次请求时这里是 null
    };
  }
  // ...
}
```

**HTML内容获取逻辑**：
```typescript
// packages/trpc/models/bookmarks.ts:862-883
static async getBookmarkHtmlContent({ contentAssetId, htmlContent }, userId) {
  if (contentAssetId) {
    // 大内容存储在资产表，从存储服务读取
    const asset = await readAsset({ userId, assetId: contentAssetId });
    return asset.asset.toString("utf8");
  } else if (htmlContent) {
    // 小内容内联在 bookmark_links 表
    return htmlContent;
  }
  return null;
}
```

#### 3.4 调用B - 高亮数据获取（getForBookmark）

**服务端 Procedure**：
```typescript
// packages/trpc/routers/highlights.ts:61-70
getForBookmark: highlightsProcedure
  .input(z.object({ bookmarkId: z.string() }))
  .output(z.object({ highlights: z.array(zHighlightSchema) }))
  .use(ensureBookmarkAccess)  // 复用书签权限验证
  .query(async ({ ctx }) => {
    const highlights = await ctx.highlightsService.getForBookmark(
      ctx.bookmark.id,
    );
    return { highlights };
  }),
```

**Highlight 数据结构**：
```typescript
// packages/shared/types/highlights.ts
{
  id: string,
  userId: string,         // 高亮创建者ID
  bookmarkId: string,
  text: string | null,    // 高亮的文本内容（可空）
  startOffset: number,    // HTML中的起始位置（文本字符偏移）
  endOffset: number,      // HTML中的结束位置（文本字符偏移）
  color: "yellow" | "red" | "green" | "blue",  // 高亮颜色枚举（默认yellow）
  note: string | null,    // 用户批注（可空）
  createdAt: Date,
}
```

**偏移值计算依据**（基于 DOM Range API）：
1. 使用 `document.createTreeWalker` 遍历 HTML 中所有文本节点（`NodeFilter.SHOW_TEXT`）
2. 累积每个文本节点的 `textContent.length`，计算每个文本节点的全局偏移量
3. 用户选中文本后，通过 `window.getSelection().getRangeAt(0)` 获取 DOM Range
4. **公式**：
   - `startOffset = 文本节点全局偏移量 + range.startOffset`
   - `endOffset = 文本节点全局偏移量 + range.endOffset`
5. 反向定位时，根据偏移量找到对应文本节点和局部位置，再用 `range.setStart()`/`setEnd()` 还原选区

#### 3.5 阅读模式渲染落点

**核心渲染组件**：`BookmarkHtmlHighlighterDom`（React Native WebView 封装）
```typescript
// BookmarkLinkPreview.tsx:209-243
<BookmarkHtmlHighlighterDom
  // 内容数据（二次请求A的结果）
  htmlContent={bookmarkWithContent.content.htmlContent ?? ""}
  contentStyle={contentStyle}
  
  // 高亮数据（二次请求B的结果）
  highlights={highlights?.highlights ?? []}
  
  // 阅读进度相关
  readingProgressOffset={readingProgressOffset}
  readingProgressAnchor={readingProgressAnchor}
  restoreReadingPosition={restorePosition}
  onSavePosition={onSavePosition}
  onScrollPositionChange={onScrollPositionChange}
  
  // 交互回调（React Native Native Events）
  onLinkPress={handleLinkPress}
  onImagePress={handleImagePress}
  onHighlight={(h) => createHighlight({ ... })}
  onUpdateHighlight={(h) => updateHighlight({ ... })}
  onDeleteHighlight={(h) => deleteHighlight({ ... })}
  
  dom={{ scrollEnabled: true }}
/>
```

#### 3.6 二次请求数据流图
```
首次请求完成（元数据已渲染，React Native 视图已挂载）
        │
        ▼
┌─────────────────────────────────────────┐
│ BookmarkLinkReaderPreview 组件挂载       │
│ (React Native Component Lifecycle)       │
└───────────────────┬─────────────────────┘
                    │
    ┌───────────────┴───────────────┐
    ▼                               ▼
┌────────────────────┐    ┌───────────────────────┐
│ 调用A: getBookmark │    │ 调用B: highlights.     │
│ includeContent: true│    │   getForBookmark      │
│ bookmarkId: xxx     │    │ bookmarkId: xxx       │
└──────────┬─────────┘    └───────────┬───────────┘
           │                           │
           ▼                           ▼
┌────────────────────┐    ┌───────────────────────┐
│ Bookmark Model     │    │ HighlightsService     │
│ getBookmarkHtmlContent │ │ getForBookmark()     │
│ - 从 assets 读取大HTML │ │ - 从 highlights表查询 │
│ - 或直接返回内联HTML   │ │   所有记录           │
└──────────┬─────────┘    └───────────┬───────────┘
           │                           │
           ▼                           ▼
┌────────────────────┐    ┌───────────────────────┐
│ 返回完整HTML内容    │    │ 返回高亮数组 []        │
│ 大小: 通常几十KB   │    │ 每个: {startOffset,   │
│ 至几MB             │    │ endOffset, color, ...}│
└──────────┬─────────┘    └───────────┬───────────┘
           │                           │
           └──────────────┬────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────┐
│ React Native 阅读模式完整渲染                     │
│ ┌─────────────────────────────────────────────┐  │
│ │ 阅读进度横幅（上次阅读到x%，View 组件）      │  │
│ └─────────────────────────────────────────────┘  │
│ ┌─────────────────────────────────────────────┐  │
│ │ BookmarkHtmlHighlighterDom                    │  │
│ │ - React Native WebView 加载完整HTML          │  │
│ │ - 注入JS高亮样式（用start/endOffset定位）      │  │
│ │ - 支持选中文本→创建新高亮                      │  │
│ │ - 阅读进度恢复/追踪                          │  │
│ └─────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## 三、完整时序图（三段流程整合）

```
┌────────────────┐                                              ┌──────────┐
│ 用户 (iOS/Android) │                                           │ 服务端   │
└────────┬───────┘                                              └─────┬────┘
         │                                                                │
         │ 1. 点击列表中 BookmarkCard（React Native Pressable）          │
         │───────────────────────────────────▶                                │
         │                                                                    │
         │    ┌──────────────────────────────────────────────────────────┐   │
         │    │ 流程一：列表→详情跳转判断                                  │   │
         │    │ ──────────────────────────────────────                   │   │
         │    │  IF (type == LINK && defaultBookmarkView == external)    │   │
         │    │    → React Native Linking.openURL(外部浏览器)             │   │
         │    │    → 失败兜底: Expo Router.push(详情页)                   │   │
         │    │  ELSE                                                     │   │
         │    │    → Expo Router.push(详情页)                             │   │
         │    └──────────────────────────────────────────────────────────┘   │
         │                                                                    │
         │ 2. 详情页加载，首次API请求                                         │
         │    (includeContent: false)                                        │
         │───────────────────────────────────▶                                │
         │                                                                    │
         │    ┌──────────────────────────────────────────────────────────┐   │
         │    │ 流程二：首次请求（元数据）                                  │   │
         │    │ ──────────────────────────────────────                   │   │
         │    │  tRPC: bookmarks.getBookmark                             │   │
         │    │    → ensureBookmarkAccess (权限验证)                      │   │
         │    │    → Bookmark.fromId (Drizzle关联查询)                    │   │
         │    │    → 返回: 元数据 + tags + assets (无HTML)                │   │
         │    └──────────────────────────────────────────────────────────┘   │
         │                                                                    │
         │ ◀───────────────────────────────────                                │
         │    返回元数据，渲染页面框架                                        │
         │                                                                    │
         │ 3. 阅读模式组件挂载，二次并行请求                                  │
         │───────────────────────────────────▶                                │
         │                                                                    │
         │    ┌──────────────────────────────────────────────────────────┐   │
         │    │ 流程三：阅读模式二次请求                                    │   │
         │    │ ──────────────────────────────────────                   │   │
         │    │  调用A: getBookmark (includeContent: true)               │   │
         │    │    → Bookmark.getBookmarkHtmlContent                      │   │
         │    │    → 从 assets 读取大HTML或内联HTML                        │   │
         │    │                                                           │   │
         │    │  调用B: highlights.getForBookmark (并行)                   │   │
         │    │    → HighlightsService.getForBookmark                     │   │
         │    │    → 查询所有高亮记录                                      │   │
         │    └──────────────────────────────────────────────────────────┘   │
         │                                                                    │
         │ ◀───────────────────────────────────                                │
         │    返回HTML内容 + 高亮数据                                          │
         │                                                                    │
         │ 4. 完整阅读模式渲染                                                │
         │    ┌─────────────────────────────────────────┐                   │
         │    │ React Native WebView 渲染                 │                   │
         │    │ - BookmarkHtmlHighlighterDom             │                   │
         │    │ - 注入高亮样式 (start/endOffset)         │                   │
         │    │ - 阅读进度恢复/追踪                      │                   │
         │    │ - 选中文本→创建高亮                      │                   │
         │    └─────────────────────────────────────────┘                   │
         │                                                                    │
```

---

## 四、关键设计决策分析

### 4.1 分层加载策略（两次 getBookmark 调用）

| 策略 | 说明 | 优势 |
|------|------|------|
| **首次请求** | `includeContent: false` | 1. 快速响应，React Native 页面秒开<br>2. 减少首屏数据传输<br>3. 非阅读模式（如截图、PDF）不需要HTML |
| **二次请求** | `includeContent: true` | 1. 按需加载，节省流量<br>2. 大HTML异步获取，不阻塞UI<br>3. React Query 缓存命中时可直接使用 |

### 4.2 高亮数据独立存储

**设计**：高亮数据不内联在书签HTML中，独立存储在 `highlights` 表

**优势**：
1. **性能**：高亮查询不依赖大HTML解析
2. **可移植性**：同一书签的高亮可跨端复用（Web/移动端）
3. **可扩展性**：可单独支持高亮搜索、导出、分享等功能
4. **并发友好**：多个用户可各自添加高亮，互不干扰

### 4.3 高亮定位设计

**偏移量设计**：使用文本字符偏移量 (`startOffset`, `endOffset`) 而非 DOM 节点定位

**核心实现逻辑**（`BookmarkHtmlHighlighter.tsx`）：
- **正向计算**（创建高亮）：TreeWalker 遍历所有文本节点，累积 `textContent.length` 计算全局偏移
- **反向定位**（还原高亮）：根据偏移量反查文本节点，计算局部偏移后用 `splitText()` 拆分 DOM 节点

**优势**：
1. 与HTML解析库解耦，跨端兼容（Web React / React Native WebView）
2. 不受前端渲染框架影响，纯 DOM API 实现
3. 支持大文档快速定位（无需解析完整 DOM，TreeWalker 高效遍历）
4. 高亮数据可序列化存储，便于数据迁移和导出

---

## 五、相关文件索引

| 层级 | 文件路径 | 核心职责 |
|------|---------|---------|
| **列表→详情跳转** | `apps/mobile/components/bookmarks/BookmarkCard.tsx:461-479` | 点击事件处理，外部浏览器分支判断 |
| **用户设置存储** | `apps/mobile/lib/settings.ts:37-63` | defaultBookmarkView 配置定义，Zod + Expo SecureStore 持久化 |
| **路由入口** | `apps/mobile/app/dashboard/bookmarks/[slug]/index.tsx` | Expo Router 页面，首次请求发起，视图分发 |
| **阅读模式组件** | `apps/mobile/components/bookmarks/BookmarkLinkPreview.tsx:103-246` | 二次请求（HTML + 高亮），阅读模式渲染入口 |
| **WebView 高亮渲染** | `packages/shared-react/components/BookmarkHtmlHighlighter.tsx` | React Native WebView 封装，JS 注入高亮，选中文本交互 |
| **高亮Hook** | `packages/shared-react/hooks/highlights.ts` | useCreateHighlight / useUpdateHighlight / useDeleteHighlight |
| **高亮tRPC** | `packages/trpc/routers/highlights.ts` | getForBookmark / create / update / delete procedures |
| **高亮Service** | `packages/trpc/models/highlights.service.ts` | 业务逻辑层 |
