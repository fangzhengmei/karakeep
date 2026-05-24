# Karakeep 用户设置跨端同步与冲突处理分析

## 一、存储模型分析

### 1.1 数据库表结构

用户设置最初设计为独立的 `userSettings` 表（迁移 `0048_add_user_settings.sql`），但在迁移 `0061_merge_user_settings.sql` 中被合并到主 `user` 表中，作为直接列存储。

**当前表结构**（`packages/db/schema.ts:32-101`）：

```typescript
export const users = sqliteTable("user", {
  id: text("id").notNull().primaryKey(),
  // ... 用户基本信息 ...
  
  // === 用户设置列 ===
  bookmarkClickAction: text("bookmarkClickAction", {
    enum: ["open_original_link", "expand_bookmark_preview"],
  }).notNull().default("open_original_link"),
  
  archiveDisplayBehaviour: text("archiveDisplayBehaviour", {
    enum: ["show", "hide"],
  }).notNull().default("show"),
  
  timezone: text("timezone").default("UTC"),
  
  // 备份设置
  backupsEnabled: integer("backupsEnabled", { mode: "boolean" }).notNull().default(false),
  backupsFrequency: text("backupsFrequency", { enum: ["daily", "weekly"] }).notNull().default("weekly"),
  backupsRetentionDays: integer("backupsRetentionDays").notNull().default(30),
  
  // 阅读器设置（nullable = 可选，null 表示使用客户端默认值）
  readerFontSize: integer("readerFontSize"),
  readerLineHeight: real("readerLineHeight"),
  readerFontFamily: text("readerFontFamily", { enum: ["serif", "sans", "mono"] }),
  
  // AI 设置（nullable = 可选，null 表示使用服务器默认值）
  autoTaggingEnabled: integer("autoTaggingEnabled", { mode: "boolean" }),
  autoSummarizationEnabled: integer("autoSummarizationEnabled", { mode: "boolean" }),
  tagStyle: text("tagStyle", { enum: [...]}).default("titlecase-spaces"),
  curatedTagIds: text("curatedTagIds", { mode: "json" }).$type<string[]>(),
  inferredTagLang: text("inferredTagLang"),
});
```

**关键特征**：
- 设置项直接作为 `user` 表的列存储，无独立设置表
- **没有 `version` 或 `modifiedAt` 字段**用于并发控制
- 部分字段使用 `nullable` 设计，表示"未设置时使用默认值"

### 1.2 数据类型定义

设置项的 Zod schema 定义在 `packages/shared/types/users.ts:199-238`：

```typescript
// 完整设置 schema
export const zUserSettingsSchema = z.object({
  bookmarkClickAction: z.enum(["open_original_link", "expand_bookmark_preview"]),
  archiveDisplayBehaviour: z.enum(["show", "hide"]),
  timezone: z.string(),
  backupsEnabled: z.boolean(),
  backupsFrequency: z.enum(["daily", "weekly"]),
  backupsRetentionDays: z.number().int().min(1).max(365),
  readerFontSize: z.number().int().min(12).max(24).nullable(),
  readerLineHeight: z.number().min(1.2).max(2.5).nullable(),
  readerFontFamily: zReaderFontFamilySchema.nullable(),
  autoTaggingEnabled: z.boolean().nullable(),
  autoSummarizationEnabled: z.boolean().nullable(),
  tagStyle: zTagStyleSchema,
  curatedTagIds: z.array(z.string()).nullable(),
  inferredTagLang: z.string().nullable(),
});

// 部分更新 schema（允许只更新部分字段）
export const zUpdateUserSettingsSchema = zUserSettingsSchema.partial().pick({...});
```

### 1.3 本地设置 vs 服务端设置

系统中存在**两套独立的设置体系**：

| 设置类型 | 存储位置 | 同步范围 | 包含内容 |
|---------|---------|---------|---------|
| **服务端同步设置** | 数据库 `user` 表 | 跨端同步（Web ↔ Mobile） | 上述 14 个设置项 |
| **Web 本地设置** | Cookie（`hoarder-user-local-settings`） | 不同步 | 布局模式、语言、网格列数、显示选项等 |
| **Mobile 本地设置** | Expo SecureStore | 不同步 | API Key、服务器地址、主题、工具栏配置、图片质量等 |

**Web 本地设置**（`apps/web/lib/userLocalSettings/types.ts`）：
```typescript
const zUserLocalSettings = z.object({
  bookmarkGridLayout: zBookmarkGridLayout.optional().default("masonry"),
  lang: z.string().optional().default("en"),
  gridColumns: z.number().min(1).max(6).optional().default(3),
  showNotes: z.boolean().optional().default(false),
  showTags: z.boolean().optional().default(true),
  showTitle: z.boolean().optional().default(true),
  imageFit: z.enum(["cover", "contain"]).optional().default("cover"),
});
```

**Mobile 本地设置**（`apps/mobile/lib/settings.ts`）：
```typescript
const zSettingsSchema = z.object({
  apiKey: z.string().optional(),
  address: z.string().optional().default("https://cloud.karakeep.app"),
  imageQuality: z.number().optional().default(0.2),
  theme: z.enum(["light", "dark", "system"]).optional().default("system"),
  defaultBookmarkView: z.enum(["reader", "browser", "externalBrowser"]).optional().default("reader"),
  showNotes: z.boolean().optional().default(false),
  keepScreenOnWhileReading: z.boolean().optional().default(false),
  customHeaders: z.record(z.string(), z.string()).optional().default({}),
  readerFontSize: z.number().int().min(12).max(24).optional(),  // 阅读器本地覆盖
  readerLineHeight: z.number().min(1.2).max(2.5).optional(),    // 阅读器本地覆盖
  readerFontFamily: zReaderFontFamilySchema.optional(),          // 阅读器本地覆盖
  toolbarActions: z.array(zToolbarActionId).optional().default(...),
  overflowActions: z.array(zToolbarActionId).optional().default(...),
});
```

---

## 二、跨端同步触发机制

### 2.1 tRPC 接口

同步通过两个 tRPC procedure 实现（`packages/trpc/routers/users.ts:196-207`）：

```typescript
// 获取设置
settings: usersProcedure
  .output(zUserSettingsSchema)
  .query(async ({ ctx }) => {
    const user = await User.fromCtx(ctx);
    return await user.getSettings();
  }),

// 更新设置
updateSettings: usersProcedure
  .input(zUpdateUserSettingsSchema)
  .mutation(async ({ input, ctx }) => {
    const user = await User.fromCtx(ctx);
    await user.updateSettings(input);
  }),
```

### 2.2 服务端数据读写

**读取逻辑**（`packages/trpc/models/users.ts:467-511`）：
```typescript
async getSettings(): Promise<z.infer<typeof zUserSettingsSchema>> {
  const settings = await this.ctx.db.query.users.findFirst({
    where: eq(users.id, this.user.id),
    columns: { /* 所有设置列 */ },
  });
  // 返回时处理默认值
  return {
    bookmarkClickAction: settings.bookmarkClickAction,
    timezone: settings.timezone || "UTC",
    tagStyle: settings.tagStyle ?? "as-generated",
    // ...
  };
}
```

**写入逻辑**（`packages/trpc/models/users.ts:513-542`）：
```typescript
async updateSettings(input: z.infer<typeof zUpdateUserSettingsSchema>): Promise<void> {
  if (Object.keys(input).length === 0) {
    throw new TRPCError({ code: "BAD_REQUEST", message: "No settings provided" });
  }

  await this.ctx.db
    .update(users)
    .set({
      bookmarkClickAction: input.bookmarkClickAction,
      archiveDisplayBehaviour: input.archiveDisplayBehaviour,
      timezone: input.timezone,
      // ... 所有设置字段
    })
    .where(eq(users.id, this.user.id));  // 无版本检查！
}
```

### 2.3 Web 端同步流程

**React Query 封装**（`packages/shared-react/hooks/users.ts:7-21`）：
```typescript
export function useUpdateUserSettings(opts?) {
  const api = useTRPC();
  const queryClient = useQueryClient();
  return useMutation(
    api.users.updateSettings.mutationOptions({
      ...opts,
      onSuccess: (res, req, meta, context) => {
        // 更新成功后使缓存失效，触发重新拉取
        queryClient.invalidateQueries(api.users.settings.pathFilter());
        return opts?.onSuccess?.(res, req, meta, context);
      },
    }),
  );
}
```

**Context 初始化**（`apps/web/lib/userSettings.tsx:26-45`）：
```typescript
export function UserSettingsContextProvider({ userSettings, children }) {
  const api = useTRPC();
  const { data } = useQuery(
    api.users.settings.queryOptions(undefined, {
      initialData: userSettings,  // SSR 初始数据
    }),
  );
  return (
    <UserSettingsContext.Provider value={data}>
      {children}
    </UserSettingsContext.Provider>
  );
}
```

**UI 触发更新**（`apps/web/components/settings/UserOptions.tsx:60-72`）：
```typescript
const { mutate } = useUpdateUserSettings({
  onSuccess: () => {
    toast({ description: t("settings.info.user_settings.user_settings_updated") });
  },
  onError: () => {
    toast({ description: t("common.something_went_wrong"), variant: "destructive" });
  },
});

// 例如：时区变更时立即调用
onValueChange={(value) => {
  mutate({ timezone: value });
}}
```

### 2.4 Mobile 端同步流程

Mobile 端使用相同的 React Query 模式：

**读取设置**（`apps/mobile/lib/hooks.ts:47-59`）：
```typescript
export function useArchiveFilter(): { archived: false | undefined; isLoading: boolean } {
  const api = useTRPC();
  const { data: userSettings, isLoading } = useQuery(
    api.users.settings.queryOptions(),
  );
  return {
    archived: userSettings?.archiveDisplayBehaviour === "show" ? undefined : false,
    isLoading,
  };
}
```

**阅读器设置的特殊处理**：
Mobile 端通过 `ReaderSettingsProvider` 包装共享逻辑，实现本地覆盖 + 服务端同步的双层机制（`apps/mobile/lib/readerSettings.tsx`）。

### 2.5 同步触发时机

| 触发方式 | Web | Mobile |
|---------|-----|--------|
| **页面加载** | ✅ SSR 预取 + React Query 缓存 | ✅ React Query 懒加载 |
| **设置变更后** | ✅ `invalidateQueries` 触发重新拉取 | ✅ 相同机制 |
| **定时轮询** | ❌ 无自动轮询 | ❌ 无自动轮询 |
| **实时推送** | ❌ 无 WebSocket/Server-Sent Events | ❌ 无实时推送 |
| **应用回到前台** | ❌ 无显式刷新 | ❌ 无显式刷新 |

---

## 三、冲突解决策略分析

### 3.1 当前策略：Last Write Wins (LWW)

**核心问题**：系统**没有任何显式的冲突检测和解决机制**。

**证据**：
1. **无版本字段**：`user` 表没有 `version` 或 `settingsVersion` 列
2. **无条件更新**：`updateSettings` 中的 SQL 是：
   ```sql
   UPDATE user SET ... WHERE id = ?
   ```
   没有 `AND version = ?` 这样的乐观锁条件
3. **无时间戳比较**：没有 `lastModifiedAt` 字段用于比较新旧
4. **无合并逻辑**：服务端直接覆盖所有传入字段

**冲突场景示例**：
```
时序：
  T0: 服务端状态：bookmarkClickAction = "open_original_link"
  T1: Web 加载设置，缓存为 "open_original_link"
  T2: Mobile 加载设置，缓存为 "open_original_link"
  T3: Web 用户修改为 "expand_bookmark_preview" → 服务端更新成功
  T4: Mobile 用户修改为 "open_original_link"（未感知 Web 的变更）
  T5: Mobile 提交更新 → 服务端直接覆盖为 "open_original_link"
  
结果：
  Web 的变更被静默覆盖，用户无任何感知
```

### 3.2 阅读器设置的层级优先级

虽然没有冲突解决，但阅读器设置实现了**多层级优先级**用于本地显示（`packages/shared-react/hooks/reader-settings.tsx:110-132`）：

```typescript
// 优先级从高到低：
const settings: ReaderSettings = useMemo(() => ({
  fontSize:
    sessionOverrides.fontSize ??      // 1. 会话临时覆盖（仅 Web 预览用）
    localOverrides.fontSize ??        // 2. 设备本地覆盖（不同步）
    pendingServerSave?.fontSize ??    // 3. 待提交的服务器保存（乐观更新）
    serverSettings?.readerFontSize ?? // 4. 服务端同步值
    READER_DEFAULTS.fontSize,         // 5. 系统默认值
  // ... lineHeight, fontFamily 同理
}), [sessionOverrides, localOverrides, pendingServerSave, serverSettings]);
```

这是**显示优先级**，不是冲突解决策略。当用户选择"保存为默认值"时，会清除本地覆盖并同步到服务端。

### 3.3 乐观更新与回滚

在 `useReaderSettings` hook 中实现了**乐观更新**模式（`packages/shared-react/hooks/reader-settings.tsx:175-191`）：

```typescript
const saveAsDefault = useCallback((settingsToSave?) => {
  const toSave: ReaderSettings = { ... };
  setPendingServerSave(toSave);  // 立即设置为待保存状态（乐观显示）
  saveServerSettings({           // 异步提交到服务端
    readerFontSize: toSave.fontSize,
    readerLineHeight: toSave.lineHeight,
    readerFontFamily: toSave.fontFamily,
  });
}, [settings, saveServerSettings]);
```

失败时的回滚（`packages/shared-react/hooks/reader-settings.tsx:99-102`）：
```typescript
onError: () => {
  setPendingServerSave(null);  // 清除待保存状态，回退到服务器值
},
```

### 3.4 字段级部分更新

虽然没有冲突检测，但支持**字段级部分更新**：

```typescript
// 可以只更新单个字段
mutate({ timezone: "Asia/Shanghai" });
mutate({ autoTaggingEnabled: true });
mutate({ bookmarkClickAction: "expand_bookmark_preview", tagStyle: "lowercase-hyphens" });
```

这减少了冲突概率（只更新变更的字段），但如果两端同时修改**不同字段**，后提交的仍会覆盖先提交的**相同字段**。

---

## 四、问题总结与改进建议

### 4.1 现存问题

| 问题 | 风险等级 | 说明 |
|-----|---------|------|
| **无冲突检测** | 🔴 高 | 并发修改时后写入者静默覆盖先写入者，用户无感知 |
| **无实时同步** | 🟡 中 | 设置变更后，另一端必须刷新页面才能看到更新 |
| **设置分散** | 🟡 中 | 三套设置体系（服务端/Web本地/Mobile本地）增加认知负担 |
| **无修改历史** | 🟠 中高 | 无法追溯设置变更历史，问题排查困难 |

### 4.2 改进建议

#### 建议 1：添加乐观锁机制（高优先级）

在 `user` 表增加 `settingsVersion` 字段，更新时进行版本检查：

```sql
-- 新增字段
ALTER TABLE user ADD COLUMN settingsVersion INTEGER NOT NULL DEFAULT 0;

-- 更新时检查版本
UPDATE user 
SET bookmarkClickAction = ?, settingsVersion = settingsVersion + 1
WHERE id = ? AND settingsVersion = ?;

-- 如果影响行数为 0，说明有冲突，需要处理
```

#### 建议 2：添加 `settingsModifiedAt` 时间戳

用于客户端检测是否需要刷新：

```sql
ALTER TABLE user ADD COLUMN settingsModifiedAt INTEGER NOT NULL DEFAULT 0;
```

#### 建议 3：实现字段级冲突检测

如果两个请求修改不同字段，允许合并；如果修改相同字段，按策略解决：

```typescript
async updateSettings(input, currentVersion) {
  await this.ctx.db.transaction(async (tx) => {
    const current = await tx.query.users.findFirst({
      where: eq(users.id, this.user.id),
      columns: { settingsVersion: true, bookmarkClickAction: true, ... },
    });
    
    if (current.settingsVersion !== currentVersion) {
      // 检测冲突
      const merged = { ...current };
      for (const [key, value] of Object.entries(input)) {
        // 只合并没有被其他人修改的字段
        if (current[key] === originalFromClient[key]) {
          merged[key] = value;
        } else {
          // 冲突字段，按策略处理（LWW / 提示用户 / 保留较新值）
        }
      }
      // ...
    }
  });
}
```

#### 建议 4：考虑实时同步机制

使用 WebSocket 或 Server-Sent Events 推送设置变更：

```typescript
// 服务端更新设置后广播事件
eventBus.emit(`user:${userId}:settings-updated`, newSettings);
```

#### 建议 5：统一设置模型

考虑将更多本地设置纳入同步范围，减少"本地 vs 服务端"的概念割裂。

---

## 五、关键代码引用

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| 数据库 schema | `packages/db/schema.ts` | 32-101 |
| 设置类型定义 | `packages/shared/types/users.ts` | 199-238 |
| tRPC 接口 | `packages/trpc/routers/users.ts` | 196-207 |
| 服务端读取 | `packages/trpc/models/users.ts` | 467-511 |
| 服务端写入 | `packages/trpc/models/users.ts` | 513-542 |
| Web React Query hook | `packages/shared-react/hooks/users.ts` | 7-21 |
| Web Context Provider | `apps/web/lib/userSettings.tsx` | 26-45 |
| 阅读器设置 hook | `packages/shared-react/hooks/reader-settings.tsx` | 42-248 |
| Mobile 本地设置 | `apps/mobile/lib/settings.ts` | 37-141 |
| Mobile 阅读器设置 Provider | `apps/mobile/lib/readerSettings.tsx` | 44-90 |
| Web 本地设置 | `apps/web/lib/userLocalSettings/types.ts` | 8-16 |
| 单元测试 | `packages/trpc/routers/users.test.ts` | 167-248 |
