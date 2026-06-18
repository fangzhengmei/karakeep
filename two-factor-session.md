# 双因素认证与设备会话管理数据流分析

## 概述

本文档分析 Karakeep 项目中与认证安全相关的数据流。经过代码审计，**当前版本尚未实现 TOTP 双因素认证、恢复码（Backup Codes）和设备会话列表功能**。项目现有的认证安全体系包括：密码认证、JWT 会话、API Key（作为设备/客户端接入点）、邮箱验证、密码重置、账户有效性校验等。

---

## 一、数据库层 — 状态存储

### 1.1 用户表 `users`

定义位置：[schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/db/schema.ts#L32-L101)

| 字段 | 说明 | 与安全相关的用途 |
|------|------|-----------------|
| `password` | bcrypt 哈希后的密码 | 本地账号密码校验 |
| `salt` | 每个用户独立的密码盐值 | 加强密码哈希安全性 |
| `emailVerified` | 邮箱验证时间戳 | 登录时检查是否已验证邮箱 |
| `role` | `admin` / `user` | 权限分级 |

> ⚠️ **当前缺少的字段**：`twoFactorEnabled`、`totpSecret`、`recoveryCodes`、`twoFactorBackupCodes` 等 2FA 相关字段均未在 users 表中定义。

### 1.2 会话表 `sessions`

定义位置：[schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/db/schema.ts#L127-L136)

```typescript
export const sessions = sqliteTable("session", {
  sessionToken: text("sessionToken").notNull().primaryKey(),
  userId: text("userId").notNull().references(() => users.id, { onDelete: "cascade" }),
  expires: integer("expires", { mode: "timestamp_ms" }).notNull(),
});
```

> ⚠️ **注意**：虽然存在 `sessions` 表，但 [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/server/auth.ts#L181-L183) 中 `session.strategy` 配置为 `"jwt"`，意味着 **Web 端实际上不使用数据库会话表**，所有会话状态存储在 JWT Token 中（HttpOnly Cookie）。该表目前仅作为 NextAuth DrizzleAdapter 的兼容存在。

### 1.3 API Key 表 `apiKeys`

定义位置：[schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/db/schema.ts#L165-L186)

| 字段 | 说明 |
|------|------|
| `name` | Key 名称（用户自定义，用于识别设备/用途） |
| `keyId` | Key 公开标识（用于查询） |
| `keyHash` | Key 密钥的哈希值（SHA256 或 bcrypt） |
| `scopes` | 权限范围（JSON 数组） |
| `lastUsedAt` | 最近使用时间（10 分钟节流更新） |
| `createdAt` | 创建时间 |

> API Key 表目前承担了"设备接入凭证"的角色，浏览器扩展和移动端通过 API Key 接入服务，每个 Key 可视为一个设备会话。

### 1.4 邮箱验证 Token 表 `verificationTokens`

定义位置：[schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/db/schema.ts#L138-L146)

用于新用户邮箱验证和 NextAuth 内部流程。

### 1.5 密码重置 Token 表 `passwordResetTokens`

定义位置：[schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/db/schema.ts#L148-L163)

| 字段 | 说明 |
|------|------|
| `userId` | 关联用户 |
| `token` | 重置 Token（唯一） |
| `expires` | 过期时间（1 小时） |
| `createdAt` | 创建时间 |

---

## 二、TOTP / 恢复码（尚未实现）

### 2.1 现状说明

经过全代码库搜索，以下关键词均无匹配：
- `totp` / `TOTP`
- `twoFactor` / `two_factor` / `2fa` / `2FA`
- `recovery.*code` / `backup.*code`

数据库 schema、trpc 路由、前端设置页面均无 2FA 相关实现。

### 2.2 如要实现的建议接入点

根据现有架构，2FA 功能的建议接入位置：

| 层级 | 建议文件 | 新增内容 |
|------|---------|---------|
| 数据库 | [schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/db/schema.ts) | `users` 表增加 `totpSecret`、`twoFactorEnabled`；新增 `recoveryCodes` 表 |
| 认证核心 | [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/auth.ts) | 新增 `validateTotpCode()`、`generateRecoveryCodes()` 函数 |
| tRPC 路由 | [users.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/routers/users.ts) | 新增 `enableTwoFactor`、`disableTwoFactor`、`verifyTwoFactor`、`regenerateRecoveryCodes` 过程 |
| 登录流程 | [auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/server/auth.ts#L191-L252) | `signIn` callback 中增加 2FA 状态检查，未验证时不签发完整 JWT |
| 前端设置页 | [info/page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/settings/info/page.tsx) | 新增 TwoFactorSection 组件，包含二维码、输入验证、恢复码展示 |
| 登录页 | [CredentialsForm.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/signin/CredentialsForm.tsx) | 密码校验通过后，跳转到 2FA 验证码输入步骤 |

---

## 三、现有认证数据流

### 3.1 密码登录流程（Web 端）

```
用户输入邮箱+密码
    ↓
[CredentialsForm.tsx]  signIn("credentials", ...)
    ↓
[auth.ts:NextAuth authorize]  validatePassword(email, password, db)
    ↓
[auth.ts:validatePassword]  bcrypt.compare(password + salt, user.password)
    │  ├─ 失败 → 执行 dummy bcrypt 比较（防时序攻击）→ 抛出错误
    │  └─ 成功 → 返回 user 对象
    ↓
[auth.ts:signIn callback]
    │  ├─ 检查 emailVerified（若配置要求）
    │  ├─ 记录日志 user.login / user.login_failed
    │  └─ 返回 true / 抛出错误
    ↓
[auth.ts:jwt callback]  将 {id, name, email, image, role} 写入 JWT payload
    ↓
[auth.ts:session callback]  从 JWT 提取用户信息到 session 对象
    ↓
浏览器收到 HttpOnly Cookie（含 JWT）
```

关键代码：
- 前端登录表单：[CredentialsForm.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/signin/CredentialsForm.tsx#L76-L95)
- NextAuth 配置：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/server/auth.ts#L177-L270)
- 密码校验：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/auth.ts#L166-L201)

### 3.2 OAuth 登录流程

```
用户点击 OAuth Provider 按钮
    ↓
[SignInProviderButton.tsx]  → NextAuth signIn(providerId)
    ↓
OAuth 服务商回调 → NextAuth 处理
    ↓
[auth.ts:CustomProvider.createUser]  首次登录自动创建用户
    │  └─ 第一个注册用户自动获得 admin 角色
    ↓
[auth.ts:signIn callback]
    │  ├─ 新用户且禁用注册时 → 拒绝
    │  └─ 记录 user.login 事件
    ↓
签发 JWT Cookie（同密码登录）
```

关键代码：
- OAuth Profile 处理：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/server/auth.ts#L161-L174)
- 自动建用户：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/server/auth.ts#L98-L110)
- 首用户 admin 判定：[users.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/models/users.ts#L106-L112)

### 3.3 API Key 交换流程（浏览器扩展 / CLI / 移动端）

这是非 Web 客户端获取认证凭证的方式。

```
用户在扩展/客户端输入 邮箱+密码
    ↓
调用 apiKeys.exchange(email, password, keyName)
    ↓
[apiKeys.ts:exchange]  validatePassword() → 校验密码
    │  ├─ 检查 emailVerificationRequired
    │  └─ 速率限制：15 分钟 10 次
    ↓
[auth.ts:generateApiKey]
    │  ├─ 生成 keyId (10 bytes hex) + secret (16 bytes hex)
    │  ├─ 计算 secretHash = SHA256(secret).base64
    │  └─ 存入 apiKeys 表（仅存哈希，不存明文）
    ↓
返回明文 Key：ak2_{keyId}_{secret}
    ↓
客户端存储 apiKey 和 apiKeyId
```

关键代码：
- 交换路由：[apiKeys.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/routers/apiKeys.ts#L134-L194)
- Key 生成：[auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/auth.ts#L52-L82)

---

## 四、会话列表与设备管理（基于 API Key）

虽然项目没有传统意义上的"设备会话列表"，但 API Key 列表承担了类似功能。

### 4.1 列表查询数据流

```
用户访问 /settings/api-keys
    ↓
[api-keys/page.tsx]  服务端调用 api.apiKeys.list()
    ↓
[apiKeys.ts:list]  sessionProcedure → 仅允许会话认证（禁止 API Key 调 API Key 列表）
    ↓
查询 apiKeys 表：按 userId 过滤，按 createdAt 倒序
    │  返回字段：id, name, createdAt, lastUsedAt, keyId, scopes
    ↓
[ApiKeySettings.tsx]  渲染为表格：名称 | KeyID | 权限范围 | 创建时间 | 最近使用 | 操作
```

关键代码：
- 列表路由：[apiKeys.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/routers/apiKeys.ts#L102-L131)
- 前端渲染：[ApiKeySettings.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/settings/ApiKeySettings.tsx#L19-L78)
- sessionProcedure 定义（拒绝 API Key 访问敏感端点）：[index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/index.ts#L197)

### 4.2 API Key 创建数据流

```
用户点击"新建 API Key" → 填写名称 + 选择权限范围
    ↓
[AddApiKey.tsx:AddApiKeyForm]  提交 name + scopes
    ↓
[apiKeys.ts:create]  sessionProcedure（必须是 Web 会话认证）
    ↓
[auth.ts:generateApiKey]  生成 ak2_{keyId}_{secret}，存 keyHash 到 DB
    ↓
返回明文 Key 给前端 → [ApiKeySuccess.tsx] 展示（仅此一次可见）
```

关键代码：
- 创建表单：[AddApiKey.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/settings/AddApiKey.tsx#L159-L359)
- 创建路由：[apiKeys.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/routers/apiKeys.ts#L36-L60)

### 4.3 API Key 重新生成（轮换）

```
用户点击"重新生成"
    ↓
[apiKeys.ts:regenerate]  验证所有权（userId 匹配）
    ↓
[auth.ts:regenerateApiKey]  生成新 keyId + secret，更新 keyHash
    ↓
旧 Key 立即失效，返回新明文 Key
```

关键代码：
- 重生成路由：[apiKeys.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/routers/apiKeys.ts#L61-L88)

---

## 五、注销下线数据流

### 5.1 Web 端注销

```
用户点击 Profile → Sign Out
    ↓
路由跳转 /logout
    ↓
[logout/page.tsx]  useEffect 中执行：
    │  ├─ signOut({ redirect: false }) → 清除 NextAuth JWT Cookie
    │  ├─ clearHistory() → 清除 localStorage 中的搜索历史
    │  └─ router.push("/") → 跳转首页
```

关键代码：
- 注销入口：[ProfileOptions.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/dashboard/header/ProfileOptions.tsx#L144-L147)
- 注销页面：[logout/page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/logout/page.tsx#L9-L26)

> ⚠️ **局限**：由于使用 JWT 策略，JWT 本身是无状态的。服务端注销仅清除浏览器 Cookie，但已签发的 JWT 在过期前仍然有效（如果被窃取）。数据库 `sessions` 表未被实际使用，无法做服务端主动失效。

### 5.2 浏览器扩展注销

```
用户在扩展 Options 页面点击 Logout
    ↓
[OptionsPage.tsx:onLogout]
    │  ├─ 若存在 apiKeyId → 调用 api.apiKeys.revoke({ id }) → 从 DB 删除该 Key
    │  ├─ setSettings 清除本地存储的 apiKey 和 apiKeyId
    │  └─ navigate("/notconfigured")
```

关键代码：
- 扩展注销：[OptionsPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/browser-extension/src/OptionsPage.tsx#L102-L109)
- 删除 Key 路由：[apiKeys.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/routers/apiKeys.ts#L89-L101)

### 5.3 移动端注销

```
调用 useSession().logout()
    ↓
[session.ts:logout]
    │  ├─ 若存在 settings.apiKeyId → 调用 api.apiKeys.revoke()
    │  └─ 清除本地 settings 中的 apiKey 和 apiKeyId
```

关键代码：
- 移动端会话 Hook：[session.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/mobile/lib/session.ts#L8-L26)

### 5.4 远程强制下线（撤销 API Key）

Web 用户可以在设置页撤销任意 API Key，实现对扩展/移动设备的远程下线：

```
用户在 /settings/api-keys 点击删除
    ↓
[DeleteApiKey.tsx]  → api.apiKeys.revoke.mutate({ id })
    ↓
[apiKeys.ts:revoke]  sessionProcedure + 校验 userId 所有权
    ↓
DELETE FROM apiKeys WHERE id = ? AND userId = ?
    ↓
对应设备下次请求时 authenticateApiKey 失败 → 设备显示"未登录"
```

---

## 六、安全提示与账户有效性校验

### 6.1 ValidAccountCheck — 前台存活探针

这是一个关键的安全组件，解决 **"JWT 仍有效但用户已被删除/禁用"** 的不一致问题。

挂载位置：
- [SidebarLayout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/shared/sidebar/SidebarLayout.tsx)（通过全局布局引入）

数据流：
```
页面加载
    ↓
[ValidAccountCheck.tsx]  useQuery(api.users.whoami)
    │  ├─ 若返回 UNAUTHORIZED → 不重试
    │  └─ 其他错误正常重试
    ↓
useEffect 监听 error
    ├─ 若 error.code === "UNAUTHORIZED"
    │    └─ router.push("/logout") → 触发完整注销流程
    └─ 否则无操作
```

关键代码：
- [ValidAccountCheck.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/utils/ValidAccountCheck.tsx#L13-L33)

### 6.2 服务端布局层二次校验

在所有需要认证的 Server Component 布局中，除了 `getServerAuthSession()` 检查 JWT 有效性外，还会调用 `api.users.settings()` 做二次校验：

```typescript
// dashboard/layout.tsx, reader/layout.tsx, settings/layout.tsx 中均存在以下模式
const session = await getServerAuthSession();
if (!session) redirect("/");

const userSettings = await tryCatch(api.users.settings());
if (userSettings.error) {
  if (userSettings.error instanceof TRPCError) {
    if (error.code === "NOT_FOUND" || error.code === "UNAUTHORIZED") {
      redirect("/logout");
    }
  }
  throw userSettings.error;
}
```

涉及文件：
- [dashboard/layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/dashboard/layout.tsx#L32-L52)
- [reader/layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/reader/layout.tsx#L15-L32)
- [settings/layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/settings/layout.tsx#L119-L136)

### 6.3 tRPC 认证中间件

定义位置：[index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/index.ts#L131-L152)

```typescript
export const authedProcedure = procedure
  .use(createRateLimitMiddleware({ ... }))  // 已认证用户：60秒 3000次
  .use(function isAuthed(opts) {
    if (!opts.ctx.user?.id) {
      throw new TRPCError({ code: "UNAUTHORIZED" });
    }
    return opts.next({ ctx: { user } });
  });
```

Context 中的 `user` 来源：
- Web 端：由 NextAuth JWT 解码后注入
- API 端：由 [auth.ts:authenticateApiKey()](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/auth.ts#L107-L160) 解析 API Key 后注入

Hono API 层进一步封装：
- [api/middlewares/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/api/middlewares/auth.ts#L24-L37) — `authMiddleware` 校验 `ctx.user` 存在
- [api/middlewares/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/api/middlewares/auth.ts#L39-L59) — `adminAuthMiddleware` 额外校验 `role === "admin"`

### 6.4 密码修改安全

数据流：
```
用户在 /settings/info 填写 当前密码 + 新密码
    ↓
[ChangePassword.tsx]  提交 → api.users.changePassword.mutate()
    ↓
[users.ts:changePassword]  速率限制：15 分钟 5 次
    ↓
[User.changePassword]
    │  ├─ validatePassword(email, currentPassword, db) → 验证旧密码
    │  ├─ 生成新 salt (generatePasswordSalt)
    │  └─ bcrypt.hash(newPassword + newSalt) → 更新 password 和 salt 字段
    ↓
前端 toast 提示成功，表单重置
```

关键代码：
- 修改密码表单：[ChangePassword.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/settings/ChangePassword.tsx#L28-L209)
- 修改密码模型方法：[users.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/models/users.ts#L440-L465)

### 6.5 邮箱验证强制登录拦截

在 `signIn` callback 中：

```typescript
// auth.ts
if (serverConfig.auth.emailVerificationRequired && !user.emailVerified) {
  throw new Error("Please verify your email address before signing in");
}
```

前端收到此错误时，跳转到 `/check-email` 页面：
- [CredentialsForm.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/signin/CredentialsForm.tsx#L85-L88)

---

## 七、整体数据流架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                         浏览器 (Web 端)                            │
│  ┌──────────────┐   JWT Cookie    ┌────────────────────────────┐  │
│  │ Credentials  │ ──────────────► │ NextAuth (/api/auth/*)     │  │
│  │ OAuth Button │                 │  authorize → signIn → jwt │  │
│  └──────────────┘                 └─────────────┬──────────────┘  │
│                                                  │                 │
│  ┌──────────────────────────────────────────────▼──────────────┐  │
│  │           ValidAccountCheck (全局存活探针)                    │  │
│  │  whoami 查询 → UNAUTHORIZED → /logout                       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                  │                 │
│  ┌──────────────┐  tRPC (fetch)  ┌──────────────▼──────────────┐  │
│  │ Page/Component │ ───────────► │  authedProcedure            │  │
│  │ (dashboard/   │                │  sessionProcedure           │  │
│  │  settings)    │                │  createScopedAuthedProcedure│  │
│  └──────────────┘                └──────────────┬──────────────┘  │
└──────────────────────────────────────────────────┼─────────────────┘
                                                   │
┌──────────────────────────────────────────────────┼─────────────────┐
│                浏览器扩展 / 移动端                │                 │
│  ┌──────────────┐  apiKeys.exchange  ┌──────────▼──────────┐     │
│  │  登录表单     │ ─────────────────► │  邮箱+密码 → API Key│     │
│  └──────────────┘                    └──────────┬──────────┘     │
│                                     存储 apiKey + apiKeyId       │
│  ┌──────────────┐  tRPC + API Key  ┌──────────▼──────────┐     │
│  │ 业务请求      │ ────────────────► │ authenticateApiKey │     │
│  └──────────────┘                   └──────────┬──────────┘     │
└─────────────────────────────────────────────────┼────────────────┘
                                                  │
                        ┌─────────────────────────▼─────────────────┐
                        │           数据库 (SQLite)                 │
                        │  users | sessions | apiKeys               │
                        │  verificationTokens | passwordResetTokens │
                        └───────────────────────────────────────────┘
```

---

## 八、实现 2FA 后需要补充的数据流节点

如果后续实现 TOTP + 恢复码 + 设备会话列表，需要在以下节点插入逻辑：

| 现有节点 | 需要扩展的逻辑 |
|---------|--------------|
| `users.create` / OAuth 首次登录 | 默认为 `twoFactorEnabled = false` |
| `validatePassword()` | 密码正确后，若 2FA 已启用则返回 `requiresTwoFactor` 状态而非直接返回 user |
| NextAuth `authorize` | 支持两步认证：先密码，再 TOTP/恢复码 |
| JWT `jwt callback` | 增加 `twoFactorVerified` 字段；敏感操作（修改密码、撤销 Key）需检查此字段 |
| `apiKeys.exchange` | 同登录流程，2FA 用户需额外提交验证码 |
| API Key 列表页 | 新增"当前会话"标记、设备信息（User-Agent、IP）、下线按钮 |
| `users.deleteAccount` | 要求输入 TOTP 验证码（若已启用） |
| `ValidAccountCheck` | 可扩展检查 2FA 状态变更后的重新验证 |

---

## 九、涉及文件索引

| 文件路径 | 角色 |
|---------|------|
| [packages/db/schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/db/schema.ts) | 数据库表定义 |
| [apps/web/server/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/server/auth.ts) | NextAuth 配置（Web 端登录核心） |
| [apps/web/lib/auth/client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/lib/auth/client.ts) | 客户端认证导出层 |
| [packages/trpc/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/auth.ts) | 密码校验、API Key 生成与认证 |
| [packages/trpc/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/index.ts) | tRPC 过程定义（authedProcedure 等） |
| [packages/trpc/routers/users.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/routers/users.ts) | 用户相关路由（注册、改密、注销等） |
| [packages/trpc/routers/apiKeys.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/routers/apiKeys.ts) | API Key 路由（设备凭证管理） |
| [packages/trpc/models/users.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/trpc/models/users.ts) | User 模型（改密、邮箱验证、重置密码） |
| [packages/api/middlewares/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/packages/api/middlewares/auth.ts) | Hono API 认证中间件 |
| [apps/web/app/logout/page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/logout/page.tsx) | Web 端注销页面 |
| [apps/web/components/utils/ValidAccountCheck.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/utils/ValidAccountCheck.tsx) | 账户有效性校验 |
| [apps/web/components/dashboard/header/ProfileOptions.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/dashboard/header/ProfileOptions.tsx) | 用户菜单（含注销入口） |
| [apps/web/components/signin/CredentialsForm.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/signin/CredentialsForm.tsx) | 密码登录表单 |
| [apps/web/components/signin/SignInForm.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/signin/SignInForm.tsx) | 登录页组合组件 |
| [apps/web/components/settings/ChangePassword.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/settings/ChangePassword.tsx) | 修改密码表单 |
| [apps/web/components/settings/ApiKeySettings.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/settings/ApiKeySettings.tsx) | API Key 列表（设备列表） |
| [apps/web/components/settings/AddApiKey.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/components/settings/AddApiKey.tsx) | 新建 API Key |
| [apps/web/app/dashboard/layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/dashboard/layout.tsx) | Dashboard 布局（含二次认证校验） |
| [apps/web/app/reader/layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/reader/layout.tsx) | Reader 布局（含二次认证校验） |
| [apps/web/app/settings/layout.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/web/app/settings/layout.tsx) | Settings 布局（含二次认证校验） |
| [apps/browser-extension/src/OptionsPage.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/browser-extension/src/OptionsPage.tsx) | 浏览器扩展设置（含注销） |
| [apps/mobile/lib/session.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/44-karakeep/apps/mobile/lib/session.ts) | 移动端会话管理 |
