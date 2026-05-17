# Karakeep 协作列表邀请流程分析

## 概述

Karakeep 的协作列表邀请流程是一个完整的端到端流程，涵盖了从邀请创建、接收确认、权限落地到成员视图更新的全链路协作机制。该流程基于 tRPC + Drizzle ORM 架构实现，采用了事务处理保证数据一致性，并通过细粒度的权限控制确保协作安全。

## 核心数据表结构

### 1. 列表邀请表 (`listInvitations`)

```typescript
// packages/db/schema.ts:562-593
export const listInvitations = sqliteTable("listInvitations", {
  id: text("id").notNull().primaryKey(),
  listId: text("listId").notNull().references(() => bookmarkLists.id),
  userId: text("userId").notNull().references(() => users.id),
  role: text("role", { enum: ["viewer", "editor"] }).notNull(),
  status: text("status", { enum: ["pending", "declined"] }).notNull().default("pending"),
  invitedAt: integer("invitedAt", { mode: "timestamp" }).notNull(),
  invitedEmail: text("invitedEmail"),
  invitedBy: text("invitedBy").references(() => users.id),
});
```

### 2. 列表协作者表 (`listCollaborators`)

```typescript
// packages/db/schema.ts:536-560
export const listCollaborators = sqliteTable("listCollaborators", {
  id: text("id").notNull().primaryKey(),
  listId: text("listId").notNull().references(() => bookmarkLists.id),
  userId: text("userId").notNull().references(() => users.id),
  role: text("role", { enum: ["viewer", "editor"] }).notNull(),
  addedAt: createdAtField(),
  addedBy: text("addedBy").references(() => users.id),
});
```

## 流程详解

### 阶段一：邀请创建

#### 1. 前端触发 - 管理协作者模态框

**文件：** `apps/web/components/dashboard/lists/ManageCollaboratorsModal.tsx`

- 列表所有者通过 `ManageCollaboratorsModal` 组件打开协作者管理界面
- 输入被邀请者邮箱，选择角色（`viewer` 或 `editor`）
- 调用 `lists.addCollaborator` mutation

```typescript
// ManageCollaboratorsModal.tsx:166-180
const handleAddCollaborator = () => {
  addCollaborator.mutate({
    listId: list.id,
    email: newCollaboratorEmail,
    role: newCollaboratorRole,
  });
};
```

#### 2. 后端路由 - 权限校验与调用

**文件：** `packages/trpc/routers/lists.ts:250-279`

- 经过两层中间件校验：
  - `ensureListAtLeastViewer`：确保用户至少有查看权限
  - `ensureListAtLeastOwner`：确保用户是列表所有者
- 限流控制：15 分钟内最多 20 次邀请

```typescript
// lists.ts:250-279
addCollaborator: listsProcedure
  .input(z.object({
    listId: z.string(),
    email: z.string().email(),
    role: z.enum(["viewer", "editor"]),
  }))
  .use(createRateLimitMiddleware({
    name: "lists.addCollaborator",
    windowMs: 15 * 60 * 1000,
    maxRequests: 20,
  }))
  .use(ensureListAtLeastViewer)
  .use(ensureListAtLeastOwner)
  .mutation(async ({ input, ctx }) => {
    return {
      invitationId: await ctx.list.addCollaboratorByEmail(
        input.email,
        input.role,
      ),
    };
  }),
```

#### 3. 业务模型 - 邀请创建逻辑

**文件：** `packages/trpc/models/lists.ts:689-705`

List 模型的 `addCollaboratorByEmail` 方法委托给 `ListInvitation.inviteByEmail` 处理。

#### 4. 邀请业务逻辑 - ListInvitation 模型

**文件：** `packages/trpc/models/listInvitations.ts:174-293`

`inviteByEmail` 方法执行以下校验和操作：

| 校验项 | 说明 | 错误处理 |
|--------|------|---------|
| 用户存在性 | 根据邮箱查找用户 | NOT_FOUND: "No user found with that email address" |
| 非列表所有者 | 不能邀请列表所有者自己 | BAD_REQUEST: "Cannot add the list owner as a collaborator" |
| 列表类型 | 只有手动列表支持协作 | BAD_REQUEST: "Only manual lists can have collaborators" |
| 已为协作者 | 检查是否已经是协作者 | BAD_REQUEST: "User is already a collaborator on this list" |
| 待处理邀请 | 检查是否已有待处理邀请 | BAD_REQUEST: "User already has a pending invitation for this list" |

如果用户之前拒绝过邀请（status = "declined"），则更新邀请状态为 "pending" 并重新发送邮件。

创建邀请后，调用 `sendInvitationEmail` 发送通知邮件。

#### 5. 邮件通知

**文件：** `packages/trpc/email.ts:194-241`

```typescript
// email.ts:203
const inviteUrl = `${serverConfig.publicUrl}/dashboard/lists?pendingInvitation=${encodeURIComponent(listId)}`;
```

邮件包含指向 `/dashboard/lists` 的链接，带有 `pendingInvitation` 查询参数。

---

### 阶段二：接收确认

#### 1. 邀请通知展示

**文件：** `apps/web/components/dashboard/sidebar/InvitationNotificationBadge.tsx`

- 侧边栏徽章显示待处理邀请数量
- 每 5 分钟自动刷新一次
- 调用 `lists.getPendingInvitations` 查询

```typescript
// InvitationNotificationBadge.tsx:9-13
const { data: pendingInvitations } = useQuery(
  api.lists.getPendingInvitations.queryOptions(undefined, {
    refetchInterval: 1000 * 60 * 5,
  }),
);
```

#### 2. 待处理邀请卡片

**文件：** `apps/web/components/dashboard/lists/PendingInvitationsCard.tsx`

在列表页面顶部展示所有待处理邀请：

- 显示列表名称、图标、描述
- 显示邀请者姓名和角色
- 提供接受和拒绝按钮

#### 3. 接受邀请流程

**前端：** `PendingInvitationsCard.tsx:36-56`

调用 `lists.acceptInvitation` mutation，成功后：
- 刷新待处理邀请列表
- 刷新用户列表（新协作列表将出现）

**后端路由：** `lists.ts:344-353`

```typescript
// lists.ts:344-353
acceptInvitation: listsProcedure
  .input(z.object({ invitationId: z.string() }))
  .use(ensureInvitationAccess)
  .mutation(async ({ ctx }) => {
    await ctx.invitation.accept();
  }),
```

#### 4. 邀请访问中间件

**文件：** `lists.ts:60-74`

`ensureInvitationAccess` 中间件：
- 调用 `ListInvitation.fromId` 加载邀请
- 校验当前用户是被邀请者或列表所有者
- 将邀请对象注入上下文

---

### 阶段三：权限落地

#### 1. 接受邀请 - 事务处理

**文件：** `listInvitations.ts:112-137`

`accept` 方法在数据库事务中执行两个关键操作：

```typescript
// listInvitations.ts:122-136
await this.ctx.db.transaction(async (tx) => {
  // 1. 删除邀请记录
  await tx
    .delete(listInvitations)
    .where(eq(listInvitations.id, this.invitation.id));

  // 2. 插入协作者记录（幂等操作）
  await tx
    .insert(listCollaborators)
    .values({
      listId: this.invitation.listId,
      userId: this.invitation.userId,
      role: this.invitation.role,
      addedBy: this.invitation.invitedBy,
    })
    .onConflictDoNothing();
});
```

**事务保证：** 邀请删除和协作者添加是原子操作，确保数据一致性。

**幂等处理：** 使用 `onConflictDoNothing` 处理重复接受的情况。

#### 2. 拒绝邀请

**文件：** `listInvitations.ts:142-158`

将邀请状态更新为 "declined"，保留记录以便后续重新邀请。

#### 3. 权限模型

**文件：** `lists.ts:397-429`

| 角色 | 查看 | 编辑（添加/移除书签） | 管理（协作者、设置） |
|------|------|----------------------|----------------------|
| owner | ✅ | ✅ | ✅ |
| editor | ✅ | ✅ | ❌ |
| viewer | ✅ | ❌ | ❌ |
| public | ✅ | ❌ | ❌ |

权限校验通过中间件实现：
- `ensureListAtLeastViewer` - 查看操作
- `ensureListAtLeastEditor` - 编辑操作
- `ensureListAtLeastOwner` - 管理操作

---

### 阶段四：成员视图更新

#### 1. 列表加载 - 双路径查询

**文件：** `lists.ts:85-159`

`List.fromId` 方法采用两级查询策略：

**路径 1 - 所有者路径：**
```sql
SELECT * FROM bookmarkLists 
WHERE id = ? AND userId = currentUserId
```

**路径 2 - 协作者路径：**
```sql
SELECT lc.*, l.* FROM listCollaborators lc
JOIN bookmarkLists l ON lc.listId = l.id
WHERE lc.listId = ? AND lc.userId = currentUserId
```

根据查询结果设置 `userRole` 字段，用于后续权限判断。

#### 2. 获取用户所有列表

**文件：** `lists.ts:282-288`

```typescript
// lists.ts:282-288
static async getAll(ctx: AuthedContext) {
  const [ownedLists, sharedLists] = await Promise.all([
    this.getAllOwned(ctx),
    this.getSharedWithUser(ctx),
  ]);
  return [...ownedLists, ...sharedLists];
}
```

并行查询用户拥有的列表和作为协作者的列表，合并返回。

#### 3. 协作者视图的隐私保护

**文件：** `lists.ts:48-71`

非列表所有者访问时，返回数据经过脱敏处理：

```typescript
// lists.ts:48-71
asZBookmarkList() {
  if (this.list.userId === this.ctx.user.id) {
    return this.list;
  }
  return {
    id: this.list.id,
    name: this.list.name,
    description: this.list.description,
    userId: this.list.userId,
    icon: this.list.icon,
    type: this.list.type,
    query: this.list.query,
    userRole: this.list.userRole,
    hasCollaborators: this.list.hasCollaborators,
    parentId: null,        // 隐藏
    public: false,         // 隐藏
  };
}
```

#### 4. 协作者信息查询的隐私控制

**文件：** `lists.ts:794-862`

`getCollaborators` 方法根据访问者身份控制信息可见性：

| 信息 | 所有者可见 | 协作者可见 |
|------|-----------|-----------|
| 用户姓名 | ✅ | ✅ |
| 用户邮箱 | ✅ | ❌ |
| 用户头像 | ✅ | ✅ |
| 待处理邀请列表 | ✅ | ❌ |

待处理邀请的用户信息被掩码为 "Pending User" 以保护隐私。

#### 5. 前端缓存失效

**文件：** `ManageCollaboratorsModal.tsx:68-83`

协作状态变更时，同时失效多个查询缓存：

```typescript
// ManageCollaboratorsModal.tsx:68-83
const invalidateListCaches = () =>
  Promise.all([
    queryClient.invalidateQueries(api.lists.getCollaborators.queryFilter({ listId: list.id })),
    queryClient.invalidateQueries(api.lists.get.queryFilter({ listId: list.id })),
    queryClient.invalidateQueries(api.lists.list.pathFilter()),
    queryClient.invalidateQueries(api.bookmarks.getBookmarks.queryFilter({ listId: list.id })),
    queryClient.invalidateQueries(api.bookmarks.getBookmarks.infiniteQueryFilter({ listId: list.id })),
  ]);
```

确保所有相关视图都能获得最新数据。

---

## 完整流程图

```
邀请创建阶段
┌─────────────────────────────────────────────────────────────┐
│ 列表所有者                                                   │
│  ┌─ ManageCollaboratorsModal ─┐                              │
│  │ 输入邮箱+选择角色          │                              │
│  │  addCollaborator.mutate() │  ┌────────────────────────┐  │
│  └────────────────────────────┘→│ lists.addCollaborator │  │
│                                 └───────────┬────────────┘  │
│                                             │ 限流+权限校验 │
│                                 ┌───────────▼────────────┐  │
│                                 │ List.addCollaboratorBy│  │
│                                 │ Email()                │  │
│                                 └───────────┬────────────┘  │
│                                             │                │
│                                 ┌───────────▼────────────┐  │
│                                 │ ListInvitation.        │  │
│                                 │ inviteByEmail()        │  │
│                                 │ - 用户存在校验         │  │
│                                 │ - 非所有者校验         │  │
│                                 │ - 非协作者校验         │  │
│                                 │ - 非待处理邀请校验     │  │
│                                 └───────────┬────────────┘  │
│                                             │                │
│                                 ┌───────────▼────────────┐  │
│                                 │ 插入 listInvitations   │  │
│                                 └───────────┬────────────┘  │
│                                             │                │
│                                 ┌───────────▼────────────┐  │
│                                 │ sendListInvitationEmail│  │
│                                 └────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

接收确认阶段
┌─────────────────────────────────────────────────────────────┐
│ 被邀请用户                                                   │
│  ┌─ 邮件通知 ─┐                                              │
│  │ 点击链接  │                                              │
│  └─────┬──────┘                                              │
│        │                                                     │
│  ┌─────▼───────────────────────────────────┐                │
│  │ /dashboard/lists?pendingInvitation=xxx  │                │
│  └─────┬───────────────────────────────────┘                │
│        │                                                     │
│  ┌─────▼──────────────────────────────┐                     │
│  │ InvitationNotificationBadge        │ 每5分钟轮询        │
│  │ - 显示待处理邀请数量                │                     │
│  └─────┬──────────────────────────────┘                     │
│        │                                                     │
│  ┌─────▼──────────────────────────────┐                     │
│  │ PendingInvitationsCard             │                     │
│  │ - 显示邀请详情                      │                     │
│  │ - acceptInvitation.mutate()        │                     │
│  │ - declineInvitation.mutate()       │                     │
│  └─────┬──────────────────────────────┘                     │
│        │                                                     │
│  ┌─────▼──────────────────────────────┐                     │
│  │ lists.acceptInvitation             │                     │
│  └─────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────┘

权限落地阶段
┌─────────────────────────────────────────────────────────────┐
│ 服务端                                                       │
│  ┌─ ensureInvitationAccess ─┐                                │
│  │ - 加载邀请               │                                │
│  │ - 校验被邀请者身份       │                                │
│  └─────────────┬────────────┘                                │
│                │                                             │
│  ┌─────────────▼────────────┐                                │
│  │ ListInvitation.accept()  │                                │
│  │ ┌──────────────────────┐ │                                │
│  │ │  DB Transaction      │ │                                │
│  │ │ 1. DELETE listInvit- │ │                                │
│  │ │    ations            │ │                                │
│  │ │ 2. INSERT listCollab-│ │                                │
│  │ │    orators           │ │                                │
│  │ └──────────────────────┘ │                                │
│  └──────────────────────────┘                                │
└─────────────────────────────────────────────────────────────┘

视图更新阶段
┌─────────────────────────────────────────────────────────────┐
│ 被邀请用户                                                   │
│  ┌─ 缓存失效 ───────────────────────────────────┐           │
│  │ - lists.getPendingInvitations                │           │
│  │ - lists.list                                 │           │
│  │ - lists.get                                  │           │
│  │ - bookmarks.getBookmarks                     │           │
│  └─────────────┬────────────────────────────────┘           │
│                │                                             │
│  ┌─────────────▼────────────┐                                │
│  │ List.getAll()            │                                │
│  │ - getAllOwned()          │                                │
│  │ - getSharedWithUser()    │                                │
│  └─────────────┬────────────┘                                │
│                │                                             │
│  ┌─────────────▼────────────┐                                │
│  │ List.fromId()            │                                │
│  │ - 检查是否为所有者       │                                │
│  │ - 检查协作者身份         │                                │
│  │ - 设置 userRole          │                                │
│  └─────────────┬────────────┘                                │
│                │                                             │
│  ┌─────────────▼────────────┐                                │
│  │ 协作列表出现在用户侧边栏 │                                │
│  │ - 带有协作者标记         │                                │
│  │ - 根据角色显示操作按钮   │                                │
│  └──────────────────────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

## 关键设计要点

### 1. 数据一致性
- 邀请接受采用数据库事务，确保邀请删除和协作者添加的原子性
- 使用 `onConflictDoNothing` 处理幂等性

### 2. 隐私保护
- 待处理邀请用户信息掩码为 "Pending User"
- 非列表所有者无法查看协作者邮箱
- 共享列表隐藏 `parentId` 和 `public` 等敏感字段

### 3. 权限控制
- 三级权限模型（owner/editor/viewer）
- 中间件统一权限校验
- 只有手动列表支持协作

### 4. 用户体验
- 侧边栏徽章实时提示待处理邀请
- 5 分钟自动刷新
- 细粒度的缓存失效策略
- 完整的邮件通知链路

### 5. 限流与安全
- 邀请创建限流（15 分钟 20 次）
- 邀请令牌使用一次性机制
- 严格的输入校验（邮箱格式、角色枚举等）

## 代码溯源

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 邀请数据表 | `packages/db/schema.ts` | 562-593 |
| 协作者数据表 | `packages/db/schema.ts` | 536-560 |
| 邀请 API 路由 | `packages/trpc/routers/lists.ts` | 250-414 |
| 邀请业务模型 | `packages/trpc/models/listInvitations.ts` | 全文件 |
| 列表业务模型 | `packages/trpc/models/lists.ts` | 689-895 |
| 邮件发送 | `packages/trpc/email.ts` | 194-241 |
| 管理协作者 UI | `apps/web/components/dashboard/lists/ManageCollaboratorsModal.tsx` | 全文件 |
| 待处理邀请 UI | `apps/web/components/dashboard/lists/PendingInvitationsCard.tsx` | 全文件 |
| 通知徽章 UI | `apps/web/components/dashboard/sidebar/InvitationNotificationBadge.tsx` | 全文件 |
