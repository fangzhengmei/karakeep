# 移动端书签离线同步与冲突合并 —— 代码链路梳理（核准版）

> 结论先行：Karakeep 移动端**并没有**一套独立的"离线同步队列 + 客户端冲突合并引擎"。
> 它的实现是"分布式"的——缓存层、同步触发、合并/一致性策略分别落在 **TanStack Query（内存缓存）**、
> **tRPC mutation 的 `onSuccess` 失效**、**轮询 / 前台聚焦重取** 以及 **服务端权威写入** 之上。
> 这正是它"读起来不直白"的根因：没有一个 `sync.ts` 文件可以一次性读完，需要把多条链路拼起来理解。

---

## 1. 整体架构与分层

```
移动端 (apps/mobile)
  └── lib/providers.tsx ── 注入 QueryClient + tRPC Client
        └── shared-react/providers/trpc-provider.tsx  ← 缓存层（QueryClient）
              └── shared-react/hooks/*.ts             ← mutation/query 封装（同步触发点）
                    └── trpc (httpBatchLink)          ← 网络层 → 服务端权威数据源
```

关键事实：

- 缓存 = **TanStack Query 的内存缓存**，没有持久化（无 `persistQueryClient` / AsyncStorage persister）。
- 写入 = tRPC mutation，**服务端权威**；客户端不做字段级合并。
- 所谓"离线"只是 React Query 在断网时仍展示内存里的 stale 数据；**没有落盘的 mutation 待发队列**。
- 唯一真正落盘的本地数据是 **搜索历史**（AsyncStorage）和 **应用设置**（SecureStore），不是书签缓存。

---

## 2. 缓存层（Cache Layer）

### 2.1 QueryClient 的构建与配置

入口在 [trpc-provider.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/providers/trpc-provider.tsx)，由移动端 [providers.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/providers.tsx) 注入。

```ts
// trpc-provider.tsx
function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60_000,   // 1 分钟内视为"新鲜"，不自动重取
      },
    },
  });
}
```

要点：

- `staleTime: 60_000`：查询结果在 1 分钟内不会触发后台 refetch（被动刷新由 mutation 失效和聚焦事件驱动）。
- **未配置 `gcTime` 之外的持久化**：App 被杀进程后缓存清空，下次冷启动重新从服务端拉取。
- 浏览器侧用单例 `browserQueryClient` 避免 Suspense 时重建；移动端通过 [Providers](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/providers.tsx#L22-L29) 复用同一 client。

### 2.2 写入操作断网后的重试行为（核准点 ✅）

> **校正：此前"mutation 默认 `retry: 3`"的说法是错误的。**

TanStack Query v5 的默认值 **对 query 和 mutation 是不一样的**：

| 类型 | 默认 `retry` |
| --- | --- |
| `useQuery` | **3** 次（指数退避） |
| `useMutation` | **0** 次（不重试） |

本项目 [trpc-provider.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/providers/trpc-provider.tsx) **没有**在 `defaultOptions` 中覆盖 `mutations.retry`；shared-react/hooks 下所有 `useMutation` 调用也都**没有**传 `retry` 参数。因此：

- **书签的创建 / 更新 / 删除 / 打标签、列表增删、高亮增删改等所有写入操作，断网时立即失败，不会有任何内存重试。**
- 仅有的网络层容错是 [trpc-provider.tsx#getTRPCClient](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/providers/trpc-provider.tsx#L48-L81) 的 **30 秒超时硬中断**（`AbortController`），这不是重试，只是防止请求永久挂起。
- 仓库里显式覆盖 `retry` 的地方只有：
  - [HighlightCard.tsx:72](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/components/highlights/HighlightCard.tsx#L72) —— `retry: false`，但它是 `useQuery`（查询单个书签），不是 mutation。
  - Web 端的 `SidebarVersion.tsx`（`retry: 1`）和 `ValidAccountCheck.tsx`（自定义 predicate），同样是 query。

**后果**：断网期间用户执行的任何写入（收藏、归档、改标题、打标签等）在 30s 超时后直接失败，无重试、无 outbox 落盘、无重放——用户操作丢失，仅由 UI Toast 提示"Something went wrong"。

### 2.3 网络层（tRPC httpBatchLink）

```ts
// trpc-provider.tsx#getTRPCClient
httpBatchLink({
  url: `${settings.address}/api/trpc`,
  maxURLLength: TRPC_MAX_URL_LENGTH_EXTERNAL,
  fetch: (url, options) => {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 30_000); // 30s 超时
    // 透传 tRPC / React Query 的外部 abort signal
    ...
  },
  transformer: superjson,   // 支持 Date 等类型
  headers() { return { Authorization: `Bearer ${settings.apiKey}`, ...customHeaders }; }
})
```

- 批量请求合并（`httpBatchLink`），30s 超时硬中断。
- superjson 传输，保证 `createdAt` 等 `Date` 字段在端到端往返中保持类型。

### 2.4 本地落盘的"伪缓存"

真正写到本地存储的只有两类非书签数据：

| 内容 | 存储 | 代码 |
| --- | --- | --- |
| 搜索历史 | AsyncStorage | [search-history.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/search-history.ts) |
| 应用/连接设置 | SecureStore | [settings.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/settings.ts) |

搜索查询本身则用 React Query 的 `keepPreviousData` 做占位（[useBookmarkSearchState.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/useBookmarkSearchState.ts#L32-L44)），`gcTime: 0` 即用即弃，避免缓存膨胀。

> ⚠️ 常见误解：这里 **没有** `@tanstack/query-async-storage-persister`。因此"离线打开 App 还能看到上次书签"依赖的是进程未被回收时的内存缓存，而非磁盘缓存。

---

## 3. 同步触发链路（"同步队列"的真实形态）

移动端没有显式的 mutation outbox/queue，而是由 **四种触发器** 共同组成"同步"行为：

### 3.1 写入后失效（主动同步主线）

所有写操作封装在 [bookmarks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/bookmarks.ts)、[lists.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/lists.ts)、[highlights.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/highlights.ts)。模式统一：**mutation 成功 → 失效相关 query → React Query 后台 refetch 服务端权威数据**。

以更新书签为例（[useUpdateBookmark](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/bookmarks.ts#L103-L130)）：

```ts
return useMutation(api.bookmarks.updateBookmark.mutationOptions({
  ...opts,
  onSuccess: (res, req, meta, context) => {
    scheduleInvalidateQueries(queryClient, api.bookmarks.getBookmarks.pathFilter());
    scheduleInvalidateQueries(queryClient, api.bookmarks.searchBookmarks.pathFilter());
    queryClient.invalidateQueries(api.bookmarks.getBookmark.queryFilter({ bookmarkId: req.bookmarkId }));
    scheduleInvalidateQueries(queryClient, api.lists.stats.pathFilter());
    return opts?.onSuccess?.(res, req, meta, context);
  },
}));
```

特征：

- **无 `onMutate` 乐观更新**（除了阅读器设置，见 §4.3）。UI 等 mutation 返回后再刷新。
- **无 `onError` 回滚**。失败时由调用方自行 toast 提示（如 [ActionBar.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/components/bookmarks/card/ActionBar.tsx#L40-L46) 的 `onError`）。
- `removeQueries` 用于删除型操作，避免残留旧 key（见 [useDeleteBookmark](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/bookmarks.ts#L93-L95)）。

### 3.2 失效去抖（最接近"队列"的机制）

[query-invalidation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/query-invalidation.ts) 实现了一个**去抖 + 最大等待**的失效合并器，这是整个链路里唯一带"排队"语义的组件：

```ts
const DEFAULT_INVALIDATION_DEBOUNCE_MS = 250;   // 250ms 去抖
const DEFAULT_INVALIDATION_MAX_WAIT_MS = 3_000; // 最长压 3s 必发

export function scheduleInvalidateQueries(queryClient, filters, debounceMs = 250, maxWaitMs = 3_000) {
  // 用 hashKey 把相同 filter 的失效请求合并到同一个 timer
  // 若距首次请求已接近 maxWait，则立即触发，避免无限延迟
}
```

它解决的问题是：一次批量操作（如给书签加多个 tag）会触发多次 `invalidateQueries`，若每次都立即 refetch 会造成请求风暴。这里用 `WeakMap<QueryClient, Map<key, PendingInvalidation>>` 按 filter 结构做哈希合并，把多次失效压成一次。**注意：带 `predicate` 的 filter 不可哈希，直接同步执行。**

### 3.3 轮询同步（服务端异步任务的进度拉取）

书签创建后，服务端会异步爬取/打标签/摘要。客户端用轮询拉取进度，见 [useAutoRefreshingBookmarkQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/bookmarks.ts#L12-L27) 配合 [getBookmarkRefreshInterval](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared/utils/bookmarkUtils.ts#L56-L83)：

| 创建后时长 | 轮询间隔 |
| --- | --- |
| 0–30s | 1s |
| 30s–10min | 10s |
| 10min–6h | 60s |
| >6h 或已不再 loading | 停止 |

"是否还在 loading"由 [isBookmarkStillLoading](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared/utils/bookmarkUtils.ts#L48-L54) 判断（爬取 / 打标签 / 摘要任一 pending）。这是**服务端推不动、客户端拉**的同步模式。

### 3.4 前台聚焦 + 下拉刷新

- **前台聚焦重取**：[dashboard/_layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/app/dashboard/_layout.tsx#L11-L14) 监听 `AppState`，调用 `focusManager.setFocused(status === "active")`，React Query 据此对 stale query 自动 refetch——这是回到 App 时数据"自动同步"的原因。
- **下拉刷新**：[UpdatingBookmarkList.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/components/bookmarks/UpdatingBookmarkList.tsx#L55-L57) 的 `onRefresh` 直接 `invalidateQueries` 列表与详情，强制重取。

### 3.5 小结：同步链路全景

```
用户操作
  │
  ▼
useUpdateBookmark / useDeleteBookmark / ...   (shared-react/hooks)
  │  mutate → tRPC httpBatchLink → 服务端
  │  (断网 30s 超时立即失败，mutation 默认 retry=0，无重试)
  │
  ▼ onSuccess
scheduleInvalidateQueries (250ms 去抖 / 3s 上限)   ← "队列"形态
  │
  ▼ 触发
QueryClient.invalidateQueries → 后台 refetch
  │
  ▼ 合并到内存缓存
UI 自动重渲染（stale-while-revalidate）

并行的"拉"同步：
  • AppState active → focusManager → stale query refetch
  • 下拉刷新 → invalidateQueries
  • 新书签 → useAutoRefreshingBookmarkQuery 轮询（1s/10s/60s 递减）
```

---

## 4. 合并 / 一致性策略（Merge Strategy）

代码里"merge"一词出现在三处不同语义，需分别理解，这是最容易混淆的点。本节重点核准**多端编辑边界**。

### 4.1 书签本身：服务端权威 + 字段级部分更新（核准点 ✅）

> **校正：此前"整字段覆盖写入"的说法不准确。**

服务端 [updateBookmark mutation](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/routers/bookmarks.ts#L466-L652) 采用的是 **按字段条件 UPDATE（partial patch）**，不是整条记录覆盖。其请求 schema [zUpdateBookmarksRequestSchema](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared/types/bookmarks.ts#L227-L251) 中除 `bookmarkId` 外所有字段都是 `optional()` / `nullish()`：

```ts
// bookmarks.ts#L554-L594 —— 服务端按字段条件更新
const commonUpdateData: Partial<{...}> = { modifiedAt: new Date() };
if (input.title !== undefined)      commonUpdateData.title = input.title;
if (input.archived !== undefined)   commonUpdateData.archived = input.archived;
if (input.favourited !== undefined) commonUpdateData.favourited = input.favourited;
if (input.note !== undefined)       commonUpdateData.note = input.note;
if (input.summary !== undefined)    commonUpdateData.summary = input.summary;
if (input.createdAt !== undefined)  commonUpdateData.createdAt = input.createdAt;

if (Object.keys(commonUpdateData).length > 1 || somethingChanged) {
  await tx.update(bookmarks).set(commonUpdateData)
    .where(and(eq(bookmarks.userId, ctx.user.id), eq(bookmarks.id, input.bookmarkId)));
}
```

link / text / asset 各自的专属字段也有同样的 `if (input.xxx)` 保护。

**多端并发编辑覆盖边界：**

| 场景 | 结果 | 依据 |
| --- | --- | --- |
| A 改 `archived`，B 改 `favourited`（不同字段） | **两个修改都保留**，互不覆盖 | 各自的 `if` 分支独立执行，SQL 只 UPDATE 被包含的列 |
| A 改 `title: "X"`，B 改 `title: "Y"`（同字段） | **last-write-wins**，后提交事务的一方覆盖先提交的 | 同列被两次 UPDATE，无版本号 / CAS 保护 |
| A 改 `note`，B 同时 detach tag `T` | **两个修改都保留** | 打标签走 `updateTags`（独立 mutation、独立事务、`tagsOnBookmarks` 表），与 `bookmarks` 表 UPDATE 无冲突 |
| A 改 `title`，B 同时改同一个 link 的 `description` | **两个修改都保留** | `title` 走 `bookmarks` 表，`description` 走 `bookmarkLinks` 表，两条独立的 UPDATE |

**注意事项：**

- 没有乐观锁 / ETag / 版本向量 / `modifiedAt` 比较；并发写同字段时服务端不检测冲突。
- `modifiedAt` 在任何字段更新时都会被刷新为 `new Date()`，它不是 CAS 依据，仅作展示。
- 写入成功后客户端靠 invalidate → 重取服务端最新版本"对齐"。
- 客户端层面无字段级三方合并逻辑。

`updateTags` 的边界更简单：attach 用 `onConflictDoNothing` 幂等，detach 直接删行。两端同时 attach 不同 tag 无冲突；两端同时对同一 tag 做 attach + detach 则最后执行的操作生效。

### 4.2 列表合并 `lists.merge`（这是"合并两个清单"，不是同步冲突）

[useMergeLists](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/lists.ts#L59-L82) → 服务端 [lists.ts#merge](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/routers/lists.ts#L103-L116) → [ManualList.mergeInto](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/models/lists.ts#L1109-L1142)：

```ts
const bookmarkIds = await this.getBookmarkIds();
await this.ctx.db.transaction(async (tx) => {
  await tx.insert(bookmarksInLists).values(
    bookmarkIds.map((id) => ({ bookmarkId: id, listId: targetList.id })),
  ).onConflictDoNothing();            // 去重：目标已存在的书签跳过
  if (deleteSourceAfterMerge) {
    await tx.delete(bookmarkLists).where(eq(bookmarkLists.id, this.list.id));
    await this.cleanupRulesAfterListDeletion(tx);
  }
});
```

要点：

- 把源清单的所有书签"搬"进目标清单，`onConflictDoNothing` 做幂等去重——这是全代码库里**唯一真正意义上的"合并"**，但语义是"清单间成员合并"。
- 只能合并进 `manual` 清单，`SmartList.mergeInto` 直接抛 `BAD_REQUEST`（[lists.ts:979-987](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/models/lists.ts#L979-L987)）。
- 在事务内执行，可选删除源清单，保证原子性。
- 成功后客户端 invalidate 清单树、目标清单书签列表、stats。

### 4.3 阅读器设置：分层优先级 + pending 防抖（最接近"冲突合并"的模式）

[reader-settings.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/reader-settings.tsx) 是移动端唯一带有"乐观 + 服务端确认 + 回滚清理"语义的代码，可作为理解"合并策略"的最佳范例：

**有效值优先级（高 → 低）：**

```
sessionOverrides → localOverrides → pendingServerSave → serverSettings → READER_DEFAULTS
```

见 [useReaderSettings#settings](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/reader-settings.tsx#L110-L132)。

**同步与防抖机制：**

- `updateLocal`：立即写本地（每设备）并展示，不等服务端。
- `saveAsDefault`：设 `pendingServerSave`（乐观值，防止服务端回包前的闪烁）→ 发 `updateSettings` mutation。
- **服务端确认**：[useEffect](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/reader-settings.tsx#L66-L79) 监听 `serverSettings`，当服务端值与 `pendingServerSave` 匹配（lineHeight 容忍 1e-6 浮点误差）时清空 pending。
- **失败回滚**：`saveServerSettings` 的 `onError` 把 `pendingServerSave` 置空，避免展示未真正持久化的值（[L99-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/reader-settings.tsx#L99-L102)）。
- `onSettled` 一律 `refetchQueries(users.settings)` 拉取服务端最新值对齐。

这是一种 **本地优先 + 服务端最终权威 + 乐观挂起 + 失败回退** 的轻量一致性策略，但**仅作用于阅读器设置**，书签本体未采用。

---

## 5. 仓库内链接可移植性（核准点 ✅）

本文件使用 `file:///d:/absolute/path` 形式的绝对 URI 引用代码，可移植性要点：

- **路径分隔符统一用正斜杠 `/`**，即使在 Windows 环境下。这符合 Markdown / file URI 的通用规范，在任何平台的 IDE 和 Markdown 渲染器中都能正确解析；Windows 原生反斜杠 `\` 在 URI 中是非法字符。
- **绝对路径绑定当前工作目录**（`d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/`）。将仓库克隆到另一台机器或另一路径后，这些链接会失效。若需仓库级可移植链接，可改写为相对路径（如 `packages/shared-react/providers/trpc-provider.tsx`），但会牺牲 IDE 中的跳转能力。
- **行号锚点使用 `#L<start>-L<end>` 格式**（两端都带 `L` 前缀），这是 LSP / IDE 通用的锚点规范。

本文件所有代码引用均已按上述三条规则书写。

---

## 6. 为什么"读起来不直白"——定位索引

| 关注点 | 应该读的文件 | 一句话 |
| --- | --- | --- |
| 缓存初始化 / staleTime | [trpc-provider.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/providers/trpc-provider.tsx) | 内存缓存，60s 新鲜期，无持久化 |
| mutation 重试行为（默认 0 次） | [trpc-provider.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/providers/trpc-provider.tsx) + TanStack Query v5 默认值 | 写入断网立即失败，不重试 |
| mutation → 失效 | [bookmarks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/bookmarks.ts) | 写后 invalidate，无乐观/无回滚 |
| 失效去抖"队列" | [query-invalidation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/query-invalidation.ts) | 250ms/3s 合并失效，唯一的"排队" |
| 轮询同步 | [bookmarks.ts#useAutoRefreshingBookmarkQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/bookmarks.ts#L12-L27) + [bookmarkUtils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared/utils/bookmarkUtils.ts#L56-L83) | 1s/10s/60s 递减拉取 |
| 前台重取 | [dashboard/_layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/app/dashboard/_layout.tsx#L11-L14) | AppState → focusManager |
| 服务端部分更新逻辑 | [bookmarks.ts (router)#updateBookmark](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/routers/bookmarks.ts#L466-L652) | 按字段条件 UPDATE，不同字段互不覆盖 |
| 请求 schema（字段 optional） | [zUpdateBookmarksRequestSchema](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared/types/bookmarks.ts#L227-L251) | 除 bookmarkId 外全字段可选 |
| 标签增删边界 | [bookmarks.ts (router)#updateTags](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/routers/bookmarks.ts#L975-L1174) | `onConflictDoNothing` 幂等 attach |
| 列表合并 | [lists.ts (router)](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/routers/lists.ts#L103-L116) + [lists.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/models/lists.ts#L1109-L1142) | `onConflictDoNothing` 成员合并 |
| 乐观 + 回滚（仅阅读器） | [reader-settings.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/reader-settings.tsx) | 分层优先级 + pending 防抖 |
| 资源/连接设置落盘 | [settings.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/settings.ts) | SecureStore，非书签缓存 |

---

## 7. 结论与设计提示

1. **没有离线 outbox，mutation 断网立即失败（retry=0）**：断网期间写操作 30s 超时后直接丢失，无重试、无持久化、无重放。若需真正的离线同步，需要新增 mutation 持久化层（如 outbox 表 + 自定义 mutation 队列 + 重放调度），并配套乐观更新 + onError 回滚。
2. **书签按字段部分 UPDATE，不是整行覆盖**：多端编辑**不同字段**时修改都保留（不冲突）；编辑**同一字段**时 last-write-wins。无 CAS / 版本号保护。
3. **"合并"唯一真实存在**于清单成员迁移（`mergeInto` 用 `onConflictDoNothing` 去重）和阅读器设置的分层优先级。
4. 阅读器设置的 `pendingServerSave` 模式是可复用的"乐观 + 确认 + 回滚"范式，若未来要给书签加乐观更新，可参考其结构（`onMutate` 写缓存 → `onError` 回滚 → `onSettled` 重取）。
5. **链接可移植性**：本文档所有 `file://` URI 均使用正斜杠和 `#Lx-Ly` 锚点，符合通用规范；但路径是当前机器绝对路径，跨机器迁移时需改写为相对路径。
