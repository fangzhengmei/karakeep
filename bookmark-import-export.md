# 书签导入导出转换规则分析（修正版）

## 概述

Karakeep 的书签导入导出系统位于 `packages/shared/import-export/`，采用 **"解析器 + 处理器 + 后台Worker"** 三层架构，支持 **11 种外部格式的导入** 和 **2 种格式的导出**。

---

## 核心组件边界与数据流

### 架构分层

| 层级 | 模块 | 职责 | 有无副作用 |
|------|------|------|-----------|
| 1. 格式解析层 | `parsers.ts` | 纯函数式格式转换，外部格式 → 内部统一结构 | ❌ 纯函数，无副作用 |
| 2. 导入编排层 | `importer.ts` | 目录结构重建、批量暂存编排，通过 `deps` 接口注入外部依赖 | ⚠️ 通过依赖间接产生副作用 |
| 3. 会话管理层 | `importSessions.service.ts` | 导入会话状态机管理、暂存数据封装 | ✅ 数据库操作 |
| 4. 后台处理层 | `importWorker.ts` | 实际书签创建、并发控制、下游处理等待 | ✅ 完整业务逻辑 |
| 5. 前端接入层 | `useBookmarkImport.ts` | 配额检查、进度展示、用户交互 | ✅ UI 层 |

### 完整数据流

```
前端用户上传文件
    ↓
[useBookmarkImport.ts]
    ├─ 预解析文件统计数量
    ├─ 检查配额（quotaUsage）
    └─ 调用 importBookmarksFromFile
        ↓
[importer.ts] importBookmarksFromFile
    ├─ 解析文件（parseImportFile）
    ├─ 创建导入根列表
    ├─ 重建目录结构（paths 或 listExternalIds）
    ├─ 转换为 StagedBookmark
    ├─ 批量暂存（50条/批）
    └─ finalize → 会话状态从 staging → pending
        ↓
[importWorker.ts] ImportWorker（轮询，5秒/次）
    ├─ resetStaleProcessingItems（每60次轮询≈5分钟）
    ├─ checkAndCompleteProcessingItems（每次轮询先执行）
    ├─ processBatch
    │   ├─ 公平调度取候选
    │   ├─ 原子声明为 processing
    │   ├─ processOneBookmark（并行处理）
    │   │   ├─ createBookmark
    │   │   ├─ updateTags
    │   │   ├─ attachBookmarkToLists
    │   │   └─ 状态更新
    │   └─ checkAndCompleteEmptySessions
    └─ checkAndCompleteIdleSessions
```

---

## 数据库字段与状态机

### 数据库 Schema 核心字段

**`importSessions` 表**（schema.ts:851-879）

| 字段 | 类型 | 枚举值 | 默认值 |
|------|------|--------|--------|
| `status` | text | `["staging", "pending", "running", "paused", "completed", "failed"]` | `staging` |
| `lastProcessedAt` | timestamp | - | - |
| `rootListId` | text | - | - |

**`importStagingBookmarks` 表**（schema.ts:903-950）

| 字段 | 类型 | 枚举值 | 默认值 |
|------|------|--------|--------|
| `status` | text | `["pending", "processing", "completed", "failed"]` | `pending` |
| `result` | text | `["accepted", "rejected", "skipped_duplicate"]` | `NULL` |
| `resultReason` | text | - | `NULL` |
| `resultBookmarkId` | text | - | `NULL` |
| `processingStartedAt` | timestamp | - | `NULL` |
| `completedAt` | timestamp | - | `NULL` |

> **重要区分**：
> - `status`: 处理进度状态（pending/processing/completed/failed）
> - `result`: 最终结果类型（仅 completed/failed 时有意义）
> - 函数内部返回值（如 `"unsupported"`, `"reset"`）是代码级标记，**不存入数据库**

---

### 暂存书签完整状态机

```
                          +-----------------+
                          |     pending     |
                          +-----------------+
                                   |
              [processBatch 原子声明，importWorker.ts:181-190]
                                   ↓
                          +-----------------+
                          |   processing    |
                          +-----------------+
                                   |
         +-------------------------+-------------------------+
         |                         |                         |
[createBookmark 抛出异常]   [已存在重复]          [成功创建新书签]
         |                         |                         |
         ↓                         ↓                         ↓
+-----------------+      +-----------------+      +-----------------+
|     failed      |      |   completed     |      |   processing    |←──┐
| result: rejected|      |result: skipped_ |      | result: accepted|   |
| completedAt 设置|      |   duplicate     |      | resultBookmarkId|   |
+-----------------+      | completedAt 设置|      |  设置           |   |
         ^               +-----------------+      +-----------------+   |
         |                         |                         |         |
         |                         |                         ↓         |
[asset 类型不支持]           [终态，不可逆]    [checkAndCompleteProcessingItems]
         |                                                 |         |
         ↓                                                 ↓         |
+-----------------+                              +-----------------+  |
|     failed      |                              |crawl/tag 都完成?|  |
| result: rejected|                              +-----------------+  |
| reason: "Asset   |                                         |         |
|  not supported" |                          +--------------+--------------+
+-----------------+                          |              |              |
                                              ↓              ↓              ↓
                                     +-----------------+ +-----------+ +-----------+
                                     |   completed     | |  failed   | |  继续等   |
                                     | result: accepted| |result: rej| | processing|
                                     | completedAt 设置| |reason: Crawl| +-----------+
                                     +-----------------+ | /Tag failed|       ↑
                                                         +-----------+        |
                                                              |               |
                                                              └───────────────┘
                                                           [下次轮询继续检查]
```

---

### 状态转换的代码路径

#### 1. `pending` → `processing`（原子声明）

**位置**: `importWorker.ts:181-190`

```typescript
// 原子 UPDATE，只有仍为 pending 的行才会被声明
const batch = await db
  .update(importStagingBookmarks)
  .set({ status: "processing", processingStartedAt: new Date() })
  .where(
    and(
      eq(importStagingBookmarks.status, "pending"),
      inArray(importStagingBookmarks.id, candidateIds),
    ),
  )
  .returning();
```

#### 2. `processing` → 终态（`processOneBookmark` 内）

**路径 A：创建失败**（importWorker.ts:469-485）
```typescript
status: "failed"
result: "rejected"
resultReason: getSafeErrorMessage(error)
completedAt: new Date()
```

**路径 B：asset 类型不支持**（importWorker.ts:412-422）
```typescript
status: "failed"
result: "rejected"  // ⚠️ 不是 "unsupported"，数据库没有这个枚举
resultReason: "Asset bookmarks not yet supported"
completedAt: new Date()
// 函数返回 "unsupported" 只是代码内部标记，不入库
```

**路径 C：重复 URL**（importWorker.ts:437-452）
```typescript
status: "completed"
result: "skipped_duplicate"
resultReason: "URL already exists"
resultBookmarkId: result.id  // 关联到已存在的书签
completedAt: new Date()
```

**路径 D：成功创建，等待下游**（importWorker.ts:457-463）
```typescript
// ⚠️ 注意：status 仍为 "processing"，不设置 completedAt
status: "processing"  // 保持不变！
result: "accepted"
resultBookmarkId: result.id
// completedAt 不设置
```

#### 3. `processing` → 终态（`checkAndCompleteProcessingItems` 内）

**位置**: `importWorker.ts:546-650`

**调用时机**: **每次轮询最先执行**（importWorker.ts:126），在 `processBatch` 之前。

```typescript
// 查找条件：
eq(importStagingBookmarks.status, "processing"),
isNotNull(importStagingBookmarks.resultBookmarkId),  // 已创建书签
or(isNull(crawlStatus), eq(crawlStatus, "success"), eq(crawlStatus, "failure")),
or(isNull(taggingStatus), eq(taggingStatus, "success"), eq(taggingStatus, "failure")),

// 都成功 → status: "completed", completedAt: now
// 任一失败 → status: "failed", result: "rejected", reason: "Crawl failed" / "Tagging failed"
```

---

## 失败容忍与重试机制

### 1. 导入前过滤（第一道防线）

**位置**: `importSessions.service.ts:95-100`

暂存前就过滤无效书签，根本不进入处理队列：
```typescript
const validBookmarks = bookmarks.filter((bookmark) => {
  if (bookmark.type === "link" && !bookmark.url) return false;
  if (bookmark.type === "text" && !bookmark.content) return false;
  return true;
});
```

### 2. 处理过程中的错误隔离

- **`Promise.allSettled`**（importWorker.ts:213）：单条失败不影响批次
- **列表关联容错**（importWorker.ts:341-347）：单个列表关联失败只打 warn，不影响书签状态
- **错误信息安全过滤**（importWorker.ts:82-100）：避免泄露堆栈、数据库错误等内部详情给用户

### 3. 挂起项重试机制

**位置**: `importWorker.ts:682-718`

**触发条件**（必须同时满足）：
```
status === "processing"
AND processingStartedAt < NOW - 1小时
AND resultBookmarkId IS NULL  // ⚠️ 关键：已创建书签的不重试！
```

**重置动作**:
```typescript
status: "pending"
processingStartedAt: null  // 清空
```

**触发频率**: 每 60 次轮询 = 60 × 5秒 = **5分钟**（importWorker.ts:120）

> **重要限制**：
> - 已成功创建书签（有 `resultBookmarkId`）的项目**永远不会被重试**
> - 哪怕后续 crawl/tagging 失败了，也只会标记为 `failed`，不会重试
> - 只有"书签创建前"挂起的项目才会被重置重试

### 4. 循环引用与父列表缺失兜底

**位置**: `importer.ts:133-147`

拓扑排序中如果一轮没有创建任何列表（说明有循环引用或父引用不存在），强制全部挂到根目录：
```typescript
if (!createdAny) {
  for (const [externalId, list] of unresolvedLists) {
    const createdList = await deps.createList({
      // ...
      parentId: rootList.id,  // 强制根目录
    });
  }
  unresolvedLists.clear();
}
```

### 5. 会话暂停后的状态回滚

**位置**: `importWorker.ts:358-364`

处理中发现会话已暂停，把当前条目重置回 `pending`：
```typescript
if (!session || session.status === "paused") {
  await db
    .update(importStagingBookmarks)
    .set({ status: "pending" })  // 重置，不保留 processingStartedAt
    .where(eq(importStagingBookmarks.id, staged.id));
  return "reset";  // 代码内部标记，不入库
}
```

### 6. 并发控制与原子性

- **原子声明**：UPDATE + WHERE 确保多 Worker 不重复处理同一条
- **背压控制**：`maxInFlight = 50`，通过 `processingStartedAt` 判断是否为有效在处理项
- **公平调度**：按 `importSessions.lastProcessedAt` 排序，避免大导入阻塞其他用户

---

## 结果类型汇总

### 数据库存储的结果类型

| `result` | `status` | 场景 |
|----------|----------|------|
| `NULL` | `pending` | 等待处理 |
| `NULL` | `processing` | 处理中（书签创建前） |
| `"accepted"` | `processing` | 书签已创建，等待 crawl/tagging |
| `"accepted"` | `completed` | 全部成功 |
| `"skipped_duplicate"` | `completed` | URL 已存在，跳过但仍关联标签/列表 |
| `"rejected"` | `failed` | 验证失败、创建异常、下游失败、asset 不支持 |

### 函数内部返回值（不入库）

| 返回值 | 场景 |
|--------|------|
| `"reset"` | 会话暂停，状态已回滚 |
| `"unsupported"` | asset 类型，数据库已存 `result: "rejected"` |
| `"duplicate"` | 重复 URL，数据库已存 `result: "skipped_duplicate"` |
| `"accepted"` | 成功创建，数据库已存 `result: "accepted"`（status 仍为 processing） |
| `"failed"` | 处理失败，数据库已存 `result: "rejected"` |

---

## 各格式字段映射（修正版）

### 1. Netscape HTML (`html`)

**代码位置**: `parsers.ts:53-111`

| HTML 属性 | 内部字段 | 说明 |
|----------|---------|------|
| `<A>` 文本 | `title` | 书签标题 |
| `HREF` | `content.url` | 链接地址 |
| `ADD_DATE` | `addDate` | Unix 时间戳（秒） |
| `TAGS` | `tags` | 逗号分隔，`split(",")` |
| `<H3>` 路径 | `paths` | 递归遍历 `<DL><DT><H3>` 构建路径 |

**特殊处理**:
- 空文件夹名 → `"Unnamed"` (parsers.ts:72)
- 无 URL 的书签也会被保留（`content` 为 `undefined`），但会在暂存前被过滤

### 2. Pocket (`pocket`)

**代码位置**: `parsers.ts:113-135`

| CSV 列 | 内部字段 | 说明 |
|-------|---------|------|
| `title` | `title` | 标题 |
| `url` | `content.url` | 链接地址 |
| `time_added` | `addDate` | Unix 时间戳 |
| `tags` | `tags` | 管道符分隔，`split("|")` |
| `status` | `archived` | `status === "archive"` → `true` |

### 3. Readwise Reader (`readwise-reader`)

**代码位置**: `parsers.ts:498-576`

| CSV 列 | 内部字段 | 转换规则 |
|-------|---------|---------|
| `Title` | `title` | 直接映射 |
| `URL` | `content.url` | 直接映射 |
| `Document tags` | `tags` | 特殊引号转义（`\'` → `'`，`'` → `"`）后 `JSON.parse()` |
| `Saved date` | `addDate` | `new Date().getTime() / 1000` |
| `Location` | `archived` | `"archive"` → `true` |
| `Location` | 过滤 | `"feed"` → 排除 |

**特殊处理**:
- 空 URL 项目会被过滤（importWorker.ts 中还会再检查一次）
- 标签 JSON 解析失败时 `tags = []`，不抛出错误（parsers.ts:563-564）

### 4. Instapaper (`instapaper`)

**代码位置**: `parsers.ts:426-496`

| CSV 列 | 内部字段 | 转换规则 |
|-------|---------|---------|
| `Title` | `title` | 直接映射 |
| `URL` | `content.url` | 优先使用 URL |
| `Selection` | `content.text` | 无 URL 时使用 |
| `Timestamp` | `addDate` | `parseInt()` |
| `Tags` | `tags` | `JSON.parse()`，失败则 `[]` |
| `Folder` | `archived`/`paths` | `"Archive"` → `archived: true`; `"Unread"` → 无路径; 其他 → `paths: [[Folder]]` |

### 5. Karakeep 自有格式 (`karakeep`)

**代码位置**: `parsers.ts:183-238`

这是唯一支持完整列表结构（包括智能列表）的格式：

| JSON 字段 | 内部字段 | 说明 |
|----------|---------|------|
| `bookmarks[].title` | `title` | 直接映射 |
| `bookmarks[].content` | `content` | 判别联合类型（link/text） |
| `bookmarks[].tags` | `tags` | 直接映射 |
| `bookmarks[].createdAt` | `addDate` | 直接映射 |
| `bookmarks[].note` | `notes` | 直接映射 |
| `bookmarks[].archived` | `archived` | 直接映射 |
| `bookmarks[].lists` | `listExternalIds` | 仅保留 `manual` 类型列表，过滤智能列表 |
| `lists[]` | `lists` | 完整保留（含 `parentId`, `type`, `query`） |

**特殊处理**:
- 智能列表（`type: "smart"`）不与书签关联，仅作为列表元数据导入
- `listExternalIds` 过滤掉智能列表 ID

---

## 去重合并规则

**位置**: `parsers.ts:614-661` (`deduplicateBookmarks`)

**作用范围**: 仅**同一导入文件内**的去重（解析阶段），跨导入去重由 `createBookmark` 内部处理。

**去重键**: 仅对 `link` 类型按 URL 去重，`text` 类型不去重。

| 字段 | 合并策略 |
|-----|---------|
| `tags` | 合并去重：`[...new Set([...existing, ...new])]` |
| `paths` | 合并所有路径（数组元素合并） |
| `listExternalIds` | 合并去重 |
| `addDate` | 保留较早的日期 |
| `notes` | 两者都有时用 `\n---\n` 分隔追加 |
| `archived` | 任一为 `true` 则为 `true` |
| `title` | 保留先出现的 |

---

## 导入工作流详细步骤

### 阶段一：前端触发 (`useBookmarkImport.ts`)

```typescript
// 1. 预解析文件（用于配额检查）
const textContent = await file.text();
const parsedImport = parseImportFile(source, textContent);
const bookmarkCount = parsedImport.bookmarks.length;

// 2. 配额检查
const quotaUsage = await queryClient.fetchQuery(api.subscriptions.getQuotaUsage.queryOptions());
// 剩余配额不足则抛出错误

// 3. 调用导入主流程，传入自定义 parser 避免重复解析
const result = await importBookmarksFromFile(
  {
    file,
    source,
    rootListName: t("settings.import.imported_bookmarks"),
    deps: {
      createImportSession,      // tRPC mutation
      createList,               // tRPC mutation
      stageImportedBookmarks,   // tRPC mutation
      finalizeImportStaging,    // tRPC mutation
    },
    onProgress: (done, total) => setImportProgress({ done, total }),
  },
  {
    // ⚠️ 关键优化：自定义 parser 复用预解析结果，避免二次解析
    parsers: {
      [source]: () => parsedImport,
    },
  },
);
```

### 阶段二：解析与暂存 (`importer.ts`)

#### 目录结构重建

**方式 A：通过 `paths` 构建（大多数格式）**

```typescript
// 路径分隔符："$$__$$"（importer.ts:151）
const PATH_DELIMITER = "$$__$$";

// 1. 收集所有需要的路径（含所有父路径，确保中间目录也被创建）
for (const bookmark of bookmarksWithPathMembership) {
  for (const path of bookmark.paths) {
    for (let i = 1; i <= path.length; i++) {
      const subPath = path.slice(0, i); // 例：["a","b","c"] → ["a"], ["a","b"], ["a","b","c"]
      allRequiredPaths.set(getPathKey(subPath), folderName);
    }
  }
}

// 2. 按路径深度排序，确保先创建父目录
.sort((a, b) => a.pathKey.split(PATH_DELIMITER).length - 
                b.pathKey.split(PATH_DELIMITER).length)

// 3. 逐级创建列表
for (const { pathKey, folderName } of allRequiredPathsArray) {
  const parts = pathKey.split(PATH_DELIMITER);
  const parentKey = parts.slice(0, -1).join(PATH_DELIMITER);
  const parentId = pathMap[parentKey] || rootList.id;
  
  const folderList = await deps.createList({
    name: folderName.substring(0, MAX_LIST_NAME_LENGTH),
    parentId,
    icon: "📁",
  });
  pathMap[pathKey] = folderList.id;
}
```

**方式 B：通过 `lists` + `listExternalIds` 构建（仅 Karakeep）**

```typescript
// 拓扑排序处理有依赖的列表创建
while (unresolvedLists.size > 0) {
  for (const [externalId, list] of unresolvedLists) {
    // 父列表未创建则跳过，等下一轮
    if (list.parentExternalId && !externalListIdToCreatedListId[list.parentExternalId]) {
      continue;
    }
    
    const parentId = list.parentExternalId 
      ? externalListIdToCreatedListId[list.parentExternalId] 
      : rootList.id;
    
    const createdList = await deps.createList({
      name: list.name.substring(0, MAX_LIST_NAME_LENGTH),
      parentId,
      icon: list.icon ?? "📁",
      description: list.description,
      ...(list.type === "smart" && list.query ? { type: "smart", query: list.query } : {}),
    });
    
    externalListIdToCreatedListId[externalId] = createdList.id;
    unresolvedLists.delete(externalId);
    createdAny = true;
  }
  
  // 循环引用或父缺失的容错：挂到根目录
  if (!createdAny) {
    for (const [externalId, list] of unresolvedLists) {
      const createdList = await deps.createList({
        name: list.name.substring(0, MAX_LIST_NAME_LENGTH),
        parentId: rootList.id,  // 强制根目录
        // ...
      });
    }
    unresolvedLists.clear();
  }
}
```

#### 书签关联列表优先级

```typescript
// listExternalIds 优先于 paths（importer.ts:225-228）
const listIds =
  listIdsFromExternalListIds.length > 0
    ? listIdsFromExternalListIds
    : listIdsFromPaths;
```

#### 批量暂存

- 批次大小：50 条 (importer.ts:261)
- 暂存状态：`pending`
- `finalizeImportStaging` 后会话状态从 `staging` → `pending`，Worker 开始处理

### 阶段三：后台 Worker 处理 (`importWorker.ts`)

#### 轮询循环

```typescript
while (this.running) {
  // 1. 每 60 次轮询（≈5分钟）重置挂起项
  if (iterationCount % 60 === 0) {
    await this.resetStaleProcessingItems();
  }
  
  // 2. 每次轮询先检查已完成下游处理的项目
  await this.checkAndCompleteProcessingItems();
  
  // 3. 处理一个批次
  const processed = await this.processBatch();
  
  if (processed === 0) {
    // 4. 无任务时检查空闲会话并完成
    await this.checkAndCompleteIdleSessions();
    await this.updateGauges();
    await sleep(this.pollIntervalMs); // 5秒
  }
}
```

#### 单条书签处理 (`processOneBookmark`)

```typescript
// 1. 会话检查：已暂停则重置回 pending
if (!session || session.status === "paused") {
  await db.update(importStagingBookmarks)
    .set({ status: "pending" })
    .where(eq(importStagingBookmarks.id, staged.id));
  return "reset";
}

// 2. 验证类型并构建请求
if (staged.type === "link") {
  if (!staged.url) throw new Error("URL is required for link bookmarks");
  bookmarkRequest = { type: BookmarkTypes.LINK, url: staged.url, ... };
} else if (staged.type === "text") {
  if (!staged.content) throw new Error("Content is required for text bookmarks");
  bookmarkRequest = { type: BookmarkTypes.TEXT, text: staged.content, ... };
} else {
  // asset 类型 → 标记为 failed
  await db.update(importStagingBookmarks)
    .set({
      status: "failed",
      result: "rejected",  // ⚠️ 不是 "unsupported"
      resultReason: "Asset bookmarks not yet supported",
      completedAt: new Date(),
    });
  return "unsupported"; // 代码内部标记
}

// 3. 创建书签（内部已含跨导入去重）
const result = await caller.bookmarks.createBookmark(bookmarkRequest);

// 4. 应用标签（重复书签也应用标签）
if (staged.tags && staged.tags.length > 0) {
  await caller.bookmarks.updateTags({
    bookmarkId: result.id,
    attach: staged.tags.map((t) => ({ tagName: t })),
    detach: [],
  });
}

// 5. 处理重复
if (result.alreadyExists) {
  await db.update(importStagingBookmarks)
    .set({
      status: "completed",
      result: "skipped_duplicate",
      resultReason: "URL already exists",
      resultBookmarkId: result.id,
      completedAt: new Date(),
    });
  await this.attachBookmarkToLists(caller, session, staged, result.id);
  return "duplicate";
}

// 6. 成功创建，标记为 accepted 但保持 processing 状态
await db.update(importStagingBookmarks)
  .set({
    result: "accepted",
    resultBookmarkId: result.id,
    // ⚠️ status 仍为 "processing"，不设置 completedAt
  });
await this.attachBookmarkToLists(caller, session, staged, result.id);
return "accepted";
```

#### 列表关联逻辑 (`attachBookmarkToLists`)

```typescript
const listIds = new Set<string>();

// ⚠️ 同时关联 rootListId + staged.listIds
if (session.rootListId) {
  listIds.add(session.rootListId);
}
if (staged.listIds && staged.listIds.length > 0) {
  for (const listId of staged.listIds) {
    listIds.add(listId);
  }
}

// 单条失败不影响整体
for (const listId of listIds) {
  try {
    await caller.lists.addToList({ listId, bookmarkId });
  } catch (error) {
    logger.warn(`[import] Failed to add bookmark to list: ${error}`);
  }
}
```

---

## 导出功能

### 1. Karakeep JSON 格式 (`exporters.ts:42-91`)

```typescript
// 书签导出
toExportFormat(bookmark, listIds): {
  createdAt: Math.floor(bookmark.createdAt.getTime() / 1000),
  title: bookmark.title ?? (link type ? content.title : null),
  tags: bookmark.tags.map(t => t.name),
  lists: listIds ?? [],
  content: { type: "link" | "text", ... },
  note: bookmark.note ?? null,
  archived: bookmark.archived,
}

// 列表导出
toExportListFormat(list): {
  id, name, description, icon, type, query, parentId
}
```

### 2. Netscape HTML 格式 (`exporters.ts:93-127`)

```typescript
toNetscapeFormat(bookmarks): string {
  // 仅导出 link 类型书签，text 类型被忽略
  if (bookmark.content?.type !== BookmarkTypes.LINK) {
    return "";
  }
  
  // 输出格式：
  <DT><A HREF="{url}" ADD_DATE="{timestamp}" TAGS="{tags}">{title}</A>
}
```

---

## 代码位置索引

| 功能 | 文件 | 关键函数/类 | 行号 |
|------|------|------------|------|
| 格式解析入口 | `packages/shared/import-export/parsers.ts` | `parseImportFile` | 663-709 |
| Netscape 解析 | `parsers.ts` | `parseNetscapeBookmarkFile` | 53-111 |
| 去重合并 | `parsers.ts` | `deduplicateBookmarks` | 614-661 |
| 导入主流程 | `packages/shared/import-export/importer.ts` | `importBookmarksFromFile` | 56-286 |
| 导出功能 | `packages/shared/import-export/exporters.ts` | `toExportFormat`, `toNetscapeFormat` | 42-127 |
| 前端导入 Hook | `apps/web/lib/hooks/useBookmarkImport.ts` | `useBookmarkImport` | 23-132 |
| 导入会话服务 | `packages/trpc/models/importSessions.service.ts` | `ImportSessionsService` | 17-206 |
| 导入会话仓储 | `packages/trpc/models/importSessions.repo.ts` | `ImportSessionsRepo` | 14-124 |
| 后台导入 Worker | `apps/workers/workers/importWorker.ts` | `ImportWorker` | 102-718 |
| 安全错误信息 | `importWorker.ts` | `getSafeErrorMessage` | 82-100 |
| 挂起项重置 | `importWorker.ts` | `resetStaleProcessingItems` | 682-718 |
| 下游完成检查 | `importWorker.ts` | `checkAndCompleteProcessingItems` | 546-650 |
| 单条处理 | `importWorker.ts` | `processOneBookmark` | 351-487 |
| 批次处理 | `importWorker.ts` | `processBatch` | 149-233 |
| 数据库 Schema | `packages/db/schema.ts` | `importSessions`, `importStagingBookmarks` | 851-950 |
| 导入类型定义 | `packages/shared/types/importSessions.ts` | - | 1-78 |

---

## 常见误解澄清

| 误解 | 事实 |
|------|------|
| 结果类型有 `"unsupported"` | 数据库 `result` 枚举只有 `accepted`/`rejected`/`skipped_duplicate`，`"unsupported"` 只是函数返回值，实际存的是 `rejected` |
| 成功创建书签后状态变为 `completed` | 书签创建成功后状态仍为 `processing`，需等 crawl/tagging 完成后才由 `checkAndCompleteProcessingItems` 标记为 `completed` |
| 处理失败后会自动重试 | 只有 `resultBookmarkId IS NULL` 的挂起项会被重置，已创建书签的项目（哪怕下游失败）不会重试 |
| `listExternalIds` 和 `paths` 是二选一 | `listExternalIds` 优先级高于 `paths`，但 `attachBookmarkToLists` 还会额外加上 `session.rootListId` |
| 去重只在导入时做一次 | 两次去重：解析阶段同一文件内去重 + `createBookmark` 跨导入去重 |
| `checkAndCompleteProcessingItems` 在批次后调用 | **每次轮询最先调用**，在 `processBatch` 之前 |
| 挂起项每小时检查一次 | 每 **5 分钟** 检查一次（60次轮询 × 5秒） |
