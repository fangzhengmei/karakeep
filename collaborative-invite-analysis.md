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

> **重要澄清：** `pendingInvitation` 参数在当前前端代码中**未被任何组件消费**。该参数仅作为 URL 的一部分存在，但实际的待处理邀请展示完全不依赖此参数。详情见下文"参数与入口的真实对应关系"。

---

### 阶段二：接收确认

#### 0. 邮件链接参数与前端入口的真实对应关系

**实际链路：**

```
邮件链接: /dashboard/lists?pendingInvitation={listId}
              │
              ▼
       跳转到列表页面 (page.tsx)
              │
              ▼  无参数消费逻辑
       PendingInvitationsCard 组件
              │
              ▼  调用
       lists.getPendingInvitations() 查询
              │
              ▼
       返回当前用户的所有待处理邀请
```

**关键发现：**

| 项目 | 状态 | 说明 |
|------|------|------|
| `pendingInvitation` URL 参数 | ❌ 未消费 | 代码库中无任何地方读取或使用该参数 |
| 实际展示依赖 | ✅ `lists.getPendingInvitations` 查询 | 基于当前登录用户 ID 返回所有待处理邀请 |
| 参数设计意图 | 可能为预留功能 | 理论上可用于高亮特定邀请，但当前未实现 |

**前端入口代码溯源：**

1. **列表页面** (`apps/web/app/dashboard/lists/page.tsx:36`)：直接渲染 `<PendingInvitationsCard />`，不读取任何 searchParams
2. **待处理邀请卡片** (`PendingInvitationsCard.tsx:144-146`)：调用 `api.lists.getPendingInvitations.queryOptions()` 获取所有待处理邀请
3. **侧边栏徽章** (`InvitationNotificationBadge.tsx:9-13`)：同样调用 `getPendingInvitations` 查询，每 5 分钟刷新

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

### 6. 邮件链接参数的设计与现状
- `pendingInvitation` 参数已预置但未实际消费
- 实际展示依赖 `lists.getPendingInvitations` 查询
- 参数仅起到跳转到正确页面的作用，无业务逻辑依赖

---

## 边界行为与用户感知偏差说明

### 核心边界场景分析（结合代码事实）

---

#### 场景一：链接携带 listId 但页面展示全部待处理邀请

**触发条件：**
- 用户点击邮件中的链接：`/dashboard/lists?pendingInvitation={listId}`
- 该用户同时有多个待处理邀请

**代码事实：**
1. 邮件链接生成（`email.ts:203`）：
   ```typescript
   const inviteUrl = `${serverConfig.publicUrl}/dashboard/lists?pendingInvitation=${encodeURIComponent(listId)}`;
   ```
2. 列表页面（`page.tsx:36`）：直接渲染 `<PendingInvitationsCard />`，**不读取任何 searchParams**
3. 待处理邀请卡片（`PendingInvitationsCard.tsx:144-146`）：
   ```typescript
   const { data: invitations, isLoading } = useQuery(
     api.lists.getPendingInvitations.queryOptions(),
   );
   ```
4. 后端查询（`listInvitations.ts:296-300`）：
   ```sql
   WHERE listInvitations.userId = currentUserId 
     AND listInvitations.status = 'pending'
   ```
   **不包含**对 `listId` 的过滤条件

**页面表现：**
- 页面顶部展示该用户的**所有**待处理邀请卡片
- 邮件中提到的特定列表邀请不会被高亮、排序到顶部或有任何特殊标记
- 如果用户有 N 个待处理邀请，全部都会显示

**潜在误导点：**
- 用户期望："我点击了这个列表的邀请链接，应该只看到这个列表的邀请"
- 实际体验：需要在多个邀请中自行寻找邮件中提到的那个
- 感知偏差：用户可能疑惑"为什么给我看其他邀请？"

---

#### 场景二：pendingInvitation 无效或过期时无任何提示

**触发条件：**
- 链接中的 `listId` 对应的邀请已被接受
- 链接中的 `listId` 对应的邀请已被拒绝
- 链接中的 `listId` 对应的邀请已被所有者撤销
- 链接中的 `listId` 对应的列表已被删除

**代码事实：**
1. `pendingInvitation` 参数**完全未被消费**，前端不会校验该参数的有效性
2. `PendingInvitationsCard.tsx:152-154`：
   ```typescript
   if (!invitations || invitations.length === 0) {
     return null; // 无待处理邀请时直接不渲染
   }
   ```
3. 后端 `getPendingInvitations` 只返回状态为 `pending` 的邀请

**页面表现：**
- 页面静默跳转到 `/dashboard/lists`
- 如果用户当前没有其他待处理邀请：**页面上不会出现任何与邀请相关的元素**，就像什么都没发生过
- 如果用户有其他待处理邀请：只显示那些仍有效的邀请，不会提示"你点击的那个邀请已失效"
- 没有任何错误提示、警告信息或 toast 通知

**潜在误导点：**
- 用户可能以为链接无效、系统出问题，或者自己"错过了"邀请
- 无法区分"邀请已被处理"和"链接根本没生效"两种情况
- 用户可能反复点击链接，困惑为什么没有反应

---

#### 场景三：非被邀请用户打开链接时的可见结果

**触发条件：**
- 邀请发送到用户 A 的邮箱（链接中的 listId 对应的邀请属于用户 A）
- 当前登录的是用户 B（不是被邀请者）
- 或用户使用了与邀请邮箱不同的账号登录

**代码事实：**
1. 邀请创建时（`listInvitations.ts:198-207`）：通过邮箱查询用户 ID 并绑定到邀请记录
   ```typescript
   const user = await ctx.db.query.users.findFirst({
     where: eq(users.email, email),
   });
   if (!user) { /* 抛出 NOT_FOUND 错误 */ }
   ```
2. 待处理邀请查询（`listInvitations.ts:297-299`）：
   ```sql
   WHERE listInvitations.userId = currentUserId 
     AND listInvitations.status = 'pending'
   ```
3. `page.tsx` 不读取 URL 参数，不做任何跨用户校验

**页面表现：**
- 用户 B 打开链接后，页面只显示**用户 B 自己的**待处理邀请（如果有的话）
- 不会出现任何提示告知"这个邀请不是发给你的"
- 如果用户 B 没有待处理邀请：页面无任何邀请相关内容，静默失败
- 被邀请者用户 A 登录后，在自己的待处理邀请列表中能看到该邀请

**潜在误导点：**
- 用户可能以为"邀请链接失效了"，但实际上是登录错了账号
- 多人共用设备时容易出现这种混淆
- 没有任何机制提示用户"你需要使用 xxx@example.com 邮箱登录才能看到此邀请"

---

### 其他边界场景

#### 4. 未登录用户点击邀请链接
**现象：** 未登录用户点击邮件邀请链接，会被重定向到登录页面，登录后可能不会自动跳回列表页面。

**原因：** 邀请链接没有携带 `callbackUrl` 参数，NextAuth 的默认登录流程可能丢失原始目标路径。

#### 5. 拒绝后重新邀请的状态变化
**现象：** 用户拒绝邀请后，所有者可以重新发送邀请。此时 `listInvitations` 记录不会重新创建，而是将 `status` 从 "declined" 更新为 "pending"。

**潜在问题：** 邀请的 `invitedAt` 字段会被更新为最新时间，用户无法看到最初邀请的时间。

#### 6. 协作者被移除后的数据处理（原说法修正）

> **原说法不准确修正：** 原描述"协作者被移除后其添加的书签仍会保留在列表中"不符合实际代码行为。

**真实行为：协作者被移除后，他们添加的书签会被级联删除**

**代码事实链：**

1. **协作者添加书签时**（`lists.ts:1024-1028`）：
   ```typescript
   await this.ctx.db.insert(bookmarksInLists).values({
     listId: this.list.id,
     bookmarkId,
     listMembershipId: this.collaboratorEntry?.membershipId,
   });
   ```
   - 协作者添加的书签会记录 `listMembershipId`，指向该协作者的 `listCollaborators` 记录
   - 列表所有者添加的书签不会设置此字段（`collaboratorEntry` 为 null）

2. **数据库级联约束**（`schema.ts:517-521`）：
   ```typescript
   listMembershipId: text("listMembershipId").references(
     () => listCollaborators.id,
     {
       onDelete: "cascade",  // 关键：级联删除
     },
   ),
   ```

3. **移除协作者时**（`lists.ts:715-722`）：
   ```typescript
   await this.ctx.db
     .delete(listCollaborators)
     .where(
       and(
         eq(listCollaborators.listId, this.list.id),
         eq(listCollaborators.userId, userId),
       ),
     );
   ```
   - 只删除 `listCollaborators` 记录
   - 数据库通过 `ON DELETE CASCADE` 自动删除 `bookmarksInLists` 中关联的记录

**页面表现：**
- 协作者被移除后，列表中该协作者添加的所有书签都会消失
- 列表所有者添加的书签不受影响，继续保留

**用户感知差异：**
- **用户可能期望：** "移除协作者只是不让他们继续编辑，之前添加的内容应该保留"
- **实际行为：** 协作者添加的所有书签都会被移除，列表内容可能大幅减少
- **潜在困惑：** 所有者可能疑惑"那些书签去哪儿了？"（实际上书签本身仍存在于协作者的个人库中，只是从该列表中移除）

**与 leaveList 行为一致：**
`leaveList` 方法的注释（`lists.ts:735`）明确说明：
> "This also removes all bookmarks that the user added to the list."

这与级联删除的实际行为完全一致。

#### 7. 智能列表不支持协作
**现象：** 只有 `type = "manual"` 的列表可以邀请协作者，智能列表（`type = "smart"`）不支持协作。

**代码校验：** `listInvitations.ts:216-221` 明确检查列表类型。

#### 8. 邀请接受的幂等性
**现象：** 用户多次点击接受邀请按钮，第一次成功后第二次会报错。

**代码事实：** 事务中先删除邀请再插入协作者（`listInvitations.ts:122-136`），第二次点击时邀请已不存在，会抛出 "Invitation not found"。

#### 9. 列表删除后的邀请失效
**现象：** 列表被删除后，相关的邀请记录会通过外键 `ON DELETE CASCADE` 自动删除。

#### 10. 待处理邀请的隐私保护
**现象：** 列表所有者查看协作者列表时，待处理邀请的用户姓名显示为 "Pending User"，而不是真实姓名。

**代码事实：** `listInvitations.ts:373` 硬编码返回 "Pending User"。

---

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
