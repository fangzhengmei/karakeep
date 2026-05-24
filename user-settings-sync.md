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

系统中存在**三套独立的设置体系**，但**服务端存储的14个设置项并非全部在移动端同步或修改：

| 设置类型 | 存储位置 | 同步范围 | 包含内容 |
|---------|---------|---------|---------|
| **服务端存储设置** | 数据库 `user` 表 | Web 可读写全部 14 项，移动端仅读写 3 项 | bookmarkClickAction、archiveDisplayBehaviour、timezone、backupsEnabled、backupsFrequency、backupsRetentionDays、readerFontSize、readerLineHeight、readerFontFamily、autoTaggingEnabled、autoSummarizationEnabled、tagStyle、curatedTagIds、inferredTagLang |
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

### 1.4 移动端设置分类与同步边界

移动端的设置体系与 Web 端有显著差异。**移动端没有直接调用 `useUpdateUserSettings`**，所有服务端设置更新仅通过阅读器设置的共享 hook 间接完成。

#### 移动端对服务端设置的操作权限

| 服务端设置字段 | 移动端读取 | 移动端写入 | 备注 |
|---------------|-----------|-----------|------|
| `readerFontSize` | ✅ 读取 | ✅ 主动写入 | 仅在点击 "Save as Default (All Devices)" 时同步 |
| `readerLineHeight` | ✅ 读取 | ✅ 主动写入 | 同上 |
| `readerFontFamily` | ✅ 读取 | ✅ 主动写入 | 同上 |
| `archiveDisplayBehaviour` | ✅ 读取 | ❌ 不写入 | 仅用于 `useArchiveFilter` hook 判断是否显示已归档 |
| `bookmarkClickAction` | ❌ 不读取 | ❌ 不写入 | 移动端无此概念 |
| `timezone` | ❌ 不读取 | ❌ 不写入 | 移动端无设置界面 |
| `backupsEnabled` | ❌ 不读取 | ❌ 不写入 | 移动端无备份功能 |
| `backupsFrequency` | ❌ 不读取 | ❌ 不写入 | 同上 |
| `backupsRetentionDays` | ❌ 不读取 | ❌ 不写入 | 同上 |
| `autoTaggingEnabled` | ❌ 不读取 | ❌ 不写入 | 移动端无 AI 设置界面 |
| `autoSummarizationEnabled` | ❌ 不读取 | ❌ 不写入 | 同上 |
| `tagStyle` | ❌ 不读取 | ❌ 不写入 | 同上 |
| `curatedTagIds` | ❌ 不读取 | ❌ 不写入 | 同上 |
| `inferredTagLang` | ❌ 不读取 | ❌ 不写入 | 同上 |

#### 移动端设置分类详情

**✅ 会同步到服务端的设置（仅 3 项）**：
- `readerFontSize` - 阅读字体大小
- `readerLineHeight` - 阅读行高
- `readerFontFamily` - 阅读字体

同步触发条件（两条独立写回路径）：
1. **路径 1：保存为默认（saveAsDefault）** - 用户点击 **"Save as Default (All Devices)"** 按钮，写入具体值
2. **路径 2：清除服务器默认（clearAllDefaults）** - 用户点击 **"Clear Server Defaults"** 按钮，将字段设为 `null`

**⚠️ 仅从服务端读取但不修改的设置（仅 1 项）**：
- `archiveDisplayBehaviour` - 用于判断列表中是否显示已归档书签

**❌ 仅本地生效、永不同步的设置（10 项）**：
- `apiKey` / `apiKeyId` - API 凭证
- `address` - 服务器地址
- `imageQuality` - 上传图片质量
- `theme` - 主题（light/dark/system）
- `defaultBookmarkView` - 默认书签打开方式（reader/browser/externalBrowser）
- `showNotes` - 是否在书签卡片中显示笔记
- `keepScreenOnWhileReading` - 阅读时保持屏幕常亮
- `customHeaders` - 自定义请求头
- `toolbarActions` / `overflowActions` - 工具栏按钮配置
- 阅读器设置的**本地覆盖**（未点击 "Save as Default" 前）

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

Mobile 端**不使用** `useUpdateUserSettings` hook，所有服务端设置更新均通过 `useReaderSettings` 共享 hook 间接完成。

**仅读取服务端设置**（`apps/mobile/lib/hooks.ts:47-59`）：
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

**阅读器设置的同步流程**：
Mobile 端通过 `ReaderSettingsProvider`（`apps/mobile/lib/readerSettings.tsx`）包装共享的 `useReaderSettings` hook，实现本地覆盖 + 服务端同步的双层机制。

移动端调用 `updateLocal()` 仅修改本地覆盖，不会同步到服务端。只有当用户显式操作时，才会通过两条独立路径写回服务端。

#### 2.4.1 阅读器设置的两条写回路径

共享 hook `useReaderSettings`（`packages/shared-react/hooks/reader-settings.tsx`）中实现了两条独立的写回路径，Web 端和移动端共享同一套逻辑，但使用方式略有差异。

##### 路径 1：saveAsDefault（保存为默认）

**代码实现**（`packages/shared-react/hooks/reader-settings.tsx:175-191`）：
```typescript
const saveAsDefault = useCallback(
  (settingsToSave?: ReaderSettingsPartial) => {
    const toSave: ReaderSettings = {
      fontSize: settingsToSave?.fontSize ?? settings.fontSize,
      lineHeight: settingsToSave?.lineHeight ?? settings.lineHeight,
      fontFamily: settingsToSave?.fontFamily ?? settings.fontFamily,
    };
    // Set pending state to prevent flicker while server syncs
    setPendingServerSave(toSave);
    saveServerSettings({
      readerFontSize: toSave.fontSize,
      readerLineHeight: toSave.lineHeight,
      readerFontFamily: toSave.fontFamily,
    });
  },
  [settings, saveServerSettings],
);
```

**使用的 mutation**：`saveServerSettings`（第 90-107 行）
- 成功回调：自动清除本地覆盖（localOverrides）和会话覆盖（sessionOverrides）
- 乐观更新：设置 `pendingServerSave`，防止同步期间的闪烁
- 错误回滚：失败时清除 `pendingServerSave` 状态

**字段更新特点**：
| 端 | 调用方式 | 更新字段 | 触发场景 |
|----|---------|---------|---------|
| **Web 端** | `saveAsDefault({ fontSize: 18 })` | 可单个字段或全部 3 个 | 1. 阅读器设置页面修改下拉/滑块时立即同步单个字段<br>2. 预览弹窗点击 "Save to All Devices" 同步全部 3 个 |
| **Mobile 端** | `saveAsDefault()`（不传参数） | **总是同时更新全部 3 个字段** | 阅读器设置页面点击 "Save as Default (All Devices)" 按钮 |

##### 路径 2：clearAllDefaults（清除服务器默认）

**代码实现**（`packages/shared-react/hooks/reader-settings.tsx:207-213`）：
```typescript
const clearAllDefaults = useCallback(() => {
  updateServerSettings({
    readerFontSize: null,
    readerLineHeight: null,
    readerFontFamily: null,
  });
}, [updateServerSettings]);
```

**使用的 mutation**：`updateServerSettings`（第 81-87 行）
- 成功回调：仅 `refetchQueries`，**不清除本地覆盖**
- 乐观更新：无 `pending` 状态
- 语义：将字段设为 `null`，表示"使用系统默认值"（`READER_DEFAULTS`）

**字段更新特点**：
| 端 | 更新字段 | 触发场景 |
|----|---------|---------|
| **Web 端** | 3 个字段同时设为 `null` | 阅读器设置页面点击 "Clear defaults" 按钮 |
| **Mobile 端** | 3 个字段同时设为 `null` | 阅读器设置页面点击 "Clear Server Defaults" 按钮 |

**重要特性**：`clearAllDefaults` 不会清除本地覆盖。如果某设备有本地覆盖，清除服务器默认后，该设备仍会显示本地覆盖值，而其他设备会回退到系统默认值。

##### 附加路径：clearDefault（清除单个服务器默认）

**代码实现**（`packages/shared-react/hooks/reader-settings.tsx:194-204`）：
```typescript
const clearDefault = useCallback(
  (key: keyof ReaderSettings) => {
    const serverKeyMap = {
      fontSize: "readerFontSize",
      lineHeight: "readerLineHeight",
      fontFamily: "readerFontFamily",
    } as const;
    updateServerSettings({ [serverKeyMap[key]]: null });
  },
  [updateServerSettings],
);
```

**使用情况**：目前在 Web 端和移动端都没有 UI 调用此函数，仅作为 API 导出。

#### 2.4.2 移动端同步代码示例

```typescript
// 仅修改本地，不同步（apps/mobile/app/dashboard/settings/reader-settings.tsx:100-106）
const handleFontSizeChange = (value: number) => {
  updateLocal({ fontSize: Math.round(value) });
};

// 路径 1：显式点击才同步到服务端（写入具体值）
const handleSaveAsDefault = () => {
  saveAsDefault();  // 内部调用 saveServerSettings({ readerFontSize, readerLineHeight, readerFontFamily })
};

// 路径 2：清除服务器默认（设为 null）
const handleClearServerDefaults = () => {
  clearAllDefaults();  // 内部调用 updateServerSettings({ readerFontSize: null, readerLineHeight: null, readerFontFamily: null })
};
```

### 2.5 同步触发时机

| 触发方式 | Web | Mobile |
|---------|-----|--------|
| **页面加载** | ✅ SSR 预取 + React Query 缓存 | ✅ React Query 懒加载（仅阅读器设置和 archiveDisplayBehaviour） |
| **设置变更后主动刷新** | ✅ 所有 14 项设置变更后均 `invalidateQueries` 触发重新拉取 | ⚠️ 仅阅读器设置同步后 `refetchQueries`，其他设置永不同步 |
| **设置变更触发同步** | ✅ 所有设置变更立即同步到服务端<br>阅读器设置支持单字段即时同步 | ⚠️ 仅阅读器设置同步，且必须用户显式操作：<br>• 点击 "Save as Default" → saveAsDefault 路径<br>• 点击 "Clear Server Defaults" → clearAllDefaults 路径<br>其他设置仅本地保存 |
| **定时轮询** | ❌ 无自动轮询 | ❌ 无自动轮询 |
| **实时推送** | ❌ 无 WebSocket/Server-Sent Events | ❌ 无实时推送 |
| **应用回到前台** | ❌ 无显式刷新 | ❌ 无显式刷新 |

### 2.6 两条写回路径的行为对比

| 维度 | 路径 1：saveAsDefault | 路径 2：clearAllDefaults |
|-----|----------------------|-------------------------|
| **触发按钮** | "Save as Default (All Devices)" | "Clear Server Defaults" |
| **写入值** | 具体的非 null 数值 | `null`（表示使用系统默认） |
| **Web 端字段范围** | 可单字段或全部 3 个 | 总是全部 3 个字段 |
| **Mobile 端字段范围** | 总是全部 3 个字段 | 总是全部 3 个字段 |
| **使用的 mutation** | `saveServerSettings` | `updateServerSettings` |
| **成功后清除本地覆盖** | ✅ 自动清除 | ❌ 不清除，本地覆盖继续生效 |
| **乐观更新** | ✅ 设置 pendingServerSave | ❌ 无 pending 状态 |
| **冲突风险** | 🟡 中：可能覆盖其他端的同字段修改 | 🟠 中高：一次性清空 3 个字段，可能覆盖其他端的所有阅读器设置 |

---

## 三、冲突解决策略分析

### 3.1 当前策略：Last Write Wins (LWW)

**核心问题**：系统**没有任何显式的冲突检测和解决机制**。但需要注意的是，**冲突仅可能发生在阅读器的 3 个设置字段上**，因为这是移动端唯一会写入的服务端设置。

**证据**：
1. **无版本字段**：`user` 表没有 `version` 或 `settingsVersion` 列
2. **无条件更新**：`updateSettings` 中的 SQL 是：
   ```sql
   UPDATE user SET ... WHERE id = ?
   ```
   没有 `AND version = ?` 这样的乐观锁条件
3. **无时间戳比较**：没有 `lastModifiedAt` 字段用于比较新旧
4. **无合并逻辑**：服务端直接覆盖所有传入字段

### 3.1.1 两条写回路径对冲突范围的影响

由于两条写回路径更新字段的范围不同，冲突的风险和影响范围也有显著差异：

| 写回路径 | Web 端更新字段 | 移动端更新字段 | 冲突风险 | 影响范围 |
|---------|---------------|---------------|---------|---------|
| **saveAsDefault（单字段）** | 1 个字段（fontSize/lineHeight/fontFamily） | ❌ 不支持 | 🟡 中 | 仅单个字段可能被覆盖 |
| **saveAsDefault（全字段）** | 3 个字段同时 | 3 个字段同时 | 🟡 中高 | 可能覆盖其他端对任意阅读器字段的修改 |
| **clearAllDefaults** | 3 个字段同时设为 null | 3 个字段同时设为 null | 🟠 高 | 一次性清空所有阅读器默认值，覆盖其他端的所有修改 |

### 3.1.2 冲突场景示例

所有冲突仅可能发生在阅读器的 3 个字段上。

#### 场景 1：Web 单字段更新 vs 移动端全字段更新（saveAsDefault）
```
时序：
  T0: 服务端 fontSize=16, lineHeight=1.5, fontFamily=sans
  T1: Web 用户修改 fontSize 为 18 → 仅更新 readerFontSize=18（Web 支持单字段更新）
  T2: Mobile 用户在阅读器设置中调整字体为 20（仅本地覆盖）
  T3: Mobile 用户点击 "Save as Default" → 同时提交 fontSize=20, lineHeight=1.5, fontFamily=sans
  T4: 服务端所有 3 个字段被移动端覆盖
  
结果：
  Web 端的 fontSize 修改（18）被静默覆盖为 20
  Web 端需要刷新页面才能看到移动端同步的值
  更隐蔽的风险：即使 lineHeight 和 fontFamily 没有变化，移动端也会强制覆盖
```

#### 场景 2：两端同时 saveAsDefault
```
时序：
  T0: 服务端 fontSize=16
  T1: Web 用户点击 "Save to All Devices" → 提交 fontSize=18
  T2: 服务端更新为 18，但移动端缓存仍为 16
  T3: Mobile 用户点击 "Save as Default" → 提交 fontSize=20
  T4: 服务端直接覆盖为 20
  
结果：
  Web 端的变更被静默覆盖，用户无任何感知
```

#### 场景 3：clearAllDefaults 冲突（风险最高）
```
时序：
  T0: 服务端 fontSize=18, lineHeight=1.6, fontFamily=serf
  T1: Web 用户修改 fontFamily 为 sans → 仅更新 readerFontFamily=sans
  T2: Mobile 用户点击 "Clear Server Defaults" → 同时设置 3 个字段为 null
  T3: 服务端所有 3 个字段变为 null，回退到系统默认值
  
结果：
  Web 端的 fontFamily 修改被清空
  所有设备的阅读器设置都回退到系统默认值（READER_DEFAULTS）
  如果某设备有本地覆盖，该设备仍显示本地值，其他设备显示默认值，出现多端不一致
```

**不可能发生冲突的字段**：
- `bookmarkClickAction`、`timezone`、`backups*`、`autoTaggingEnabled` 等 10 个字段：移动端完全不写入，只能在 Web 端修改，不存在跨端冲突
- `archiveDisplayBehaviour`：移动端只读不写，也不会发生冲突
- 移动端本地设置（主题、工具栏、图片质量等）：仅本地存储，不存在跨端冲突

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

虽然没有冲突检测，但支持**字段级部分更新**。

#### Web 端更新能力

Web 端可以更新任意单个或多个字段：
```typescript
// 通过 useUpdateUserSettings 更新非阅读器设置
mutate({ timezone: "Asia/Shanghai" });
mutate({ autoTaggingEnabled: true });
mutate({ bookmarkClickAction: "expand_bookmark_preview", tagStyle: "lowercase-hyphens" });

// 通过阅读器设置 hook 单字段更新
updateServerSetting({ fontSize: 18 });           // 仅更新 readerFontSize
updateServerSetting({ lineHeight: 1.6 });        // 仅更新 readerLineHeight
updateServerSetting({ fontFamily: "mono" });     // 仅更新 readerFontFamily
```

#### 移动端更新能力

移动端只能更新阅读器的 3 个字段，且**两条写回路径都总是同时更新全部 3 个字段**：
```typescript
// 路径 1：saveAsDefault 总是同时提交 3 个阅读器字段的具体值
saveServerSettings({
  readerFontSize: toSave.fontSize,
  readerLineHeight: toSave.lineHeight,
  readerFontFamily: toSave.fontFamily,
});

// 路径 2：clearAllDefaults 总是同时将 3 个字段设为 null
updateServerSettings({
  readerFontSize: null,
  readerLineHeight: null,
  readerFontFamily: null,
});
```

#### 对冲突的影响

- Web 端的单字段更新能力**降低了冲突概率**（只更新真正变更的字段）
- 移动端的全字段更新方式**扩大了冲突范围**（即使只修改了字体大小，也会同时覆盖行高和字体）
- 这是一个**不对称的设计**：Web 端可以精细地单字段更新，而移动端每次同步都会"冲刷"全部 3 个字段，增加了意外覆盖的风险

---

## 四、问题总结与改进建议

### 4.1 现存问题

| 问题 | 风险等级 | 说明 |
|-----|---------|------|
| **阅读器设置无冲突检测** | 🟡 中 | 仅阅读器的 3 个字段可能发生跨端冲突，后写入者静默覆盖先写入者，用户无感知 |
| **clearAllDefaults 冲突风险高** | 🟠 中高 | 清除服务器默认会同时清空 3 个字段，且不清除本地覆盖，容易导致多端不一致和意外覆盖 |
| **更新粒度不对称** | 🟡 中 | Web 端支持阅读器设置单字段更新，移动端两条路径都总是同时更新全部 3 个字段，即使某些字段没有变化也会被强制覆盖 |
| **无实时同步** | 🟡 中 | 设置变更后，另一端必须刷新页面才能看到更新。例如 Web 端修改了阅读器字体，移动端必须重新进入阅读器设置页面才能看到 |
| **设置分散，权限不对称** | 🟡 中 | 三套设置体系（服务端/Web本地/Mobile本地），且 Web 端可修改全部 14 个服务端设置，移动端仅能修改 3 个，概念不统一 |
| **无修改历史** | 🟠 中高 | 无法追溯设置变更历史，问题排查困难 |
| **移动端同步触发不透明** | 🟡 中 | 阅读器设置的 "Save as Default" 和 "Clear Server Defaults" 是移动端仅有的两个同步入口，但用户可能不清楚哪些设置会同步、哪些仅本地 |

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

#### 建议 5：统一更新粒度，减少移动端全字段覆盖

修改移动端 `saveAsDefault` 的实现，支持只更新真正变更的字段，而不是每次都强制覆盖全部 3 个字段：

```typescript
// 改进前：总是同时更新 3 个字段
saveAsDefault()  // → readerFontSize, readerLineHeight, readerFontFamily 全部被覆盖

// 改进后：仅更新有本地覆盖的字段，或与服务端不同的字段
saveAsDefault()  // → 只更新实际变更的字段
```

这可以显著降低意外覆盖其他端修改的风险。

#### 建议 6：修复 clearAllDefaults 的行为一致性

`clearAllDefaults` 应该与 `saveAsDefault` 保持一致，成功后也清除本地覆盖，避免出现"服务器已清空但本地仍显示旧值"的多端不一致问题。或者在 UI 上明确提示用户"此操作不会影响本设备的本地设置"。

#### 建议 7：提高移动端同步透明度

在移动端设置界面明确标注哪些设置仅本地生效、哪些会跨端同步。例如：
- 在阅读器设置页面更明确地提示 "Save as Default" 会同步到所有设备
- 明确提示 "Clear Server Defaults" 会清空所有设备的默认设置，但保留本地设置
- 在其他仅本地的设置旁边添加 "This device only" 提示

#### 建议 8：统一设置模型

考虑将更多移动端本地设置（如主题、默认书签视图）纳入同步范围，减少"本地 vs 服务端"的概念割裂。或者在移动端提供与 Web 端一致的完整设置界面，允许用户修改所有 14 个服务端设置。

---

## 五、关键代码引用

### 核心逻辑

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| 数据库 schema | `packages/db/schema.ts` | 32-101 |
| 设置类型定义 | `packages/shared/types/users.ts` | 199-238 |
| tRPC 接口 | `packages/trpc/routers/users.ts` | 196-207 |
| 服务端读取 | `packages/trpc/models/users.ts` | 467-511 |
| 服务端写入 | `packages/trpc/models/users.ts` | 513-542 |
| Web React Query hook | `packages/shared-react/hooks/users.ts` | 7-21 |
| Web Context Provider | `apps/web/lib/userSettings.tsx` | 26-45 |
| Web 设置界面 | `apps/web/components/settings/UserOptions.tsx` | 60-72 |

### 阅读器设置两条写回路径

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| 阅读器设置共享 hook（两条路径实现） | `packages/shared-react/hooks/reader-settings.tsx` | 42-248 |
| → 路径 1：saveAsDefault 实现 | `packages/shared-react/hooks/reader-settings.tsx` | 175-191 |
| → 路径 2：clearAllDefaults 实现 | `packages/shared-react/hooks/reader-settings.tsx` | 207-213 |
| → 附加：clearDefault 实现 | `packages/shared-react/hooks/reader-settings.tsx` | 194-204 |
| → saveServerSettings mutation（路径 1 使用） | `packages/shared-react/hooks/reader-settings.tsx` | 90-107 |
| → updateServerSettings mutation（路径 2 使用） | `packages/shared-react/hooks/reader-settings.tsx` | 81-87 |

### 移动端代码

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| Mobile 本地设置 | `apps/mobile/lib/settings.ts` | 37-141 |
| Mobile 阅读器设置 Provider | `apps/mobile/lib/readerSettings.tsx` | 44-90 |
| Mobile 阅读器设置页面（两条同步入口） | `apps/mobile/app/dashboard/settings/reader-settings.tsx` | 1-271 |
| → 路径 1 调用：handleSaveAsDefault | `apps/mobile/app/dashboard/settings/reader-settings.tsx` | 108-111 |
| → 路径 2 调用：handleClearServerDefaults | `apps/mobile/app/dashboard/settings/reader-settings.tsx` | 117-119 |
| Mobile 设置主页面 | `apps/mobile/app/dashboard/settings/index.tsx` | 1-404 |
| Mobile 读取 archiveDisplayBehaviour | `apps/mobile/lib/hooks.ts` | 47-59 |

### Web 端代码

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| Web 阅读器设置 hook 包装 | `apps/web/lib/readerSettings.tsx` | 1-155 |
| Web 阅读器设置页面 | `apps/web/components/settings/ReaderSettings.tsx` | 1-270 |
| Web 阅读器预览弹窗 | `apps/web/components/dashboard/preview/ReaderSettingsPopover.tsx` | 1-460 |
| Web 本地设置 | `apps/web/lib/userLocalSettings/types.ts` | 8-16 |

### 测试

| 功能 | 文件位置 | 行号 |
|-----|---------|------|
| 单元测试 | `packages/trpc/routers/users.test.ts` | 167-248 |
