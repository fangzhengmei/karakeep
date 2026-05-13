# Karakeep 公开只读链路端到端分析报告

## 一、Worker 参与性分析

### 1.1 Worker 在公开只读链路中的角色

**结论：Worker 不直接参与公开只读视图的读取链路**

### 1.2 证据支持

通过代码检索分析 (`apps/workers/` 目录)，Worker 的职责范围仅限于：

| Worker 类型 | 主要职责 | 队列名称 | 与公开读取相关度 |
|------------|---------|---------|----------------|
| **Crawler Worker** | 抓取网页内容、生成截图、PDF | `LinkCrawlerQueue`, `LowPriorityCrawlerQueue` | ❌ 无直接关联 |
| **Inference Worker** | AI 标签生成、内容摘要 | `OpenAIQueue` | ❌ 无直接关联 |
| **Search Worker** | 搜索引擎索引更新 | `SearchIndexingQueue` | ❌ 无直接关联 |
| **Rule Engine Worker** | 自动化规则执行 | `RuleEngineQueue` | ❌ 无直接关联 |
| **Asset Worker** | 资源预处理（缩略图等） | `AssetPreprocessingQueue` | ❌ 无直接关联 |
| **Feed Worker** | RSS 源同步 | `FeedQueue` | ❌ 无直接关联 |
| **Webhook Worker** | Webhook 事件通知 | `WebhookQueue` | ❌ 无直接关联 |
| **Backup Worker** | 数据备份 | `BackupQueue` | ❌ 无直接关联 |

**关键证据**：
- Worker 代码中未检索到任何 `public` 或公开列表相关逻辑
- 所有队列均为**写操作/后台任务**触发，与读取路径无关
- 公开列表查询直接走数据库查询路径，不经过队列层

### 1.3 Worker 的间接影响

虽然 Worker 不直接参与读取链路，但会影响公开视图的**数据质量**：
1. **资源生成**：Asset Worker 生成的缩略图通过签名 URL 在公开视图中展示
2. **内容抓取**：Crawler Worker 抓取的内容通过 `asPublicBookmark()` 裁剪后暴露
3. **索引更新**：Search Worker 不影响公开列表（公开列表不使用搜索功能）

---

## 二、端到端时序分析

### 2.1 公开只读访问完整时序图

```
┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Dashboard   │    │  Shared Hooks    │    │  TRPC API Layer  │    │    Database      │
│  (Frontend)  │    │  (React Query)   │    │   (tRPC Router)  │    │   (SQLite/Drizzle)│
└──────┬───────┘    └────────┬─────────┘    └────────┬─────────┘    └────────┬─────────┘
       │                     │                        │                       │
       │  1. 访问 /public/lists/[listId]             │                       │
       ├─────────────────────┼────────────────────────┼───────────────────────▶
       │                     │                        │         2. Server Component 渲染
       │                     │                        │   (Next.js SSR - page.tsx:40-83)
       │                     │                        │                       │
       │                     │                        │  3. 调用 getPublicListMetadata
       │                     │                        ├───────────────────────▶
       │                     │                        │      4. 查询 bookmark_lists 表
       │                     │                        │     (lists.ts:161-189)
       │                     │                        │  ◀───────────────────────
       │                     │                        │      5. 验证 public=true
       │                     │                        │         失败 → NOT_FOUND
       │                     │                        │                       │
       │  6. 返回列表元数据  │                        │                       │
       │  ◀──────────────────┼────────────────────────┤                       │
       │                     │                        │                       │
       │  7. 调用 getPublicBookmarksInList           │                       │
       │  ───────────────────┼────────────────────────┼───────────────────────▶
       │                     │                        │   8. buildImpersonatingAuthedContext
       │                     │                        │      (lists.ts:221)   │
       │                     │                        │                       │
       │                     │                        │  9. List.getBookmarkIds()
       │                     │                        │  ◀───────────────────────
       │                     │                        │ 10. Bookmark.loadMulti()
       │                     │                        │  ◀───────────────────────
       │                     │                        │                       │
       │                     │                        │ 11. 字段裁剪循环:
       │                     │                        │     - asPublicBookmark()
       │                     │                        │     - getPublicSignedAssetUrl()
       │                     │                        │                       │
       │ 12. 返回裁剪数据    │                        │                       │
       │  ◀──────────────────┼────────────────────────┤                       │
       │                     │                        │                       │
       │ 13. PublicBookmarkGrid 渲染                  │                       │
       │  ◀──────────────────┼────────────────────────┤                       │
       │                     │                        │                       │
       ────┐                 │                        │                       │
           │  14. 用户切换公开状态 (所有者操作)       │                       │
       ────┼─────────────────┼────────────────────────┼───────────────────────▶
           │                 │                        │                       │
           │                 │ 15. useEditBookmarkList()                     │
           │                 │ (lists.ts:30-57)      │                       │
           │                 │                        │                       │
           │                 │ 16. onSuccess → invalidateQueries            │
           │                 │ ◀──────────────────────┤                       │
           │                 │                        │                       │
       ────┘                 │                        │                       │
```

---

## 三、各层级详细实现

### 3.1 Dashboard 层 (公开访问入口)

**文件**：`apps/web/app/public/lists/[listId]/page.tsx`

#### 身份识别
- **匿名访问**：无用户 session，不依赖认证
- **服务端组件**：直接调用 tRPC API，不经过客户端 auth middleware
- **边界处理**：捕获 `TRPCError` → 渲染 404 页面，不暴露列表存在性

#### 字段裁剪
- 仅使用公开字段：`name`、`description`、`icon`、`numItems`、`ownerName`
- 专用组件：`PublicListHeader`、`PublicBookmarkGrid`，无私有状态渲染逻辑

### 3.2 Dashboard 层 (所有者共享管理)

**文件**：`apps/web/components/dashboard/lists/PublicListLink.tsx`

#### 身份前置条件
- 仅列表所有者可见：父组件传入的 `list` 必须包含完整权限字段
- 禁用状态：`demoMode` 下开关禁用，防止演示数据修改

#### 写入阻断
```tsx
// 仅暴露一个 mutation：editList (设置 public 字段)
// 无其他写入操作暴露给公开视图
onCheckedChange={(checked) => {
  editList({ listId: list.id, public: checked });
}}
```

### 3.3 Shared Hooks 层 (共享状态管理)

**文件**：`packages/shared-react/hooks/lists.ts`

#### 身份识别
- 依赖上层 `useTRPC()` 提供的 authenticated client
- 无 hook 层的权限重校验，依赖后端返回的权限字段

#### 字段裁剪
- 客户端层面不裁剪，依赖后端返回的 `ZBookmarkList` schema 类型约束
- TypeScript 类型系统确保敏感字段不被意外访问（如 `rssToken` 不在返回类型中）

#### 写入阻断
- 所有写入 hooks (`useAddBookmarkToList`, `useRemoveBookmarkFromList` 等) 仅在 authenticated context 中可用
- 公开页面不导入/使用这些 hooks（`PublicListLink` 仅在 dashboard 内部使用）

#### 缓存一致性实现
```typescript
// useEditBookmarkList onSuccess 回调 (lists.ts:38-53)
onSuccess: (res, req, meta, context) => {
  queryClient.invalidateQueries(api.lists.list.pathFilter());    // 列表列表
  queryClient.invalidateQueries(
    api.lists.get.queryFilter({ listId: req.listId })             // 当前列表详情
  );
  if (res.type === "smart") {
    // 智能列表内容可能变化，额外失效书签缓存
    queryClient.invalidateQueries(api.bookmarks.getBookmarks.queryFilter({ listId: req.listId }));
  }
}
```

**延迟失效策略** (`query-invalidation.ts:27-78`)：
```typescript
// scheduleInvalidateQueries 实现防抖
// - 默认延迟：250ms
// - 最大等待：3000ms
// - 相同 filter 合并，避免短时间内多次重复请求
```

### 3.4 TRPC API 层 (核心权限控制)

#### 3.4.1 公开路由 (无认证)
**文件**：`packages/trpc/routers/publicBookmarks.ts`

**身份识别**：
- 使用 `publicProcedure`：无 `ctx.user`
- 仅通过 `listId` + `public=true` 条件验证访问权限

```typescript
// 身份验证核心逻辑 (lists.ts:161-189)
private static async getPublicList(ctx: Context, listId: string, token: string | null) {
  const listdb = await ctx.db.query.bookmarkLists.findFirst({
    where: and(
      eq(bookmarkLists.id, listId),
      or(
        eq(bookmarkLists.public, true),           // 主条件：公开标记
        token !== null ? eq(bookmarkLists.rssToken, token) : undefined,  // 备选：RSS token
      ),
    ),
    with: { user: { columns: { name: true } } },  // 仅加载所有者名称
  });
  if (!listdb) throw new TRPCError({ code: "NOT_FOUND" });  // 统一404，不区分不存在/无权限
  return listdb;
}
```

**字段裁剪 - 列表层面**：
```typescript
// getPublicListMetadata (lists.ts:191-204)
return {
  userId: listdb.userId,      // 保留：用于资产签名验证
  name: listdb.name,          // 保留：显示用
  description: listdb.description,  // 保留：显示用
  icon: listdb.icon,          // 保留：显示用
  ownerName: listdb.user.name, // 保留：归属展示
  // 裁剪：parentId, rssToken, type, query, hasCollaborators 等全部丢弃
};
```

**字段裁剪 - 书签层面**：
```typescript
// asPublicBookmark() (bookmarks.ts:768-860)
// 警告：明确禁止 spread 操作，防止字段泄露
return {
  id: this.bookmark.id,
  createdAt: this.bookmark.createdAt,
  modifiedAt: this.bookmark.modifiedAt,
  title: getBookmarkTitle(this.bookmark),
  tags: this.bookmark.tags.map((t) => t.name),  // 仅保留标签名，丢弃 attachedBy
  content: getContent(this.bookmark.content),    // 按类型深度裁剪
  bannerImageUrl: getBannerImageUrl(...),        // 带签名的资源URL
  // 裁剪：archived, favourited, note, userId, content.htmlContent 等
};
```

**内容类型裁剪策略**：
| 类型 | 保留字段 | 丢弃字段 |
|-----|---------|---------|
| **Link** | `type`, `url` | `htmlContent`, `author`, `publisher`, `datePublished`, `imageUrl`, `favicon` |
| **Text** | `type`, `text` | `sourceUrl` (？需确认) |
| **Asset** | `type`, `assetType`, `assetUrl` (签名), `fileName`, `sourceUrl` | 原始二进制内容 |

**写入阻断**：
- 路由仅定义 `query` 端点，无 `mutation` 端点
- `publicProcedure` 不携带用户身份，无法执行写入

#### 3.4.2 私有路由 (认证保护)
**文件**：`packages/trpc/routers/lists.ts`

**三层中间件防护**：
```
listsProcedure (authenticated + scope check)
    ↓
ensureListAtLeastViewer (验证查看权限)
    ↓
ensureListAtLeastEditor / ensureListAtLeastOwner (验证写入权限)
```

**权限断言** (`lists.ts:434-464`)：
```typescript
ensureCanEdit(): void {
  if (!this.canUserEdit()) {
    throw new TRPCError({
      code: "FORBIDDEN",
      message: "User is not allowed to edit this list",
    });
  }
}
```

**模拟身份机制** (`lists.ts:221`)：
```typescript
// 公开列表读取时使用：创建一个仅限读取的模拟上下文
// 该上下文不对应真实用户，仅用于绕过 List 模型的权限检查
// 关键：此上下文仅在 getPublicListContents 内部使用，不对外暴露
const authedCtx = await buildImpersonatingAuthedContext(listdb.userId);
```

### 3.5 数据库层 (Schema 约束)

**文件**：`packages/db/schema.ts`

#### 身份识别相关字段
```sql
-- bookmarkLists 表 (schema.ts 中定义)
public: boolean           -- 公开标记，核心权限字段
userId: text (FK)         -- 所有者ID
rssToken: text            -- RSS 访问 token (备选公开路径)
parentId: text (FK)       -- 父列表ID (嵌套列表继承权限？)
```

```sql
-- listCollaborators 表
userId: text (FK)         -- 协作用户ID
listId: text (FK)         -- 关联列表ID
role: text ['viewer', 'editor']  -- 协作角色
addedAt: timestamp        -- 添加时间
```

#### 写入阻断的数据库保障
- 外键约束：`bookmarkLists.userId` 引用 `users.id`，防止越权创建
- 应用层查询始终加入 `userId = ctx.user.id` 条件
  ```typescript
  // 示例：列表查询始终过滤 userId
  where: and(eq(bookmarkLists.id, id), eq(bookmarkLists.userId, ctx.user.id))
  ```

#### 字段裁剪的数据库层配合
- 查询时通过 `columns` 选项精确控制加载字段
  ```typescript
  // 示例：不加载 rssToken 到上下文
  columns: { rssToken: false }
  ```
- 关联查询时限制关联对象的字段（如 `user: { columns: { name: true } }`）

---

## 四、缓存一致性深度分析

### 4.1 缓存分层架构

```
┌──────────────────────────────────────────────────────────┐
│                    Client Cache Layer                     │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  React Query Cache                                   │  │
│  │  - lists.list              (列表树缓存)             │  │
│  │  - lists.get               (单列表详情缓存)         │  │
│  │  - bookmarks.getBookmarks  (列表书签缓存)           │  │
│  │  - lists.stats             (统计数据缓存)           │  │
│  └─────────────────────────────────────────────────────┘  │
│                              ↓ invalidateQueries           │
┌──────────────────────────────────────────────────────────┐
│                    API Layer                              │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  tRPC Query Cache (如果启用)                        │  │
│  └─────────────────────────────────────────────────────┘  │
│                              ↓ DB Query                    │
┌──────────────────────────────────────────────────────────┐
│                    Database Layer                         │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  SQLite Query Planner Cache (内置)                   │  │
│  └─────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 4.2 失效触发场景矩阵

| 操作 | 触发点 | 失效的 Query Key | 实现位置 |
|-----|--------|-----------------|---------|
| **编辑列表** (含公开状态) | `editList` onSuccess | `lists.list`, `lists.get` | `lists.ts:38-44` |
| **智能列表编辑** | 同上 + type 检查 | 额外失效 `bookmarks.getBookmarks` | `lists.ts:45-51` |
| **添加书签到列表** | `addToList` onSuccess | `bookmarks.getBookmarks`, `lists.getListsOfBookmark`, `lists.stats` | `lists.ts:93-108` |
| **从列表移除书签** | `removeFromList` onSuccess | 同上 | `lists.ts:124-139` |
| **删除列表** | `deleteList` onSuccess | `lists.list`, `lists.get` (remove) | `lists.ts:154-158` |

### 4.3 公开页面的缓存特殊性

- **Server Components**：公开页面使用 Next.js SSR，不经过 React Query 客户端缓存
- **缓存策略**：依赖 Next.js 内置的渲染缓存（如果启用）
- **一致性影响**：所有者切换公开状态后，访问者可能需要刷新页面看到变更
  - 原因：SSR 输出可能被 CDN/浏览器缓存
  - 缓解：无内置失效机制，依赖 HTTP 缓存头控制

---

## 五、异常处理与错误边界

### 5.1 错误代码统一策略

| 场景 | HTTP 代码 | 设计意图 | 实现位置 |
|-----|----------|---------|---------|
| **列表不存在** | `NOT_FOUND` (404) | 不暴露存在性，防止枚举 | `lists.ts:149-153`, `lists.ts:183-186` |
| **无查看权限** | `NOT_FOUND` (404) | 同上，与不存在保持一致 | `bookmarks.ts:112-117` |
| **无编辑权限** | `FORBIDDEN` (403) | 明确拒绝，用户已有上下文知道存在 | `lists.ts:436-440` |
| **无管理权限** | `FORBIDDEN` (403) | 同上 | `lists.ts:459-463` |
| **Smart List 不支持编辑** | `BAD_REQUEST` (400) | 业务逻辑约束 | `lists.ts:965-969` |

**关键安全设计**：
> 权限检查失败时，优先返回 `NOT_FOUND` 而非 `FORBIDDEN`，防止攻击者通过状态差异枚举列表 ID。仅在用户已确认资源存在时（如已成功查看），才返回 `FORBIDDEN` 明确拒绝写入。

### 5.2 异常处理链路

```
User Action
    ↓
React Hook (mutation)
    ↓  [catch TRPCError]
    - 前端提示错误
    - 不触发缓存失效（保持一致性）
    ↓
tRPC Middleware
    ↓  [ensureListAtLeastViewer]
    - 权限断言失败 → 抛出 TRPCError
    - 无错误日志 (？需确认)
    ↓
Model Layer
    ↓  [ensureCanX]
    - 权限断言失败 → 抛出 TRPCError
    - 代码位置：lists.ts:434-464
    ↓
Database Layer
    ↓  [Drizzle Query]
    - 查询结果为空 → 上层抛出 NOT_FOUND
    - 约束违反 → 500 (？需确认处理)
```

### 5.3 前端错误边界

**公开页面** (`page.tsx:80-82`)：
```typescript
// 仅捕获 NOT_FOUND，重定向到 not-found.tsx
// 其他错误向上冒泡到根错误边界
catch (e) {
  if (e instanceof TRPCError && e.code === "NOT_FOUND") {
    notFound();
  }
}
```

**Dashboard 组件**：
- 依赖 React Query 的 `error` 状态处理
- `ShareListModal` 无专门错误边界，依赖父组件处理

---

## 六、安全边界总结

### 6.1 纵深防御矩阵

| 防御层级 | 保护机制 | 关键实现 |
|---------|---------|---------|
| **L1: 路由层** | 端点隔离 | `publicProcedure` 无 mutation；`listsProcedure` 强制认证 |
| **L2: 中间件层** | 权限前置检查 | `ensureListAtLeastXxx` 三层中间件；认证后才执行业务逻辑 |
| **L3: 模型层** | 方法级断言 | `ensureCanView/Edit/Manage()` 在每个操作前执行 |
| **L4: 查询层** | 查询条件强制注入 | 所有列表查询加入 `userId` 或 `public=true` 条件 |
| **L5: 序列化层** | 字段白名单裁剪 | `asPublicBookmark()`, `asZBookmarkList()` 显式列出字段 |
| **L6: 类型层** | TypeScript 约束 | 返回类型不包含敏感字段定义 |
| **L7: 资产层** | 签名 URL 保护 | `getPublicSignedAssetUrl()` 限时 + idempotency |

### 6.2 潜在风险点

1. **模拟上下文泄露风险**：
   - `buildImpersonatingAuthedContext` 创建的上下文权限很高
   - 目前仅在 `getPublicListContents` 内部使用，使用后立即销毁
   - 风险：如果不慎对外暴露此上下文，可能导致权限越权

2. **RSS Token 权限等价性**：
   - `rssToken` 与 `public=true` 具有相同访问权限
   - Token 无过期机制（？需确认）
   - 风险：Token 泄露可能导致长期未授权访问

3. **嵌套列表权限继承**：
   - `parentId` 字段存在，但公开状态不继承（？需确认）
   - 风险：公开列表可能包含私有的子列表引用

---

## 七、附录：关键代码位置索引

| 功能 | 文件路径 | 行号 |
|-----|---------|-----|
| 公开列表元数据查询 | `packages/trpc/models/lists.ts` | 191-204 |
| 公开列表书签查询 | `packages/trpc/models/lists.ts` | 206-253 |
| 书签公开字段裁剪 | `packages/trpc/models/bookmarks.ts` | 768-860 |
| 权限断言方法 | `packages/trpc/models/lists.ts` | 397-464 |
| 列表中间件 | `packages/trpc/routers/lists.ts` | 26-58 |
| 公开路由 | `packages/trpc/routers/publicBookmarks.ts` | 全文 |
| 共享 UI 组件 | `apps/web/components/dashboard/lists/PublicListLink.tsx` | 全文 |
| 公开访问页面 | `apps/web/app/public/lists/[listId]/page.tsx` | 全文 |
| 缓存失效调度 | `packages/shared-react/hooks/query-invalidation.ts` | 全文 |
| Lists Hook | `packages/shared-react/hooks/lists.ts` | 全文 |
| 数据库 Schema | `packages/db/schema.ts` | bookmarkLists 定义 |
| Worker 队列定义 | `packages/shared-server/src/queues.ts` | 全文 |
