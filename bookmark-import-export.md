# 书签导入导出转换规则分析（三次修正版）

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
    ├─ 预解析文件统计数量（用于配额检查）
    ├─ 检查配额（quotaUsage）
    └─ 调用 importBookmarksFromFile
        ↓
[importer.ts] importBookmarksFromFile
    ├─ 解析文件（parseImportFile）⚠️ fail-fast，格式错误直接抛出
    ├─ 创建导入根列表
    ├─ 重建目录结构（paths 或 listExternalIds）
    ├─ 转换为 StagedBookmark
    ├─ 批量暂存（50条/批）⚠️ 暂存前会过滤无效书签
    └─ finalize → 会话状态从 staging → pending
        ↓
[importWorker.ts] ImportWorker（轮询，空闲时5秒/次，繁忙时无间隔）
    ├─ resetStaleProcessingItems（每60次轮询，频率取决于系统负载）
    ├─ checkAndCompleteProcessingItems（每次轮询先执行）
    ├─ processBatch
    │   ├─ getAvailableCapacity（背压控制，隐性判断 stale）
    │   ├─ 公平调度取候选
    │   ├─ 原子声明为 processing
    │   ├─ processOneBookmark（并行处理）
    │   │   ├─ 二次校验空 URL/空内容
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

> ⚠️ **重要发现**：`"failed"` 是 schema 定义的枚举值，但**当前代码中没有任何路径能让 session 达到此状态**。详见下文"Session 级别 failed 状态分析"。

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

**路径 A：创建失败 / 空 URL / 空内容**（importWorker.ts:469-485）
```typescript
status: "failed"
result: "rejected"
resultReason: getSafeErrorMessage(error)  // 如 "URL is required for link bookmarks"
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

## 失败容忍与重试机制（修正版）

### 1. 空 URL / 空内容的多入口判定路径

空 URL 和空内容的处理**取决于数据进入系统的入口**，共有 4 个可能的入口，产生 3 种不同结果：

```
                                 数据来源
                                     │
            ┌───────────┬────────────┼────────────┬───────────┐
            │           │            │            │           │
            ▼           ▼            ▼            ▼           ▼
      正常导入      直接调用      直接调用    Parser 内部   Parser 内部
   importBookmarks stageBookmarks insertStaging   过滤       产生
     FromFile      service          repo          (RW)    content:undefined
          │            │              │            │           │
          │            │              │            │           │
          ├────────────┘              │            ▼           │
          │                           │         丢弃，        │
          ▼                           │       不进入流程     │
  暂存前过滤                          │                        │
          │                           │                        │
          ├───────────────────────────┘                        │
          │                                                    │
          ▼                                                    │
    过滤丢弃                                               传递到
  无失败记录                                             暂存前过滤
                                                               │
                                                               ▼
                                                          过滤丢弃
                                                        无失败记录
                                              （除非暂存前过滤被绕过）
                                                               │
                                                               └──────┐
                                                                      │
                                                                      ▼
                                                          Worker 二次校验
                                                                      │
                                                                      ▼
                                                          抛出错误 → catch
                                                                      │
                                                                      ▼
                                                          status: "failed"
                                                          result: "rejected"
                                                          resultReason: 白名单安全消息
                                                          completedAt: now
                                                          （产生可见失败记录）
```

---

#### 入口 A：正常导入流程（`importer.ts` → `stageBookmarks`）

**路径**：
1. `parseImportFile()` 解析文件
   - Readwise Reader 内部会过滤空 URL（parsers.ts:542-544）
   - Instapaper 中 URL 和 Selection 都为空时产生 `content: undefined`
   - Pocket 无 URL 的行会产生 `content.url: ""`
2. `importBookmarksFromFile()` 调用 `stageBookmarks`
3. `stageBookmarks` 暂存前过滤（importSessions.service.ts:95-100）：
```typescript
const validBookmarks = bookmarks.filter((bookmark) => {
  if (bookmark.type === "link" && !bookmark.url) return false;
  if (bookmark.type === "text" && !bookmark.content) return false;
  return true;
});
```

**结果**：**静默丢弃**，不产生任何数据库记录，`stats.totalBookmarks` 不包含这些项。

---

#### 入口 B：直接调用 `stageBookmarks` API（绕过 parser）

**路径**：
1. 调用者直接构造 `StagedBookmark` 数据
2. 调用 `importSessions.stageBookmarks`
3. 暂存前过滤仍会执行（同上）

**结果**：**静默丢弃**，与入口 A 相同。

---

#### 入口 C：直接调用 `insertStagingBookmarks` repo 层（绕过 service 层过滤）

**路径**：
1. 调用者直接构造 `importStagingBookmarks` 数据库记录（含空 URL）
2. 调用 `importSessionsRepo.insertStagingBookmarks`
3. 不经过 service 层过滤，直接入库
4. Worker 轮询到该记录，二次校验：
```typescript
if (staged.type === "link") {
  if (!staged.url) {
    throw new Error("URL is required for link bookmarks");
  }
}
```
5. 错误被 `processOneBookmark` 的 try-catch 捕获（importWorker.ts:469-485）
6. 标记为失败状态

**结果**：**产生可见失败记录**：
- `status: "failed"`
- `result: "rejected"`
- `resultReason: "URL is required for link bookmarks"`（通过 `getSafeErrorMessage` 白名单）
- `completedAt: now`

---

#### 入口 D：Parser 内部产生 `content: undefined`（如 Instapaper）

**路径**：
1. Instapaper 中某条记录的 `URL` 和 `Selection` 都为空
2. Parser 逻辑（parsers.ts:452-460）：
```typescript
let content: ParsedBookmark["content"];
if (record.URL && record.URL.trim().length > 0) {
  content = { type: BookmarkTypes.LINK, url: record.URL.trim() };
} else if (record.Selection && record.Selection.trim().length > 0) {
  content = { type: BookmarkTypes.TEXT, text: record.Selection.trim() };
}
// 两者都为空 → content = undefined
```
3. `type` 默认是 `"link"`（由 importer.ts 推断），但 `url` 是 `undefined`
4. 进入暂存前过滤 → `bookmark.type === "link" && !bookmark.url` → `return false`

**结果**：**静默丢弃**，与入口 A 相同。

---

#### 入口 E：Readwise Reader parser 内部过滤空 URL

**路径**（parsers.ts:542-544）：
```typescript
const emptyFilteredArticles = feedFilteredArticles.filter(
  (record) => record.URL && record.URL.trim().length > 0,
);
```

**结果**：**在 parser 层就被过滤**，不会传递到后续流程，`counts.total` 也不包含这些项。

---

### 关键区别总结

| 入口 | 空 URL 处理方式 | 失败记录 | 用户可见 |
|------|----------------|---------|---------|
| 正常导入流程 | 暂存前过滤，静默丢弃 | ❌ 无 | ❌ |
| 直接调用 `stageBookmarks` | 暂存前过滤，静默丢弃 | ❌ 无 | ❌ |
| 直接调用 repo 层 | Worker 二次校验，标记失败 | ✅ 有 | ✅ |
| Instapaper `content: undefined` | 暂存前过滤，静默丢弃 | ❌ 无 | ❌ |
| Readwise Reader 内部过滤 | Parser 内过滤，不进入后续 | ❌ 无 | ❌ |

### 2. Worker 阶段二次校验（深层防御）

**位置**: `importWorker.ts:392-404`

这是**深层防御**机制，即使上游所有过滤都被绕过，Worker 中仍会校验：
```typescript
if (staged.type === "link") {
  if (!staged.url) {
    throw new Error("URL is required for link bookmarks");
  }
} else if (staged.type === "text") {
  if (!staged.content) {
    throw new Error("Content is required for text bookmarks");
  }
}
```

**失败处理路径**：
- 抛出的错误被 `processOneBookmark` 的 try-catch 捕获
- 标记为 `status: "failed"`, `result: "rejected"`
- `resultReason` 是白名单内的安全信息

> **白名单机制**（importWorker.ts:89-93）：
> 只有 `"URL is required for link bookmarks"` 和 `"Content is required for text bookmarks"` 会原样返回给用户，其他错误会被替换为通用错误消息，避免泄露内部实现细节。

### 3. 处理过程中的错误隔离

- **`Promise.allSettled`**（importWorker.ts:213）：单条失败不影响批次
- **列表关联容错**（importWorker.ts:341-347）：单个列表关联失败只打 warn，不影响书签状态
- **错误信息安全过滤**（importWorker.ts:82-100）：避免泄露堆栈、数据库错误等内部详情给用户

### 4. 挂起项重试机制（双重判断）

#### 4.1 显性重试：`resetStaleProcessingItems()`

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

**触发频率**: 每 60 次轮询，但间隔不固定（importWorker.ts:120）：
- 空闲时（`processed === 0`）：每次轮询间隔 5 秒 → 60 次 = **5分钟**
- 繁忙时（`processed > 0`）：不 sleep，轮询连续执行 → 60 次可能只需 **几秒到几十秒**

#### 4.2 隐性重试：`getAvailableCapacity()` 背压控制

**位置**: `importWorker.ts:655-673`

```typescript
const processingCount = await db
  .select({ count: count() })
  .from(importStagingBookmarks)
  .where(
    and(
      eq(importStagingBookmarks.status, "processing"),
      // ⚠️ 只统计最近1小时内开始处理的
      gt(importStagingBookmarks.processingStartedAt, new Date(Date.now() - this.staleThresholdMs)),
    ),
  );

return this.maxInFlight - inFlight;
```

**隐性效果**：
- 超过 1 小时的 `processing` 项**不计入在处理数**
- 系统认为有可用容量，会继续取新的 pending 项处理
- 如果某挂起项（有 `processingStartedAt` 但无 `resultBookmarkId`）超过 1 小时：
  - 先被 `getAvailableCapacity` 忽略，不占容量
  - 然后在某次 `resetStaleProcessingItems` 中被重置为 `pending`（取决于系统负载，繁忙时更快触发）
  - 之后可能被重新声明处理

> ⚠️ **代码注释误导**：代码注释写的是 "every 60 iterations ~= 1 min"，但 `pollIntervalMs = 5000`（5秒），实际空闲时是 5 分钟。注释本身与实际参数不符。

> **重要限制**：
> - 已成功创建书签（有 `resultBookmarkId`）的项目**永远不会被重试**
> - 哪怕后续 crawl/tagging 失败了，也只会标记为 `failed`，不会重试
> - 只有"书签创建前"挂起的项目才会被重置重试

### 5. 循环引用与父列表缺失兜底

**位置**: `importer.ts:131-147`

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

### 6. 会话暂停后的状态回滚

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

### 7. 并发控制与原子性

- **原子声明**：UPDATE + WHERE 确保多 Worker 不重复处理同一条
- **背压控制**：`maxInFlight = 50`，通过 `processingStartedAt` 判断是否为有效在处理项
- **公平调度**：按 `importSessions.lastProcessedAt` 排序，避免大导入阻塞其他用户

---

## Session 级别 failed 状态分析

### 核心结论

**`importSessions.status = "failed"` 在当前代码中是不可达状态。**

### 可达状态路径完整清单

搜索所有 `update(importSessions).set({ status: ... })` 调用，仅发现以下状态转换：

| 起始状态 | 目标状态 | 触发位置 | 触发条件 |
|---------|---------|---------|---------|
| `staging` | `pending` | `importSessions.service.ts:131` | `finalize()` 调用 |
| `pending` | `running` | `importWorker.ts:203-210` | 批次处理开始 |
| `pending`/`running` | `paused` | `importSessions.service.ts:142` | 用户调用 `pause()` |
| `paused` | `pending` | `importSessions.service.ts:153` | 用户调用 `resume()` |
| `pending`/`running` | `completed` | `importWorker.ts:514-516` | 所有 staging item 处理完成 |

### 缺失的路径

**没有任何代码会将 session 设置为 `"failed"`**。哪怕：
- 所有书签都处理失败（`result: "rejected"`）
- 部分或全部书签 crawl/tagging 失败
- Worker 崩溃重启

### 实际行为

当导入出现大量失败时：
- 单个书签会被标记为 `status: "failed"`, `result: "rejected"`
- Session 会继续处理剩余书签
- 所有书签处理完成后，Session 状态变为 `"completed"`（不是 `"failed"`）
- 用户通过 stats 可以看到 `failedBookmarks` 计数

### 设计意图推测

`"failed"` 可能是为以下场景预留的，但当前未实现：
- 整个导入任务的系统性失败（如数据库连接中断）
- 超过最大重试次数后的标记
- 批量取消/中止操作

---

## Parser 层 Fail-Fast 与宽松解析的边界

### 三层容错模型

Parser 层的容错策略分为**三个层级**，边界清晰：

| 层级 | 验证对象 | 失败策略 | 影响范围 |
|------|---------|---------|---------|
| 1. Schema 级 | 文件整体结构 | **Fail-Fast** | 整个导入失败 |
| 2. 记录级 | 单条记录格式 | **宽松解析** | 跳过坏行/静默容错 |
| 3. 字段级 | 单个字段转换 | **宽松容错** | 字段降级（如 `tags = []`） |

---

### 各 Parser 的策略矩阵

| Parser | Schema 级 | 记录级 | 字段级 |
|--------|----------|--------|--------|
| Netscape HTML | ✅ Fail-Fast（文件头检查） | ✅ 宽松（逐行解析） | ✅ 宽松（空文件夹名→"Unnamed"） |
| Matter CSV | ✅ Fail-Fast（Zod schema） | ❌ 无（schema 级已保证） | ❌ 无 |
| Karakeep JSON | ✅ Fail-Fast（Zod schema） | ❌ 无（schema 级已保证） | ❌ 无 |
| Omnivore JSON | ✅ Fail-Fast（Zod schema） | ❌ 无（schema 级已保证） | ❌ 无 |
| Linkwarden JSON | ✅ Fail-Fast（Zod schema） | ❌ 无（schema 级已保证） | ❌ 无 |
| Tab Session Manager | ✅ Fail-Fast（Zod schema） | ❌ 无（schema 级已保证） | ❌ 无 |
| mymind CSV | ✅ Fail-Fast（Zod schema） | ❌ 无（schema 级已保证） | ✅ 宽松（URL/Content 二选一） |
| Instapaper CSV | ✅ Fail-Fast（Zod schema） | ❌ 无（schema 级已保证） | ✅ 宽松（URL/Selection 二选一，tags 容错） |
| Readwise Reader CSV | ✅ Fail-Fast（Zod schema） | ✅ 宽松（过滤 feed 和空 URL） | ✅ 宽松（tags 容错） |
| Pocket CSV | ❌ 无 schema | ✅ 宽松（逐行映射，无 URL 也保留） | ✅ 宽松 |
| OneTab TXT | ❌ 无 schema | ✅ 宽松（逐行跳过非 URL） | ✅ 宽松 |

---

### Fail-Fast 边界（Schema 级）

**触发条件**：文件整体结构不符合预期，无法可靠解析。

**9 种格式的 Fail-Fast 点**：

| Parser | 验证方式 | 失败抛出 | 位置 |
|--------|---------|---------|------|
| Netscape HTML | 文件头字符串匹配 | `throw Error("The uploaded html file does not seem to be a bookmark file")` | parsers.ts:54-55 |
| Matter CSV | Zod `safeParse()` + 手动 throw | `throw new Error("The uploaded CSV file contains an invalid Matter bookmark file: ...")` | parsers.ts:164-167 |
| Karakeep JSON | Zod `safeParse()` + 手动 throw | `throw new Error("The uploaded JSON file contains an invalid bookmark file: ...")` | parsers.ts:184-187 |
| Omnivore JSON | Zod `safeParse()` + 手动 throw | `throw new Error("The uploaded JSON file contains an invalid omnivore bookmark file: ...")` | parsers.ts:251-254 |
| Linkwarden JSON | Zod `safeParse()` + 手动 throw | `throw new Error("The uploaded JSON file contains an invalid Linkwarden bookmark file: ...")` | parsers.ts:289-292 |
| Tab Session Manager | Zod `safeParse()` + 手动 throw | `throw new Error("The uploaded JSON file contains an invalid Tab Session Manager bookmark file: ...")` | parsers.ts:342-345 |
| mymind CSV | Zod `safeParse()` + 手动 throw | `throw new Error("The uploaded CSV file contains an invalid mymind bookmark file: ...")` | parsers.ts:384-387 |
| Instapaper CSV | Zod `safeParse()` + 手动 throw | `throw new Error("CSV file contains an invalid instapaper bookmark file: ...")` | parsers.ts:443-446 |
| Readwise Reader CSV | Zod `safeParse()` + 手动 throw | `throw new Error("CSV file contains an invalid Readwise Reader bookmark file: ...")` | parsers.ts:530-534 |

> **注意**：Zod `parse()` 本身就会 throw，使用 `safeParse()` + 手动 throw 是为了提供更友好的错误信息。

**宽松解析边界（2 种格式）**：
- **Pocket CSV**：无 schema 验证，逐行 `map`，行结构不匹配时由 `csv-parse` 处理或产生无效记录
- **OneTab TXT**：无 schema，逐行解析，非 `http://` 或 `https://` 开头的行静默 `continue`

---

### 宽松容错边界（字段级）

即使 schema 验证通过，部分字段转换失败时也会**静默降级**，不影响整条记录：

| Parser | 字段 | 容错策略 | 位置 |
|--------|------|---------|------|
| Readwise Reader | `tags` | `JSON.parse()` 失败 → `tags = []` | parsers.ts:554-564 |
| Instapaper | `tags` | `JSON.parse()` 失败 → `tags = []` | parsers.ts:464-471 |
| Instapaper | `content` | URL 和 Selection 都为空 → `content = undefined` | parsers.ts:452-460 |
| mymind | `content` | URL 和 content 二选一，URL 优先 | parsers.ts:393-410 |
| Readwise Reader | `content` | 空 URL 记录在 parser 内直接过滤排除 | parsers.ts:542-544 |
| Readwise Reader | `Location` | 值为 `"feed"` 时过滤排除 | parsers.ts:539-541 |

> **关键点**：字段级容错**只降级，不抛出**。`content: undefined` 的记录会继续传递到后续流程，由暂存前过滤处理。

---

### 异常传播路径

```
parseImportFile() 抛出异常（仅 Schema 级失败）
    ↓
importBookmarksFromFile() 未捕获，继续向上
    ↓
useBookmarkImport.ts 中 onError 捕获
    ↓
toast 显示错误信息给用户
```

**Schema 级失败的影响**：
- 不会创建 Import Session
- 不会创建任何列表
- 不会暂存任何书签
- 整个导入完全失败，用户需要修正文件后重新上传

**字段级/记录级宽松处理的影响**：
- 无效字段降级（如 `tags = []`）
- 部分记录被过滤（如 Readwise Reader 空 URL）
- 部分记录产生 `content: undefined`（后续由暂存前过滤处理）
- 导入继续进行，不会整体失败

---

## 导入返回 counts 的语义边界

### ImportCounts 接口定义

**位置**: `importer.ts:4-9`

```typescript
export interface ImportCounts {
  successes: number;
  failures: number;
  alreadyExisted: number;
  total: number;
}
```

### 实际返回值（修正前的关键误解）

**位置**: `importer.ts:277-283` 和 `importer.ts:81-85`

```typescript
// 正常返回（有书签）
return {
  counts: {
    successes: 0,        // ⚠️ 永远是 0！
    failures: 0,         // ⚠️ 永远是 0！
    alreadyExisted: 0,   // ⚠️ 永远是 0！
    total: parsedBookmarks.length,  // 唯一有意义的字段
  },
  rootListId: rootList.id,
  importSessionId: session.id,
};

// 空文件返回
return {
  counts: { successes: 0, failures: 0, alreadyExisted: 0, total: 0 },
  rootListId: null,
  importSessionId: null,
};
```

### 语义边界澄清

| 字段 | 含义 | 阶段 |
|------|------|------|
| `total` | **解析到**的书签总数（**过滤前**的数量，包含无 URL 的 link 等会被过滤的项） | 解析阶段 |
| `successes` | 预留字段，当前异步导入模式下**恒为 0** | 未使用 |
| `failures` | 预留字段，当前异步导入模式下**恒为 0** | 未使用 |
| `alreadyExisted` | 预留字段，当前异步导入模式下**恒为 0** | 未使用 |

### 设计意图

`ImportCounts` 接口是为**同步导入模式**设计的，即 `importBookmarksFromFile` 内部完成所有处理并返回最终结果。但当前实现采用的是**异步导入模式**：
- `importBookmarksFromFile` 只负责**解析和暂存**
- 实际处理由后台 Worker 异步完成
- 最终结果需要通过 `importSessions.getWithStats()` 后续查询

### 获取真实统计数据

**位置**: `importSessions.service.ts:170-206`

```typescript
// 通过 session stats 获取真实处理结果
const stats = {
  totalBookmarks: 0,        // 暂存的总数（过滤后）
  completedBookmarks: 0,    // status = "completed" 的数量
  failedBookmarks: 0,       // status = "failed" 的数量
  pendingBookmarks: 0,      // status = "pending" 的数量
  processingBookmarks: 0,   // status = "processing" 的数量
};
```

> **`total` 语义区别**：
> - `importBookmarksFromFile` 返回的 `counts.total` = **解析到**的数量（过滤前）
> - `getWithStats` 返回的 `totalBookmarks` = **实际暂存**的数量（过滤后）
> - 两者可能不相等（存在过滤时）

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
| `"rejected"` | `failed` | 验证失败、创建异常、下游失败、asset 不支持、空 URL、空内容 |

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
- Schema 级 Fail-Fast：文件头不匹配直接 `throw Error`
- 字段级宽松：逐行解析，单条记录问题不影响整体

### 2. Pocket (`pocket`)

**代码位置**: `parsers.ts:113-135`

| CSV 列 | 内部字段 | 说明 |
|-------|---------|------|
| `title` | `title` | 标题 |
| `url` | `content.url` | 链接地址 |
| `time_added` | `addDate` | Unix 时间戳 |
| `tags` | `tags` | 管道符分隔，`split("|")` |
| `status` | `archived` | `status === "archive"` → `true` |

**特殊处理**:
- 无 Schema 级验证，完全宽松解析
- 无 URL 的行也会被解析（`content.url: ""`），但会在暂存前被过滤
- 字段级宽松：逐行映射，不做 schema 校验

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
- 空 URL 项目在 **parser 内就被过滤**（parsers.ts:542-544），不进入后续流程
- 标签 JSON 解析失败时 `tags = []`（字段级宽松，不抛出错误）
- feed 项目在 parser 内过滤（parsers.ts:539-541）
- Schema 级 Fail-Fast：Zod schema 验证不通过直接 throw
- 记录级宽松：过滤无效记录不影响整体

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

**特殊处理**:
- URL 和 Selection 都为空时产生 `content: undefined`（字段级宽松），后续由暂存前过滤处理
- 标签 JSON 解析失败时 `tags = []`（字段级宽松，不抛出错误）
- Schema 级 Fail-Fast：Zod schema 验证不通过直接 throw

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
- Schema 级 Fail-Fast：Zod schema 验证不通过直接 throw
- 字段级宽松：无效 content 会在后续流程中被过滤

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
const parsedImport = parseImportFile(source, textContent);  // ⚠️ fail-fast
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

// 4. result.counts 只有 total 有意义，successes/failures/alreadyExisted 恒为 0
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
- ⚠️ 暂存前会过滤无效书签（link 无 URL / text 无内容）

### 阶段三：后台 Worker 处理 (`importWorker.ts`)

#### 轮询循环

```typescript
while (this.running) {
  // 1. 每 60 次轮询重置挂起项（间隔不固定，取决于系统负载）
  // ⚠️ 代码注释写的是 "~= 1 min"，但 pollIntervalMs = 5000，实际空闲时是 5 分钟
  if (iterationCount % 60 === 0) {
    await this.resetStaleProcessingItems();
  }
  iterationCount++;
  
  // 2. 每次轮询先检查已完成下游处理的项目
  await this.checkAndCompleteProcessingItems();
  
  // 3. 处理一个批次
  const processed = await this.processBatch();
  
  if (processed === 0) {
    // 4. 无任务时检查空闲会话并完成，然后 sleep 5 秒
    await this.checkAndCompleteIdleSessions();
    await this.updateGauges();
    await sleep(this.pollIntervalMs); // 5秒
  } else {
    // 5. 有任务处理完不 sleep，立即开始下一轮
    // 繁忙时 iterationCount 快速递增，60次循环可能只需几秒
    await this.updateGauges();
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

// 2. 验证类型并构建请求 ⚠️ 二次校验
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
| 导入计数接口 | `importer.ts` | `ImportCounts` | 4-9 |
| 导出功能 | `packages/shared/import-export/exporters.ts` | `toExportFormat`, `toNetscapeFormat` | 42-127 |
| 前端导入 Hook | `apps/web/lib/hooks/useBookmarkImport.ts` | `useBookmarkImport` | 23-132 |
| 导入会话服务 | `packages/trpc/models/importSessions.service.ts` | `ImportSessionsService` | 17-206 |
| 暂存前过滤 | `importSessions.service.ts` | `stageBookmarks` | 95-100 |
| 导入会话仓储 | `packages/trpc/models/importSessions.repo.ts` | `ImportSessionsRepo` | 14-124 |
| 后台导入 Worker | `apps/workers/workers/importWorker.ts` | `ImportWorker` | 102-718 |
| 安全错误信息 | `importWorker.ts` | `getSafeErrorMessage` | 82-100 |
| 挂起项重置 | `importWorker.ts` | `resetStaleProcessingItems` | 682-718 |
| 背压控制 | `importWorker.ts` | `getAvailableCapacity` | 655-673 |
| 下游完成检查 | `importWorker.ts` | `checkAndCompleteProcessingItems` | 546-650 |
| 单条处理 | `importWorker.ts` | `processOneBookmark` | 351-487 |
| 二次校验空 URL | `importWorker.ts` | `processOneBookmark` | 392-404 |
| 批次处理 | `importWorker.ts` | `processBatch` | 149-233 |
| 轮询循环 | `importWorker.ts` | `start()` | 111-142 |
| 错误信息白名单 | `importWorker.ts` | `getSafeErrorMessage` | 89-93 |
| RW parser 过滤空 URL | `parsers.ts` | `parseReadwiseReaderBookmarkFile` | 542-544 |
| Instapaper 标签容错 | `parsers.ts` | `parseInstapaperBookmarkFile` | 464-471 |
| RW 标签容错 | `parsers.ts` | `parseReadwiseReaderBookmarkFile` | 554-564 |
| 数据库 Schema | `packages/db/schema.ts` | `importSessions`, `importStagingBookmarks` | 851-950 |
| 导入类型定义 | `packages/shared/types/importSessions.ts` | - | 1-78 |
| Session 状态枚举 | `importSessions.ts` | `zImportSessionStatusSchema` | 3-10 |

---

## 常见误解澄清（三次修正版）

| 误解 | 事实 |
|------|------|
| 结果类型有 `"unsupported"` | 数据库 `result` 枚举只有 `accepted`/`rejected`/`skipped_duplicate`，`"unsupported"` 只是函数返回值，实际存的是 `rejected` |
| 成功创建书签后状态变为 `completed` | 书签创建成功后状态仍为 `processing`，需等 crawl/tagging 完成后才由 `checkAndCompleteProcessingItems` 标记为 `completed` |
| 处理失败后会自动重试 | 只有 `resultBookmarkId IS NULL` 的挂起项会被重置，已创建书签的项目（哪怕下游失败）不会重试 |
| `listExternalIds` 和 `paths` 是二选一 | `listExternalIds` 优先级高于 `paths`，但 `attachBookmarkToLists` 还会额外加上 `session.rootListId` |
| 去重只在导入时做一次 | 两次去重：解析阶段同一文件内去重 + `createBookmark` 跨导入去重 |
| `checkAndCompleteProcessingItems` 在批次后调用 | **每次轮询最先调用**，在 `processBatch` 之前 |
| 挂起项每 5 分钟检查一次 | **不固定**，取决于系统负载：空闲时 60 次 × 5秒 = 5分钟；繁忙时不 sleep，60 次循环可能只需几秒 |
| 代码注释 "every 60 iterations ~= 1 min" 是正确的 | 注释错误，`pollIntervalMs = 5000`（5秒），实际空闲时是 5 分钟，不是 1 分钟 |
| 空 URL 在暂存前过滤了就不会再处理 | Worker 中还有二次校验，若绕过暂存过滤（直接调用 repo 层）会产生可见失败记录 |
| Session 可以达到 `failed` 状态 | **当前代码中不可达**，schema 定义了但没有代码路径设置它，所有 session 最终都是 `completed` |
| Parser 会跳过坏记录继续解析 | Schema 级是 fail-fast 的，格式错误直接抛出；但字段级是宽松的，单字段转换失败只降级不抛出 |
| `counts.successes` 是成功导入的数量 | 异步模式下**恒为 0**，是为同步模式预留的字段 |
| `counts.total` 是实际暂存的数量 | 是**解析到**的数量（过滤前），实际暂存数需通过 `getWithStats` 查询 |
| 空 URL 会静默丢弃 | 取决于入口：暂存前过滤静默丢弃；直接 repo 层绕过则产生可见失败记录 |
| 所有 Parser 都是 100% fail-fast | 三层模型：Schema 级 fail-fast，记录级宽松（过滤/跳过），字段级宽松（降级） |
| Readwise Reader 空 URL 会进入暂存 | 在 **parser 内就被过滤**（parsers.ts:542-544），不会传递到后续流程 |
| Instapaper URL 和 Selection 都为空会导入失败 | 会产生 `content: undefined`，但会被暂存前过滤静默丢弃，不会整体失败 |
| 标签解析失败会导致整条记录失败 | Instapaper 和 Readwise Reader 的标签 JSON 解析失败只会导致 `tags = []`，不影响整条记录 |
