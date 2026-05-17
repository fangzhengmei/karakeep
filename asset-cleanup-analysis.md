# Karakeep 资产清理维护任务分析

## 一、系统架构总览

Karakeep 的资产清理系统采用 **异步队列驱动** 的架构设计，核心组件包括：

```
┌─────────────────┐     ┌─────────────────────┐     ┌──────────────────┐
│  触发层         │────▶│  AdminMaintenance   │────▶│  任务执行层      │
│  - Web Admin    │     │  Queue              │     │  - tidyAssets    │
│  - Admin API    │     │  (numRetries: 1)    │     │  - Worker        │
│  - CLI          │     └─────────────────────┘     └──────────────────┘
└─────────────────┘                                  │
                                                     ▼
                                             ┌──────────────────┐
                                             │  资产存储层      │
                                             │  - Local FS      │
                                             │  - S3            │
                                             └──────────────────┘
```

**关键文件定位：**

| 层级 | 文件路径 | 核心职责 |
|------|---------|---------|
| 任务定义 | `packages/shared-server/src/queues.ts:152-187` | 队列配置与任务类型定义 |
| 任务执行 | `apps/workers/workers/adminMaintenance/tasks/tidyAssets.ts` | 资产清理核心逻辑 |
| Worker管理 | `apps/workers/workers/adminMaintenanceWorker.ts` | Worker 生命周期与错误处理 |
| 触发入口 | `packages/trpc/routers/admin.ts:321-325` | Admin API 触发入口 |
| 前端触发 | `apps/web/components/admin/BackgroundJobs.tsx:454-466` | Web 管理后台触发按钮 |
| 资产存储 | `packages/shared/assetdb.ts` | 本地文件系统/S3 存储抽象 |

---

## 二、清理触发机制

### 2.1 触发方式

资产清理任务 **目前仅支持手动触发**，无自动定时调度。有三种触发途径：

#### 方式1：Web 管理后台触发
位置：`apps/web/components/admin/BackgroundJobs.tsx:454-466`

```typescript
// Admin Maintenance 卡片中的 "Clean Assets" 按钮
{
  label: t("admin.background_jobs.actions.clean_assets"),
  onClick: () =>
    runAdminMaintenanceTask({
      type: "tidy_assets",
      args: {
        cleanDanglingAssets: true,
        syncAssetMetadata: true,
      },
    }),
  loading: isAdminMaintenancePending,
}
```

#### 方式2：Admin API 触发
位置：`packages/trpc/routers/admin.ts:321-325`

```typescript
runAdminMaintenanceTask: adminJobsProcedure
  .input(zAdminMaintenanceTaskSchema)
  .mutation(async ({ input }) => {
    await AdminMaintenanceQueue.enqueue(input);
  }),
```

#### 方式3：CLI 触发（间接）
CLI 目前没有直接的资产清理命令，但可以通过扩展 `apps/cli/src/commands/admin.ts` 中的 `jobsCmd` 来添加。

### 2.2 任务入队流程

位置：`packages/shared-server/src/queues.ts:152-187`

```typescript
// 任务请求 Schema
export const zTidyAssetsRequestSchema = z.object({
  cleanDanglingAssets: z.boolean().optional().default(false),
  syncAssetMetadata: z.boolean().optional().default(false),
});

// 任务类型定义
export const zAdminMaintenanceTaskSchema = z.discriminatedUnion("type", [
  z.object({
    type: z.literal("tidy_assets"),
    args: zTidyAssetsRequestSchema,
  }),
  // ... 其他任务类型
]);

// 队列配置
export const AdminMaintenanceQueue = createDeferredQueue<ZAdminMaintenanceTask>(
  "admin_maintenance_queue",
  {
    defaultJobArgs: {
      numRetries: 1,  // 失败重试1次
    },
    keepFailedJobs: false,  // 不保留失败任务
  },
);
```

### 2.3 完整时序图

```
  用户触发
     │
     ▼
  前端按钮点击 (BackgroundJobs.tsx:454-466)
     │
     ▼
  tRPC mutation: admin.runAdminMaintenanceTask
     │
     ▼
  AdminMaintenanceQueue.enqueue({
    type: "tidy_assets",
    args: { cleanDanglingAssets: true, syncAssetMetadata: true }
  })
     │
     ▼
  任务进入队列，状态: pending
     │
     ▼
  AdminMaintenanceWorker 轮询 (pollIntervalMs: 1000)
     │
     ▼
  取出任务，创建 DequeuedJob
     │
     ▼
  调用 runAdminMaintenance(job)
     │
     ├─▶ 参数校验: zAdminMaintenanceTaskSchema.safeParse
     │
     ├─▶ task.type === "tidy_assets"
     │
     ▼
  调用 runTidyAssetsTask(job, task)
     │
     ├─▶ 参数解析: zTidyAssetsRequestSchema.safeParse(task.args)
     │
     ├─▶ 遍历所有资产: for await (const asset of getAllAssets())
     │    │
     │    ├─▶ 检查 abortSignal.aborted
     │    │
     │    └─▶ 调用 handleAsset(asset, request, jobId)
     │         │
     │         ├─▶ 数据库查询: db.query.assets.findFirst()
     │         │
     │         ├─▶ 分支1: !dbRow && cleanDanglingAssets → deleteAsset()
     │         │
     │         ├─▶ 分支2: dbRow && syncAssetMetadata → db.update()
     │         │
     │         └─▶ 异常捕获: try-catch 记录错误日志
     │
     ▼
  任务完成 → onComplete 回调
     │
     ▼
  指标统计 + 日志记录
```

---

## 三、引用判断逻辑

### 3.1 资产数据模型

位置：`packages/db/schema.ts:295-325`

```typescript
export const assets = sqliteTable(
  "assets",
  {
    id: text("id").notNull().primaryKey(),
    assetType: text("assetType", { enum: [/* 13种资产类型 */] }).notNull(),
    size: integer("size").notNull().default(0),
    contentType: text("contentType"),
    fileName: text("fileName"),
    bookmarkId: text("bookmarkId").references(() => bookmarks.id, {
      onDelete: "cascade",  // 书签删除时级联删除资产记录
    }),
    userId: text("userId")
      .notNull()
      .references(() => users.id, { onDelete: "cascade" }),
  },
);
```

### 3.2 引用判断算法

位置：`apps/workers/workers/adminMaintenance/tasks/tidyAssets.ts:14-55`

**核心逻辑：**

```
对于存储系统中的每个资产：
  1. 通过 assetId 查询数据库 assets 表
  2. 如果数据库中不存在记录 → 判定为"悬空资产"(dangling asset)
  3. 如果 cleanDanglingAssets = true → 从存储中删除该资产
  4. 如果 syncAssetMetadata = true → 用存储中的元数据更新数据库记录
```

**代码实现：**

```typescript
async function handleAsset(asset: AssetInfo, request: ZTidyAssetsRequest, jobId: string) {
  const dbRow = await db.query.assets.findFirst({
    where: eq(assets.id, asset.assetId),
  });
  
  if (!dbRow) {
    if (request.cleanDanglingAssets) {
      await deleteAsset({ userId: asset.userId, assetId: asset.assetId });
      logger.info(`[adminMaintenance:tidy_assets][${jobId}] Asset ${asset.assetId} not found in DB. Deleting.`);
    } else {
      logger.warn(`[adminMaintenance:tidy_assets][${jobId}] Asset ${asset.assetId} not found in DB. Skipping.`);
    }
    return;
  }

  if (request.syncAssetMetadata) {
    await db.update(assets).set({
      contentType: asset.contentType,
      fileName: asset.fileName,
      size: asset.size,
    }).where(eq(assets.id, asset.assetId));
    logger.info(`[adminMaintenance:tidy_assets][${jobId}] Updated metadata for asset ${asset.assetId}`);
  }
}
```

### 3.3 四种参数组合行为详解

| 组合 | cleanDanglingAssets | syncAssetMetadata | 悬空资产行为 | 存在资产行为 | 适用场景 |
|------|---------------------|-------------------|------------|------------|----------|
| **组合1** | `false` | `false` | 只打 WARN 日志，不删除 | 不做任何修改 | **干跑验证**：只检查不修改，用于首次执行前确认 |
| **组合2** | `false` | `true` | 只打 WARN 日志，不删除 | 用存储元数据更新数据库 | **元数据修复**：数据库元数据损坏时从存储恢复 |
| **组合3** | `true` | `false` | 从存储中删除 | 不做任何修改 | **清理悬空**：只清理无效资产，不影响有效资产 |
| **组合4** | `true` | `true` | 从存储中删除 | 用存储元数据更新数据库 | **完全维护**：清理 + 元数据同步，推荐生产使用 |

> **重要提示**：Web 管理后台默认使用 **组合4** (`cleanDanglingAssets: true, syncAssetMetadata: true`)

### 3.4 assetId 唯一性前提与跨用户安全性

#### 3.4.1 assetId 生成机制

位置：`packages/shared/assetdb.ts:130-132`

```typescript
export function newAssetId() {
  return crypto.randomUUID();
}
```

**关键特性**：
- 使用标准 `crypto.randomUUID()` 生成 RFC 4122 版本 4 UUID
- 基于加密安全的随机数生成器 (CSPRNG)
- 理论碰撞概率极低：每年生成 1 万亿个 UUID，碰撞概率约为 1/500亿
- 全局唯一性，不依赖任何用户上下文

#### 3.4.2 跨用户误判安全性论证

**查询逻辑**（`tidyAssets.ts:25-27`）：
```typescript
const dbRow = await db.query.assets.findFirst({
  where: eq(assets.id, asset.assetId),
});
```

**安全性论证**：

1. **主键唯一性约束**：`assets.id` 是数据库主键，全局唯一
2. **UUID 全局唯一**：`assetId` 由 `randomUUID()` 生成，确保跨用户不重复
3. **无需 userId 过滤**：由于 `assetId` 全局唯一，仅按 `assetId` 查询即可准确定位资产，不会出现用户A的资产ID匹配到用户B的资产记录的情况
4. **删除安全**：即使存储中存在两个不同用户的同名资产（实际不会发生），数据库查询也只会匹配到正确的记录

**结论**：✅ **仅按 assetId 查库不会产生跨用户误判**，系统设计是安全的。

> **补充说明**：虽然查询时不需要 userId，但删除操作 `deleteAsset({ userId, assetId })` 仍然传入了 userId，这是因为存储层按 `userId/assetId` 路径组织文件，传入 userId 是为了定位存储路径，而非安全检查。

### 3.5 资产存储遍历

位置：`packages/shared/assetdb.ts:323-343` (LocalFileSystem) 和 `packages/shared/assetdb.ts:566-612` (S3)

**本地文件系统遍历：**
```typescript
async *getAllAssets() {
  const g = new Glob(`/**/**/asset.bin`, { maxDepth: 3, root: this.rootPath, ... });
  for await (const file of g) {
    const [userId, assetId] = file.split("/").slice(0, 2);
    const [size, metadata] = await Promise.all([
      this.getAssetSize({ userId, assetId }),
      this.readAssetMetadata({ userId, assetId }),
    ]);
    yield { userId, assetId, ...metadata, size };
  }
}
```

**S3 遍历：**
```typescript
async *getAllAssets() {
  let continuationToken: string | undefined;
  do {
    const listResponse = await this.s3Client.send(new ListObjectsV2Command({ ... }));
    if (listResponse.Contents) {
      for (const obj of listResponse.Contents) {
        const pathParts = obj.Key.split("/");
        if (pathParts.length === 2) {
          const userId = pathParts[0];
          const assetId = pathParts[1];
          // 读取元数据并 yield
        }
      }
    }
    continuationToken = listResponse.NextContinuationToken;
  } while (continuationToken);
}
```

---

## 四、删除节奏控制

### 4.1 任务参数配置

清理任务支持两个独立的控制开关：

| 参数 | 类型 | 默认值 | 作用 |
|------|------|--------|------|
| `cleanDanglingAssets` | boolean | `false` | 是否删除数据库中不存在的悬空资产 |
| `syncAssetMetadata` | boolean | `false` | 是否用存储中的元数据同步更新数据库 |

### 4.2 执行节奏

位置：`apps/workers/workers/adminMaintenance/tasks/tidyAssets.ts:57-82`

**执行特点：**

1. **串行遍历**：使用 `for await (const asset of getAllAssets())` 逐个处理
2. **可中断**：每次循环检查 `job.abortSignal.aborted`，支持任务中止
3. **容错隔离**：单个资产处理失败用 `try-catch` 包裹，不影响整体流程
4. **无节流**：资产之间没有延迟，连续处理（适合后台低峰期执行）

```typescript
export async function runTidyAssetsTask(job: DequeuedJob<ZAdminMaintenanceTidyAssetsTask>, task: ZAdminMaintenanceTidyAssetsTask) {
  const jobId = job.id;
  const parseResult = zTidyAssetsRequestSchema.safeParse(task.args);
  if (!parseResult.success) {
    throw new Error(`[adminMaintenance:tidy_assets][${jobId}] Got malformed args: ${parseResult.error.toString()}`);
  }

  for await (const asset of getAllAssets()) {
    if (job.abortSignal.aborted) {
      logger.warn(`[adminMaintenance:tidy_assets][${jobId}] Aborted`);
      break;
    }
    try {
      await handleAsset(asset, parseResult.data, jobId);
    } catch (error) {
      logger.error(`[adminMaintenance:tidy_assets][${jobId}] Failed to tidy asset ${asset.assetId}: ${error}`);
    }
  }
}
```

### 4.3 Worker 配置

位置：`apps/workers/workers/adminMaintenanceWorker.ts:17-63`

```typescript
export class AdminMaintenanceWorker {
  static async build() {
    const worker = (await getQueueClient())!.createRunner<ZAdminMaintenanceTask>(
      AdminMaintenanceQueue,
      {
        run: withWorkerTracing("adminMaintenanceWorker.run", runAdminMaintenance),
        onComplete: (job) => { /* 成功回调 */ },
        onError: (job) => { /* 错误回调 */ },
      },
      {
        concurrency: 1,        // 单并发执行
        pollIntervalMs: 1000,  // 每秒轮询队列
        timeoutSecs: 600,      // 任务超时10分钟
      },
    );
    return worker;
  }
}
```

**关键配置说明：**

- `concurrency: 1`：确保同一时间只有一个维护任务在执行，避免资源竞争
- `timeoutSecs: 600`：10分钟超时，防止任务无限期挂起
- `pollIntervalMs: 1000`：每秒检查队列中是否有新任务

---

## 五、异常恢复机制

### 5.1 多层级错误处理

Karakeep 采用 **四层错误处理** 架构，确保系统稳定性：

```
┌─────────────────────────────────────────────────┐
│  1. 队列层 (Queue Level)                        │
│  - numRetries: 1  (失败重试1次)                 │
│  - keepFailedJobs: false (失败后不保留)         │
├─────────────────────────────────────────────────┤
│  2. Worker层 (Worker Level)                     │
│  - onError 回调记录错误日志和指标               │
│  - 区分临时失败 / 永久失败                      │
├─────────────────────────────────────────────────┤
│  3. 任务层 (Task Level)                         │
│  - 单个资产处理 try-catch 隔离                  │
│  - 失败后继续下一个资产                         │
├─────────────────────────────────────────────────┤
│  4. 操作层 (Operation Level)                    │
│  - deleteAsset 静默失败 (catch 吞掉异常)        │
│  - getAllAssets 单个资产读取失败不中断遍历      │
└─────────────────────────────────────────────────┘
```

### 5.2 关键异常场景分析

#### 场景1：数据库查询失败（连接超时、网络中断等）

**代码位置**：`tidyAssets.ts:25-27`

```typescript
const dbRow = await db.query.assets.findFirst({
  where: eq(assets.id, asset.assetId),
});
```

**行为分析**：
- 如果 `findFirst()` 抛出异常（如数据库连接失败），异常会从 `handleAsset` 抛出
- 被外层 `runTidyAssetsTask` 中的 `try-catch` 捕获
- 记录错误日志：`Failed to tidy asset ${asset.assetId}: ${error}`
- **不会执行删除操作**，因为代码在 `if (!dbRow)` 判断之前就抛出了异常
- 继续处理下一个资产

**结论**：✅ **查询失败不会触发误删**，系统设计是安全的。

#### 场景2：存储删除失败（文件被占用、S3 权限问题等）

**代码位置**：`tidyAssets.ts:30`

```typescript
await deleteAsset({ userId: asset.userId, assetId: asset.assetId });
```

**行为分析**：
- `deleteAsset` 本身没有 `try-catch`，会向上抛出异常
- 被外层 `try-catch` 捕获，记录错误日志
- 该资产的删除操作失败，但不影响其他资产处理
- 下次执行清理任务时会再次尝试删除

#### 场景3：数据库更新失败（锁冲突、事务失败等）

**代码位置**：`tidyAssets.ts:43-50`

```typescript
await db.update(assets).set({ ... }).where(eq(assets.id, asset.assetId));
```

**行为分析**：
- 更新操作抛出异常，被外层 `try-catch` 捕获
- 记录错误日志，继续处理下一个资产
- 元数据未更新，下次执行时会再次尝试

#### 场景4：整任务失败触发重试（中段失败）

**触发条件**：
- 任务级别的异常（如 `getAllAssets()` 遍历失败、参数解析失败）
- 未被任务层 `try-catch` 捕获的异常（如数据库连接池耗尽）
- 触发队列层重试机制，整个任务从头开始重跑

**重试配置**：`numRetries: 1`

**重试生命周期**：
```
第1次执行 (runNumber=0, numRetriesLeft=1)
    │
    ├─ 处理资产 1~N
    │    ├─ 资产 1~500 处理成功
    │    └─ 资产 501 处理时抛出未捕获异常
    │
    └─ 任务失败 → onError 回调
         │
         ├─ workerStatsCounter: failed +1
         └─ 日志: Job failed: ...
             
延迟后重试（延迟时间取决于错误类型：
    │
    ▼
第2次执行 (runNumber=1, numRetriesLeft=0)
    │
    ├─ 重新遍历所有资产（从资产 1 开始）
    │    ├─ 资产 1~500 重复处理
    │    ├─ 资产 501 再次尝试
    │    └─ ...
    │
    └─ 如果成功 → onComplete 回调
              │
              └─ workerStatsCounter: completed +1
              
       如果再次失败 → onError 回调
              │
              ├─ workerStatsCounter: failed +1
              ├─ workerStatsCounter: failed_permanent +1
              └─ 任务永久失败
```

### 5.3 重试语义深度分析

#### 5.3.1 重试机制详解

位置：`packages/plugins/queue-restate/src/dispatcher.ts:92-207`

**重试循环**：
```typescript
let runNumber = 0;
while (runNumber <= NUM_RETRIES) {  // NUM_RETRIES=1 时循环执行2次
  // 执行任务
  const res = await tryCatch(runner.run(jobData));
  
  if (result.type === "error") {
    await ctx.sleep(1000, "error retry");  // 固定1秒延迟
    runNumber++;
    continue;
  }
  break;
}
```

**关键参数传递**：
- `runNumber`：当前执行次数（0 = 首次，1 = 第1次重试）
- `numRetriesLeft`：剩余重试次数（NUM_RETRIES - runNumber）
- 每次重试都会传递给 Worker，可通过 `job.runNumber` 和 `job.numRetriesLeft` 访问

#### 5.3.2 删除分支（cleanDanglingAssets=true）的重试幂等性

**代码路径**：`tidyAssets.ts:28-33`

```typescript
if (!dbRow) {
  if (request.cleanDanglingAssets) {
    await deleteAsset({ userId: asset.userId, assetId: asset.assetId });
    logger.info(`[adminMaintenance:tidy_assets][${jobId}] Asset ${asset.assetId} not found in DB. Deleting.`);
  }
}
```

**存储层删除实现**：`assetdb.ts:306-312`
```typescript
async deleteAsset({ userId, assetId }) {
  const assetDir = this.getAssetDir(userId, assetId);
  if (!(await this.isPathExists(assetDir))) {
    return;  // 不存在则直接返回，不报错
  }
  await fs.promises.rm(assetDir, { recursive: true });
}
```

**中段失败后的重跑行为**：

| 阶段 | 资产状态 | 操作结果 | 可观测信号 |
|------|---------|---------|-----------|
| 第1次执行 | 存储存在，DB不存在 | 从存储删除成功<br>打 INFO 日志 | `Asset xxx not found in DB. Deleting.` |
| 第2次重试 | 存储已删除，DB不存在 | `deleteAsset` 检测到不存在，静默返回<br>仍然打 INFO 日志 | **重复日志**：`Asset xxx not found in DB. Deleting.` |

**幂等性结论**：✅ **删除操作完全幂等**

**潜在副作用**：
- 重复的 INFO 日志输出，可能干扰日志分析
- 重复执行 `isPathExists` 检查，产生轻微 I/O 开销
- S3 模式下产生额外的 HEAD 请求费用

#### 5.3.3 元数据同步分支（syncAssetMetadata=true）的重试幂等性

**代码路径**：`tidyAssets.ts:42-50`

```typescript
if (dbRow && request.syncAssetMetadata) {
  await db.update(assets).set({
    contentType: asset.contentType,
    fileName: asset.fileName,
    size: asset.size,
  }).where(eq(assets.id, asset.assetId));
  logger.info(`[adminMaintenance:tidy_assets][${jobId}] Updated metadata for asset ${asset.assetId}`);
}
```

**中段失败后的重跑行为**：

| 阶段 | 状态 | 操作结果 | 可观测信号 |
|------|------|---------|-----------|
| 第1次执行 | DB元数据与存储不一致 | UPDATE 成功<br>打 INFO 日志 | `Updated metadata for asset xxx` |
| 第2次重试 | DB元数据已与存储一致 | 执行相同的 UPDATE 操作<br>数据库无实际变化（值相同）<br>仍然打 INFO 日志 | **重复日志**：`Updated metadata for asset xxx` |

**幂等性结论**：✅ **元数据同步最终一致幂等**

**潜在副作用**：
- 重复的 UPDATE 语句执行，产生写入放大
- 重复的 INFO 日志输出
- 数据库写入负载增加（虽然值无变化，但仍需执行 SQL）
- 可能触发数据库的 WAL 日志写入

#### 5.3.4 重试的可观测信号

**日志信号**：

| 事件 | 日志内容 | 级别 |
|------|---------|------|
| 任务开始 | `[adminMaintenance:tidy_assets][${jobId}] Starting...` | INFO |
| 删除悬空资产 | `[adminMaintenance:tidy_assets][${jobId}] Asset ${assetId} not found in DB. Deleting.` | INFO |
| 跳过悬空资产 | `[adminMaintenance:tidy_assets][${jobId}] Asset ${assetId} not found in DB. Skipping.` | WARN |
| 更新元数据 | `[adminMaintenance:tidy_assets][${jobId}] Updated metadata for asset ${assetId}` | INFO |
| 单个资产失败 | `[adminMaintenance:tidy_assets][${jobId}] Failed to tidy asset ${assetId}: ${error}` | ERROR |
| 任务完成 | `[adminMaintenance:tidy_assets][${jobId}] Completed successfully` | INFO |
| 任务失败 | `[adminMaintenance:tidy_assets][${jobId}] Job failed: ${error}` | ERROR |

**指标信号**（Prometheus 格式）：

```
# HELP worker_stats_counter Worker task completion counter
# TYPE worker_stats_counter counter
worker_stats_counter{task="adminMaintenance:tidy_assets",status="completed"} 1
worker_stats_counter{task="adminMaintenance:tidy_assets",status="failed"} 1
worker_stats_counter{task="adminMaintenance:tidy_assets",status="failed_permanent"} 0
```

**重试识别方法**：
1. 相同 `job.id` 出现多次 `Starting...` 日志
2. 相同 assetId 出现多次 `Deleting` 或 `Updated metadata` 日志
3. `runNumber` 字段可从日志上下文推断（首次=0，重试=1）

#### 5.3.5 重试副作用总结

| 维度 | 影响程度 | 说明 |
|------|---------|------|
| 数据正确性 | ✅ 无影响 | 操作幂等，最终结果一致 |
| 日志清晰度 | ⚠️ 中等 | 重复日志可能干扰问题排查 |
| 存储 I/O | ⚠️ 低 | 重复的路径存在性检查 |
| 数据库负载 | ⚠️ 低-中 | 重复的 UPDATE 语句 |
| S3 API 成本 | ⚠️ 低 | 重复的 LIST/HEAD 请求 |
| 任务总耗时 | ⚠️ 高 | 完整重跑，已处理部分重复执行 |

### 5.4 队列层重试

位置：`packages/shared/queueing.ts:10`

```typescript
// 特殊错误类型：支持延迟重试且不计入重试次数
export class QueueRetryAfterError extends Error {
  constructor(message: string, public readonly delayMs: number) {
    super(message);
    this.name = "QueueRetryAfterError";
  }
}
```

### 5.4 Worker 层错误处理

位置：`apps/workers/workers/adminMaintenanceWorker.ts:37-53`

```typescript
onError: (job) => {
  workerStatsCounter.labels(`adminMaintenance:${job.data?.type}`, "failed").inc();
  if (job.numRetriesLeft == 0) {
    workerStatsCounter.labels(`adminMaintenance:${job.data?.type}`, "failed_permanent").inc();
  }
  logger.error(`[adminMaintenance:${job.data?.type}][${job.id}] Job failed: ${job.error}\n${job.error.stack}`);
  return Promise.resolve();
},
```

### 5.5 静默删除机制

位置：`packages/shared/assetdb.ts:794-801`

```typescript
/**
 * Deletes the passed in asset if it exists and ignores any errors
 */
export async function silentDeleteAsset(userId: string, assetId: string | undefined) {
  if (assetId) {
    await deleteAsset({ userId, assetId }).catch(() => ({}));
  }
}
```

该机制被用在资产替换和分离场景：

- `packages/trpc/models/assets.ts:161-164` (replaceAsset)
- `packages/trpc/models/assets.ts:199-201` (detachAsset)

### 5.6 任务中止支持

位置：`apps/workers/workers/adminMaintenance/tasks/tidyAssets.ts:70-73`

```typescript
if (job.abortSignal.aborted) {
  logger.warn(`[adminMaintenance:tidy_assets][${jobId}] Aborted`);
  break;
}
```

清理任务在每次循环处理资产前都会检查中止信号，支持优雅中断。

---

## 六、设计优缺点分析

### 6.1 优点

1. **存储抽象良好**：`AssetStore` 接口完美隔离了本地文件系统和 S3，切换存储后端无需修改清理逻辑
2. **容错性强**：四层错误处理确保单个资产问题不会导致整个清理任务失败
3. **可观测性完善**：每个关键操作都有日志记录和指标统计
4. **参数化设计**：`cleanDanglingAssets` 和 `syncAssetMetadata` 独立控制，支持"干跑"模式
5. **级联删除保护**：数据库层面通过 `onDelete: cascade` 确保书签/用户删除时资产记录被清理
6. **查询安全**：数据库查询失败不会触发误删，异常会被安全捕获

### 6.2 潜在改进点

1. **缺少自动调度**：目前只能手动触发，建议添加类似 `BackupSchedulingWorker` 的 cron 定时调度
2. **缺少批量删除**：S3 存储支持 `DeleteObjectsCommand` 批量删除，但当前实现是逐个删除
3. **无进度跟踪**：长时运行的清理任务无法报告进度百分比
4. **无幂等性保证**：任务中断后重新执行会从头开始遍历，浪费资源
5. **无清理前统计**：执行前无法预览将删除多少资产、释放多少空间
6. **无批量查询优化**：当前是每个资产单独查询数据库，可优化为批量查询

### 6.3 风险提示

1. **超时风险**：10分钟超时对于百万级资产可能不足，需要根据实际数据量调整
2. **性能影响**：大量资产时，`getAllAssets()` 会遍历所有对象，可能对存储系统造成压力
3. **S3 成本**：S3 模式下 `getAllAssets()` 会产生 LIST 请求费用，大量资产时需注意成本
4. **幂等性问题**：如果任务执行到一半失败，重新执行会重复处理已处理过的资产

---

## 七、操作指南

### 7.1 触发清理任务

**通过 Web 管理后台：**
1. 登录管理员账号
2. 进入 Admin → Background Jobs
3. 找到 "Admin Maintenance" 卡片
4. 点击 "Clean Assets" 按钮（默认使用组合4）
5. 确认操作

**通过 API 调用：**
```typescript
// 使用 tRPC 客户端 - 完全维护模式（推荐）
await client.admin.runAdminMaintenanceTask.mutate({
  type: "tidy_assets",
  args: {
    cleanDanglingAssets: true,   // 执行删除
    syncAssetMetadata: true,     // 同步元数据
  },
});

// 干跑模式（首次执行推荐）
await client.admin.runAdminMaintenanceTask.mutate({
  type: "tidy_assets",
  args: {
    cleanDanglingAssets: false,  // 不删除，只检查
    syncAssetMetadata: false,    // 不同步
  },
});
```

### 7.2 安全执行建议

1. **先干跑验证**：第一次执行时使用 `cleanDanglingAssets: false, syncAssetMetadata: false`，通过日志观察哪些资产会被判定为悬空
2. **低峰期执行**：大量资产时建议在业务低峰期执行
3. **监控日志**：执行期间监控 `[adminMaintenance:tidy_assets]` 前缀的日志
4. **备份数据**：执行前建议备份存储系统数据
5. **关注错误**：注意观察 `Failed to tidy asset` 错误，排查是否有系统性问题

---

## 八、相关代码索引

| 功能模块 | 文件路径 | 行号 |
|---------|---------|------|
| 任务类型定义 | `packages/shared-server/src/queues.ts` | 152-187 |
| 清理任务核心 | `apps/workers/workers/adminMaintenance/tasks/tidyAssets.ts` | 1-82 |
| Worker 定义 | `apps/workers/workers/adminMaintenanceWorker.ts` | 1-94 |
| 资产存储接口 | `packages/shared/assetdb.ts` | 87-128 |
| 本地存储实现 | `packages/shared/assetdb.ts` | 134-344 |
| S3 存储实现 | `packages/shared/assetdb.ts` | 346-621 |
| assetId 生成 | `packages/shared/assetdb.ts` | 130-132 |
| 数据库 Schema | `packages/db/schema.ts` | 295-325 |
| Admin API | `packages/trpc/routers/admin.ts` | 321-325 |
| 前端触发按钮 | `apps/web/components/admin/BackgroundJobs.tsx` | 454-466 |
| Workers 入口 | `apps/workers/index.ts` | 58-61 |
| 队列接口定义 | `packages/shared/queueing.ts` | 1-102 |
| 队列重试调度 | `packages/plugins/queue-restate/src/dispatcher.ts` | 92-207 |
| 队列任务类型 | `packages/plugins/queue-restate/src/types.ts` | 19-35 |
