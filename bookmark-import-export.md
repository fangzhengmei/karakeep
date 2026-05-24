# 书签导入导出转换规则分析

## 概述

Karakeep 的书签导入导出系统位于 `packages/shared/import-export/`，采用 **"解析器 + 处理器"** 架构设计，支持 11 种外部格式的导入和 2 种格式的导出。核心组件包括：

- **`parsers.ts`**: 格式解析器，负责将各种外部格式转换为内部统一表示
- **`importer.ts`**: 导入处理器，负责目录结构重建、去重、批量暂存
- **`exporters.ts`**: 导出处理器，将内部格式转换为外部格式
- **`importWorker.ts`**: 后台 Worker，负责异步处理实际导入

---

## 支持的格式列表

### 导入格式 (`ImportSource`)

| 格式类型 | 文件类型 | 源格式 | 代码位置 |
|---------|---------|--------|---------|
| `html` | HTML | Netscape Bookmark File | `parseNetscapeBookmarkFile` |
| `pocket` | CSV | Pocket Export | `parsePocketBookmarkFile` |
| `matter` | CSV | Matter Export | `parseMatterBookmarkFile` |
| `omnivore` | JSON | Omnivore Export | `parseOmnivoreBookmarkFile` |
| `karakeep` | JSON | Karakeep 自有格式 | `parseKarakeepBookmarkFile` |
| `linkwarden` | JSON | Linkwarden Export | `parseLinkwardenBookmarkFile` |
| `tab-session-manager` | JSON | Tab Session Manager | `parseTabSessionManagerStateFile` |
| `mymind` | CSV | mymind Export | `parseMymindBookmarkFile` |
| `readwise-reader` | CSV | Readwise Reader | `parseReadwiseReaderBookmarkFile` |
| `instapaper` | CSV | Instapaper Export | `parseInstapaperBookmarkFile` |
| `onetab` | TXT | OneTab Export | `parseOneTabFile` |

### 导出格式

| 格式类型 | 说明 | 代码位置 |
|---------|------|---------|
| Karakeep JSON | 自有完整格式（含列表结构） | `toExportFormat`, `toExportListFormat` |
| Netscape HTML | 浏览器通用书签格式 | `toNetscapeFormat` |

---

## 格式探测与解析流程

### 1. 格式探测机制

**注意：Karakeep 的格式探测是**用户驱动**而非自动探测。**

用户在导入界面选择来源格式后，系统通过 `parseImportFile(source, textContent)` 函数调度到对应解析器：

```typescript
// parsers.ts:663-709
export function parseImportFile(
  source: ImportSource,
  textContent: string,
): ParsedImportFile {
  if (source === "karakeep") {
    const parsed = parseKarakeepBookmarkFile(textContent);
    return {
      bookmarks: deduplicateBookmarks(parsed.bookmarks),
      lists: parsed.lists,
    };
  }

  let result: ParsedBookmark[];
  switch (source) {
    case "html":
      result = parseNetscapeBookmarkFile(textContent);
      break;
    // ... 其他格式
  }
  return { bookmarks: deduplicateBookmarks(result), lists: [] };
}
```

**各格式的格式验证：**

- **Netscape HTML**: 检查文件头 `<!DOCTYPE NETSCAPE-Bookmark-file-1>` (parsers.ts:54)
- **JSON 格式**: 使用 Zod schema 做严格验证（如 `zOmnivoreExportSchema`, `zLinkwardenExportSchema`）
- **CSV 格式**: 使用 `csv-parse/sync` 解析 + Zod schema 验证
- **OneTab TXT**: 按行解析，跳过非 URL 行

### 2. 解析后统一数据结构

所有外部格式解析后统一为 `ParsedImportFile` 结构：

```typescript
interface ParsedImportFile {
  bookmarks: ParsedBookmark[];   // 书签列表
  lists: ParsedImportList[];     // 列表/文件夹结构（仅 Karakeep 格式）
}

interface ParsedBookmark {
  title: string;
  content?: { type: "link"; url: string } | { type: "text"; text: string };
  tags: string[];
  addDate?: number;              // Unix 时间戳（秒）
  notes?: string;
  archived?: boolean;
  paths: string[][];             // 文件夹路径，支持多路径（如重复URL在多个文件夹）
  listExternalIds?: string[];    // 外部列表ID（仅 Karakeep 格式）
}
```

---

## 字段映射规则详解

### 通用字段映射表

| 内部字段 | 来源字段说明 |
|---------|-------------|
| `title` | 标题/名称 |
| `content` | 链接或文本内容 |
| `tags` | 标签数组 |
| `addDate` | 添加时间（Unix 秒级时间戳） |
| `notes` | 备注/笔记 |
| `archived` | 是否已归档 |
| `paths` | 文件夹路径数组（支持多层嵌套） |

---

### 各格式详细字段映射

#### 1. Netscape HTML (`html`)

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
- 支持任意深度嵌套文件夹
- 无 URL 的书签也会被保留（`content` 为 `undefined`）

---

#### 2. Pocket (`pocket`)

**代码位置**: `parsers.ts:113-135`

| CSV 列 | 内部字段 | 说明 |
|-------|---------|------|
| `title` | `title` | 标题 |
| `url` | `content.url` | 链接地址 |
| `time_added` | `addDate` | Unix 时间戳 |
| `tags` | `tags` | 管道符分隔，`split("|")` |
| `status` | `archived` | `status === "archive"` → `true` |

---

#### 3. Matter (`matter`)

**代码位置**: `parsers.ts:137-181`

| CSV 列 | 内部字段 | 转换规则 |
|-------|---------|---------|
| `Title` | `title` | 直接映射 |
| `URL` | `content.url` | 直接映射 |
| `Tags` | `tags` | 分号分隔，`split(";")` |
| `Last Interaction Date` | `addDate` | `Date.parse(date) / 1000` |
| `In Queue` | `archived` | `"False"` → `true` |

---

#### 4. Omnivore (`omnivore`)

**代码位置**: `parsers.ts:240-268`

| JSON 字段 | 内部字段 | 转换规则 |
|----------|---------|---------|
| `title` | `title` | 直接映射 |
| `url` | `content.url` | 直接映射 |
| `labels` | `tags` | 直接映射（数组） |
| `savedAt` | `addDate` | `z.coerce.date()` → `.getTime() / 1000` |
| `state` | `archived` | `state === "Archived"` → `true` |

---

#### 5. Linkwarden (`linkwarden`)

**代码位置**: `parsers.ts:270-324`

| JSON 路径 | 内部字段 | 转换规则 |
|----------|---------|---------|
| `links[].name` | `title` | 直接映射 |
| `links[].url` | `content.url` | 直接映射 |
| `links[].tags[].name` | `tags` | 提取 `name` 字段 |
| `links[].createdAt` | `addDate` | `.getTime() / 1000` |
| `collections` 层级 | `paths` | 递归 `parentId` 构建完整路径 |

**特殊处理**:
- 按 `collections[].id` → `parentId` 递归构建路径 (`getCollectionPath`)
- 同一 URL 在多个集合中会被去重，路径合并

---

#### 6. Instapaper (`instapaper`)

**代码位置**: `parsers.ts:426-496`

| CSV 列 | 内部字段 | 转换规则 |
|-------|---------|---------|
| `Title` | `title` | 直接映射 |
| `URL` | `content.url` | 优先使用 URL |
| `Selection` | `content.text` | 无 URL 时使用文本内容 |
| `Timestamp` | `addDate` | `parseInt()` |
| `Tags` | `tags` | `JSON.parse()`，失败则空数组 |
| `Folder` | `archived`/`paths` | `"Archive"` → `archived: true`; `"Unread"` → 无路径; 其他 → `paths: [[Folder]]` |

---

#### 7. Readwise Reader (`readwise-reader`)

**代码位置**: `parsers.ts:498-576`

| CSV 列 | 内部字段 | 转换规则 |
|-------|---------|---------|
| `Title` | `title` | 直接映射 |
| `URL` | `content.url` | 直接映射 |
| `Document tags` | `tags` | 特殊引号转义后 `JSON.parse()` |
| `Saved date` | `addDate` | `new Date().getTime() / 1000` |
| `Location` | `archived` | `"archive"` → `true` |
| `Location` | 过滤 | `"feed"` → 过滤排除 |

**特殊处理**:
- 标签字段需要转义：`\'` → `'`，`'` → `"` 等 (parsers.ts:519-524)
- 自动过滤 RSS feed 项目 (`Location !== "feed"`)
- 过滤空 URL 项目

---

#### 8. mymind (`mymind`)

**代码位置**: `parsers.ts:365-424`

| CSV 列 | 内部字段 | 转换规则 |
|-------|---------|---------|
| `title` | `title` | 直接映射 |
| `url` | `content.url` | 优先使用 URL |
| `content` | `content.text` | 无 URL 时使用文本 |
| `tags` | `tags` | 逗号分隔，`split(",")` |
| `note` | `notes` | 直接映射 |
| `created` | `addDate` | `new Date().getTime() / 1000` |

**类型判定**:
```
有 URL → type: "link"
无 URL 但有 content → type: "text"
```

---

#### 9. OneTab (`onetab`)

**代码位置**: `parsers.ts:578-612`

| 格式 | 内部字段 | 说明 |
|------|---------|------|
| `URL \| Title` 行 | `content.url`, `title` | 按 ` \| ` 分割 |
| 单独 `URL` 行 | `content.url` | `title` 为空 |
| 非 `http://` 或 `https://` | - | 跳过该行 |

---

#### 10. Karakeep 自有格式 (`karakeep`)

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
| `bookmarks[].lists` | `listExternalIds` | 仅保留 `manual` 类型列表 |
| `lists[]` | `lists` | 完整保留列表结构（含 `parentId`, `type`, `query`） |

**特殊处理**:
- 智能列表（`type: "smart"`）不与书签关联，仅作为列表元数据导入
- `listExternalIds` 过滤掉智能列表 ID

---

### 重复数据合并规则

**代码位置**: `parsers.ts:614-661` (`deduplicateBookmarks`)

按 URL 去重（文本书签不做去重），合并策略：

| 字段 | 合并策略 |
|-----|---------|
| `tags` | 合并去重：`[...new Set([...existing, ...new])]` |
| `paths` | 合并所有路径 |
| `listExternalIds` | 合并去重 |
| `addDate` | 保留较早的日期 |
| `notes` | 两者都有时用 `\n---\n` 分隔追加 |
| `archived` | 任一为 `true` 则为 `true` |
| `title` | 保留先出现的 |

---

## 导入工作流

### 整体流程

```
用户上传文件 → 选择格式 → 解析文件 → 重建目录结构 → 批量暂存 → 
后台Worker处理 → 实际创建书签 → 关联列表/标签 → 完成
```

### 阶段一：解析与暂存 (`importer.ts`)

**代码位置**: `importBookmarksFromFile` (importer.ts:56-286)

#### 1. 创建导入根列表
```typescript
const rootList = await deps.createList({ name: rootListName, icon: "⬆️" });
```

#### 2. 重建目录结构

**方式 A：通过 `paths` 构建（大多数格式）**

```typescript
// 路径分隔符："$$__$$"
const PATH_DELIMITER = "$$__$$";

// 1. 收集所有需要的路径（含所有父路径）
for (const bookmark of bookmarksWithPathMembership) {
  for (const path of bookmark.paths) {
    for (let i = 1; i <= path.length; i++) {
      const subPath = path.slice(0, i); // 确保父目录也被创建
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
    // 父列表未创建则跳过
    if (list.parentExternalId && !externalListIdToCreatedListId[list.parentExternalId]) {
      continue;
    }
    
    // 创建列表
    const createdList = await deps.createList({
      name: list.name.substring(0, MAX_LIST_NAME_LENGTH),
      parentId: list.parentExternalId ? externalListIdToCreatedListId[list.parentExternalId] : rootList.id,
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
        parentId: rootList.id,  // 强制挂到根
        // ...
      });
    }
    unresolvedLists.clear();
  }
}
```

#### 3. 书签关联列表优先级

```typescript
// listExternalIds 优先于 paths
const listIds =
  listIdsFromExternalListIds.length > 0
    ? listIdsFromExternalListIds
    : listIdsFromPaths;
```

#### 4. 批量暂存

- 批次大小：50 条 (importer.ts:261)
- 暂存表：`import_staging_bookmarks`
- 状态：`pending`

---

### 阶段二：后台 Worker 处理 (`importWorker.ts`)

**代码位置**: `apps/workers/workers/importWorker.ts`

#### 状态机

```
staging → pending → running → completed
                    ↓         ↓
                  paused    failed
```

#### 并发控制与公平调度

```typescript
// 最大同时处理数：50
private maxInFlight = 50;
// 批次大小：10
private batchSize = 10;

// 公平调度：按用户 lastProcessedAt 排序，再按创建时间
.orderBy(importSessions.lastProcessedAt, importStagingBookmarks.createdAt)

// 原子声明：防止多 Worker 竞争
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

#### 单条书签处理流程

```typescript
// 1. 构建创建请求
const baseRequest = {
  title: normalizedTitle?.trim().substring(0, MAX_BOOKMARK_TITLE_LENGTH),
  note: staged.note ?? undefined,
  createdAt: staged.sourceAddedAt ?? undefined,
  crawlPriority: "low",
  archived: staged.archived ?? false,
  source: "import",
};

// 2. 调用 createBookmark（内部已含重复检测）
const result = await caller.bookmarks.createBookmark(bookmarkRequest);

// 3. 应用标签
if (staged.tags && staged.tags.length > 0) {
  await caller.bookmarks.updateTags({
    bookmarkId: result.id,
    attach: staged.tags.map((t) => ({ tagName: t })),
    detach: [],
  });
}

// 4. 关联列表
for (const listId of listIds) {
  try {
    await caller.lists.addToList({ listId, bookmarkId });
  } catch (error) {
    // 单个列表关联失败不影响整体
    logger.warn(`Failed to add bookmark to list: ${error}`);
  }
}
```

---

## 失败容忍与容错处理

### 1. 导入前过滤 (`importSessions.service.ts:95-100`)

暂存前过滤无效书签：
```typescript
const validBookmarks = bookmarks.filter((bookmark) => {
  if (bookmark.type === "link" && !bookmark.url) return false;
  if (bookmark.type === "text" && !bookmark.content) return false;
  return true;
});
```

### 2. 结果类型

| `result` | `status` | 说明 |
|----------|----------|------|
| `accepted` | `completed` | 成功导入 |
| `skipped_duplicate` | `completed` | URL 已存在，跳过但仍关联列表和标签 |
| `rejected` | `failed` | 处理失败 |
| `unsupported` | `failed` | asset 类型暂不支持 |

### 3. 错误信息安全过滤 (`importWorker.ts:82-100`)

```typescript
function getSafeErrorMessage(error: unknown): string {
  // TRPCError 非 INTERNAL_SERVER_ERROR 可直接展示
  if (error instanceof TRPCError && error.code !== "INTERNAL_SERVER_ERROR") {
    return error.message;
  }
  
  // 已知安全的验证错误
  if (error instanceof Error) {
    const safeMessages = [
      "URL is required for link bookmarks",
      "Content is required for text bookmarks",
    ];
    if (safeMessages.includes(error.message)) {
      return error.message;
    }
  }
  
  return "An unexpected error occurred while processing the bookmark";
}
```

### 4. 挂起项重试机制 (`importWorker.ts:682-718`)

```typescript
// 超时阈值：1小时
private staleThresholdMs = 60 * 60 * 1000;

// 每 60 次轮询（约 5 分钟）检查一次
if (iterationCount % 60 === 0) {
  await this.resetStaleProcessingItems();
}

// 重置条件：
// 1. status === "processing"
// 2. processingStartedAt < 1小时前
// 3. resultBookmarkId IS NULL（已创建书签的不算挂起）
```

### 5. 下游处理等待 (`importWorker.ts:546-650`)

书签创建后，需等待爬虫和自动标签完成才算真正完成：

```typescript
// crawlStatus 和 taggingStatus 都不为 pending 才标记完成
or(
  isNull(bookmarkLinks.crawlStatus),
  eq(bookmarkLinks.crawlStatus, "success"),
  eq(bookmarkLinks.crawlStatus, "failure"),
),
or(
  isNull(bookmarks.taggingStatus),
  eq(bookmarks.taggingStatus, "success"),
  eq(bookmarks.taggingStatus, "failure"),
)
```

### 6. 列表关联容错

单条列表关联失败只打 warn 日志，不影响书签整体状态：
```typescript
try {
  await caller.lists.addToList({ listId, bookmarkId });
} catch (error) {
  logger.warn(`[import] Failed to add bookmark ${bookmarkId} to list ${listId}: ${error}`);
}
```

### 7. 循环引用或父列表缺失

拓扑排序中遇到无法解析的父引用时，强制挂到导入根列表：
```typescript
if (!createdAny) {
  for (const [externalId, list] of unresolvedLists) {
    const createdList = await deps.createList({
      // ...
      parentId: rootList.id,  // 强制根目录
    });
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
  // 仅导出 link 类型书签
  if (bookmark.content?.type !== BookmarkTypes.LINK) {
    return "";
  }
  
  // 格式：
  <DT><A HREF="{url}" ADD_DATE="{timestamp}" TAGS="{tags}">{title}</A>
}
```

---

## 代码位置索引

| 功能 | 文件 | 关键函数/类 |
|------|------|------------|
| 格式解析入口 | `packages/shared/import-export/parsers.ts` | `parseImportFile` |
| Netscape 解析 | `parsers.ts` | `parseNetscapeBookmarkFile` |
| Pocket 解析 | `parsers.ts` | `parsePocketBookmarkFile` |
| Linkwarden 解析 | `parsers.ts` | `parseLinkwardenBookmarkFile` |
| Readwise Reader 解析 | `parsers.ts` | `parseReadwiseReaderBookmarkFile` |
| Instapaper 解析 | `parsers.ts` | `parseInstapaperBookmarkFile` |
| Karakeep 解析 | `parsers.ts` | `parseKarakeepBookmarkFile` |
| 去重合并 | `parsers.ts` | `deduplicateBookmarks` |
| 导入主流程 | `packages/shared/import-export/importer.ts` | `importBookmarksFromFile` |
| 导出功能 | `packages/shared/import-export/exporters.ts` | `toExportFormat`, `toNetscapeFormat` |
| 导入会话服务 | `packages/trpc/models/importSessions.service.ts` | `ImportSessionsService` |
| 后台导入 Worker | `apps/workers/workers/importWorker.ts` | `ImportWorker` |
| 导入类型定义 | `packages/shared/types/importSessions.ts` | - |
