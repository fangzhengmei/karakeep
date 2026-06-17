# 移动端书签离线同步与冲突合并 —— 代码链路梳理

> 结论先行：Karakeep 移动端**并没有**一套独立的“离线同步队列 + 客户端冲突合并引擎”。
> 它的实现是“分布式”的——缓存层、同步触发、合并/一致性策略分别落在 **TanStack Query（内存缓存）**、
> **tRPC mutation 的 `onSuccess` 失效**、**轮询 / 前台聚焦重取** 以及 **服务端权威写入** 之上。
> 这正是它“读起来不直白”的根因：没有一个 `sync.ts` 文件可以一次性读完，需要把多条链路拼起来理解。

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
- 写入 = tRPC mutation，**服务端权威、最后写入胜出（last-write-wins）**，客户端不做字段级合并。
- 所谓“离线”只是 React Query 在断网时仍展示内存里的 stale 数据；没有落盘的 mutation 待发队列。
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
        staleTime: 60_000,   // 1 分钟内视为“新鲜”，不自动重取
      },
    },
  });
}
```

要点：

- `staleTime: 60_000`：数据在 1 分钟内不会触发后台 refetch（被动刷新由 mutation 失效和聚焦事件驱动）。
- **未覆盖 `retry`**，沿用 React Query 默认 `retry: 3`：网络抖动会在内存里重试 3 次，但 mutation 失败后**不落盘**，丢失即丢失。
- **未配置 `gcTime` 之外的持久化**：App 被杀进程后缓存清空，下次冷启动重新从服务端拉取。
- 浏览器侧用单例 `browserQueryClient` 避免 Suspense 时重建；移动端通过 [Providers](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/providers.tsx#L22-L29) 复用同一 client。

### 2.2 网络层（tRPC httpBatchLink）

```ts
// trpc-provider.tsx#getTRPCClient
httpBatchLink({
  url: `${settings.address}/api/trpc`,
  maxURLLength: TRPC_MAX_URL_LENGTH_EXTERNAL,
  fetch: (url, options) => {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 30_000); // 30s 超时
    // 透传外部 abort signal
    ...
  },
  transformer: superjson,   // 支持 Date 等类型
  headers() { return { Authorization: `Bearer ${settings.apiKey}`, ...customHeaders }; }
})
```

- 批量请求合并（`httpBatchLink`），30s 超时硬中断。
- superjson 传输，保证 `createdAt` 等 `Date` 字段在端到端往返中保持类型。

### 2.3 本地落盘的“伪缓存”

真正写到本地存储的只有两类非书签数据：

| 内容 | 存储 | 代码 |
| --- | --- | --- |
| 搜索历史 | AsyncStorage | [search-history.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/search-history.ts) |
| 应用/连接设置 | SecureStore | [settings.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/settings.ts) |

搜索查询本身则用 React Query 的 `keepPreviousData` 做占位（[useBookmarkSearchState.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/useBookmarkSearchState.ts#L32-L44)），`gcTime: 0` 即用即弃，避免缓存膨胀。

> ⚠️ 常见误解：这里 **没有** `@tanstack/query-async-storage-persister`。因此“离线打开 App 还能看到上次书签”依赖的是进程未被回收时的内存缓存，而非磁盘缓存。

---

## 3. 同步触发链路（“同步队列”的真实形态）

移动端没有显式的 mutation outbox/queue，而是由 **四种触发器** 共同组成“同步”行为：

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

### 3.2 失效去抖（最接近“队列”的机制）

[query-invalidation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/query-invalidation.ts) 实现了一个**去抖 + 最大等待**的失效合并器，这是整个链路里唯一带“排队”语义的组件：

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

“是否还在 loading”由 [isBookmarkStillLoading](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared/utils/bookmarkUtils.ts#L48-L54) 判断（爬取 / 打标签 / 摘要任一 pending）。这是**服务端推不动、客户端拉**的同步模式。

### 3.4 前台聚焦 + 下拉刷新

- **前台聚焦重取**：[dashboard/_layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/app/dashboard/_layout.tsx#L11-L14) 监听 `AppState`，调用 `focusManager.setFocused(status === "active")`，React Query 据此对 stale query 自动 refetch——这是回到 App 时数据“自动同步”的原因。
- **下拉刷新**：[UpdatingBookmarkList.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/components/bookmarks/UpdatingBookmarkList.tsx#L55-L57) 的 `onRefresh` 直接 `invalidateQueries` 列表与详情，强制重取。

### 3.5 小结：同步链路全景

```
用户操作
  │
  ▼
useUpdateBookmark / useDeleteBookmark / ...   (shared-react/hooks)
  │  mutate → tRPC httpBatchLink → 服务端
  │
  ▼ onSuccess
scheduleInvalidateQueries (250ms 去抖 / 3s 上限)   ← “队列”形态
  │
  ▼ 触发
QueryClient.invalidateQueries → 后台 refetch
  │
  ▼ 合并到内存缓存
UI 自动重渲染（stale-while-revalidate）

并行的“拉”同步：
  • AppState active → focusManager → stale query refetch
  • 下拉刷新 → invalidateQueries
  • 新书签 → useAutoRefreshingBookmarkQuery 轮询（1s/10s/60s 递减）
```

---

## 4. 合并 / 一致性策略（Merge Strategy）

代码里“merge”一词出现在三处不同语义，需分别理解，这是最容易混淆的点：

### 4.1 书签本身：服务端权威 + last-write-wins（无客户端合并）

书签字段（标题、笔记、归档、收藏等）的更新走 `bookmarks.updateBookmark`，**整字段覆盖**写入服务端数据库。客户端不做任何字段级三方合并：

- 谁的 mutation 最后到达服务端，谁的字段就生效。
- 并发冲突（如两端同时改标题）**不会**在客户端被检测或合并，而是直接覆盖。
- 写入成功后客户端 invalidate → 重取服务端最新版本，从而“对齐”。

> 即“冲突合并”在书签维度上**不存在**；一致性靠“服务端单点权威 + 写后重取”保证最终一致。

### 4.2 列表合并 `lists.merge`（这是“合并两个清单”，不是同步冲突）

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

- 把源清单的所有书签“搬”进目标清单，`onConflictDoNothing` 做幂等去重——这是全代码库里**唯一真正意义上的“合并”**，但语义是“清单间成员合并”。
- 只能合并进 `manual` 清单，`SmartList.mergeInto` 直接抛 `BAD_REQUEST`（[lists.ts:979-987](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/models/lists.ts#L979-L987)）。
- 在事务内执行，可选删除源清单，保证原子性。
- 成功后客户端 invalidate 清单树、目标清单书签列表、stats。

### 4.3 阅读器设置：分层优先级 + pending 防抖（最接近“冲突合并”的模式）

[reader-settings.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/reader-settings.tsx) 是移动端唯一带有“乐观 + 服务端确认 + 回滚清理”语义的代码，可作为理解“合并策略”的最佳范例：

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

## 5. 为什么“读起来不直白”——定位索引

| 关注点 | 应该读的文件 | 一句话 |
| --- | --- | --- |
| 缓存初始化 / staleTime | [trpc-provider.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/providers/trpc-provider.tsx) | 内存缓存，60s 新鲜期，无持久化 |
| mutation → 失效 | [bookmarks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/bookmarks.ts) | 写后 invalidate，无乐观/无回滚 |
| 失效去抖“队列” | [query-invalidation.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/query-invalidation.ts) | 250ms/3s 合并失效，唯一的“排队” |
| 轮询同步 | [bookmarks.ts#useAutoRefreshingBookmarkQuery](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/bookmarks.ts#L12-L27) + [bookmarkUtils.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared/utils/bookmarkUtils.ts#L56-L83) | 1s/10s/60s 递减拉取 |
| 前台重取 | [dashboard/_layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/app/dashboard/_layout.tsx#L11-L14) | AppState → focusManager |
| 列表合并 | [lists.ts (router)](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/routers/lists.ts#L103-L116) + [lists.ts (model)](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/trpc/models/lists.ts#L1109-L1142) | `onConflictDoNothing` 成员合并 |
| 乐观 + 回滚（仅阅读器） | [reader-settings.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/packages/shared-react/hooks/reader-settings.tsx) | 分层优先级 + pending 防抖 |
| 资源/连接设置落盘 | [settings.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/21-karakeep/apps/mobile/lib/settings.ts) | SecureStore，非书签缓存 |

---

## 6. 结论与设计提示

1. **没有离线 outbox**：断网期间的写操作会随 `retry: 3` 失败后丢失。若需真正的离线同步，需要新增 mutation 持久化层（如 `@tanstack/query-async-storage-persister` + 自定义 mutation 队列 + 重放）。
2. **书签无字段级冲突合并**：靠服务端 last-write-wins + 写后重取维持最终一致；多端并发编辑同一字段会互相覆盖。
3. **“合并”唯一真实存在**于清单成员迁移（`mergeInto` 用 `onConflictDoNothing` 去重）和阅读器设置的分层优先级。
4. 阅读器设置的 `pendingServerSave` 模式是可复用的“乐观 + 确认 + 回滚”范式，若未来要给书签加乐观更新，可参考其结构（`onMutate` 写缓存 → `onError` 回滚 → `onSettled` 重取）。
