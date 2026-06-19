# 双因素认证与设备会话管理数据流分析

## 概述

本文档分析 Karakeep 项目中与认证安全相关的数据流。经过代码审计，**当前版本尚未实现 TOTP 双因素认证、恢复码（Backup Codes）和设备会话列表功能**。项目现有的认证安全体系包括：密码认证、JWT 会话、API Key（作为设备/客户端接入点）、邮箱验证、密码重置、账户有效性校验等。

---

## 一、数据库层 — 状态存储

### 1.1 用户表 `users`

定义位置：[schema.ts](./packages/db/schema.ts#L32-L101)

| 字段 | 说明 | 与安全相关的用途 |
|------|------|-----------------|
| `password` | bcrypt 哈希后的密码 | 本地账号密码校验 |
| `salt` | 每个用户独立的密码盐值 | 加强密码哈希安全性 |
| `emailVerified` | 邮箱验证时间戳 | 登录时检查是否已验证邮箱 |
| `role` | `admin` / `user` | 权限分级 |

> ⚠️ **当前缺少的字段**：`twoFactorEnabled`、`totpSecret`、`recoveryCodes`、`twoFactorBackupCodes` 等 2FA 相关字段均未在 users 表中定义。

### 1.2 会话表 `sessions`

定义位置：[schema.ts](./packages/db/schema.ts#L127-L136)

```typescript
export const sessions = sqliteTable("session", {
  sessionToken: text("sessionToken").notNull().primaryKey(),
  userId: text("userId").notNull().references(() => users.id, { onDelete: "cascade" }),
  expires: integer("expires", { mode: "timestamp_ms" }).notNull(),
});
```

> ⚠️ **注意**：虽然存在 `sessions` 表，但 [auth.ts](./apps/web/server/auth.ts#L181-L183) 中 `session.strategy` 配置为 `"jwt"`，意味着 **Web 端实际上不使用数据库会话表**，所有会话状态存储在 JWT Token 中（HttpOnly Cookie）。该表目前仅作为 NextAuth DrizzleAdapter 的兼容存在。

### 1.3 API Key 表 `apiKeys`

定义位置：[schema.ts](./packages/db/schema.ts#L165-L186)

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

定义位置：[schema.ts](./packages/db/schema.ts#L138-L146)

用于新用户邮箱验证和 NextAuth 内部流程。

### 1.5 密码重置 Token 表 `passwordResetTokens`

定义位置：[schema.ts](./packages/db/schema.ts#L148-L163)

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
| 数据库 | [schema.ts](./packages/db/schema.ts) | `users` 表增加 `totpSecret`、`twoFactorEnabled`；新增 `recoveryCodes` 表 |
| 认证核心 | [auth.ts](./packages/trpc/auth.ts) | 新增 `validateTotpCode()`、`generateRecoveryCodes()` 函数 |
| tRPC 路由 | [users.ts](./packages/trpc/routers/users.ts) | 新增 `enableTwoFactor`、`disableTwoFactor`、`verifyTwoFactor`、`regenerateRecoveryCodes` 过程 |
| 登录流程 | [auth.ts](./apps/web/server/auth.ts#L191-L252) | `signIn` callback 中增加 2FA 状态检查，未验证时不签发完整 JWT |
| 前端设置页 | [info/page.tsx](./apps/web/app/settings/info/page.tsx) | 新增 TwoFactorSection 组件，包含二维码、输入验证、恢复码展示 |
| 登录页 | [CredentialsForm.tsx](./apps/web/components/signin/CredentialsForm.tsx) | 密码校验通过后，跳转到 2FA 验证码输入步骤 |

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
- 前端登录表单：[CredentialsForm.tsx](./apps/web/components/signin/CredentialsForm.tsx#L76-L95)
- NextAuth 配置：[auth.ts](./apps/web/server/auth.ts#L177-L270)
- 密码校验：[auth.ts](./packages/trpc/auth.ts#L166-L201)

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
- OAuth Profile 处理：[auth.ts](./apps/web/server/auth.ts#L161-L174)
- 自动建用户：[auth.ts](./apps/web/server/auth.ts#L98-L110)
- 首用户 admin 判定：[users.ts](./packages/trpc/models/users.ts#L106-L112)

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
- 交换路由：[apiKeys.ts](./packages/trpc/routers/apiKeys.ts#L134-L194)
- Key 生成：[auth.ts](./packages/trpc/auth.ts#L52-L82)

### 3.4 Web JWT 会话 vs API Key 设备凭证 — 边界与差异深度对比

项目存在两种完全独立的认证凭证体系，它们在状态存储、认证链路、权限边界、可撤销性等方面存在本质差异。

#### 3.4.1 Context 构建：两条并行的认证注入链路

所有请求最终都会在 tRPC/Hono Context 中生成 `ctx.user` 和 `ctx.auth`，但来源完全不同：

```
                          入站 HTTP Request
                                │
            ┌───────────────────┴───────────────────┐
            │                                       │
  存在 Authorization: Bearer <key>?           走 Cookie 路径
            │                                       │
            ▼                                       ▼
  [client.ts:createContextFromRequest]    [client.ts:createContext]
   第 16-36 行：解析 Bearer Token           第 41-64 行：getServerAuthSession()
            │                                       │
            ▼                                       ▼
  authenticateApiKey(key, db)               NextAuth 解码 JWT Cookie
  [auth.ts:107-160]                         [auth.ts:257-270] jwt/session callback
            │                                       │
            └───────────────┬───────────────────────┘
                            ▼
                   Context {
                     user: { id, name, email, role },
                     auth: { type: "apiKey" | "session", ... }
                   }
```

**Web 端 Context 注入**（[client.ts](./apps/web/server/api/client.ts#L41-L64)）：

```typescript
export const createContext = async (database?, ip?): Promise<Context> => {
  const session = await getServerAuthSession(); // 从 NextAuth JWT Cookie 解码
  return {
    user: session?.user ?? null,
    auth: session?.user
      ? { type: "session" as const }  // ← 只有 type，无额外信息
      : null,
    db,
    req: { ip },
  };
};
```

**API Key Context 注入**（[client.ts](./apps/web/server/api/client.ts#L10-L38)）：

```typescript
export async function createContextFromRequest(req: Request) {
  const authorizationHeader = req.headers.get("Authorization");
  if (authorizationHeader?.startsWith("Bearer ")) {
    const token = authorizationHeader.split(" ")[1];
    try {
      const authResult = await authenticateApiKey(token, db);
      return {
        user: authResult.user,
        auth: {
          type: "apiKey" as const,
          keyId: authResult.apiKey.keyId,   // ← 附带 keyId
          scopes: authResult.apiKey.scopes,  // ← 附带权限范围
        },
        db,
        req: { ip },
      };
    } catch {
      // Fallthrough 到 Cookie 认证
    }
  }
  return createContext(db, ip);
}
```

> **关键边界①**：API Key 认证优先于 Cookie 认证。如果请求同时携带了合法的 Bearer Token 和 JWT Cookie，系统会使用 API Key 的身份，而忽略 Cookie 中的用户。

#### 3.4.2 RequestAuth 类型与过程级权限隔离

tRPC 层通过 `ctx.auth.type` 进行三级权限隔离（[index.ts](./packages/trpc/index.ts#L33-L42)）：

```typescript
export type RequestAuth =
  | { type: "apiKey"; keyId: string; scopes: ZApiKeyScope[] }
  | { type: "session" }
  | null;
```

基于此派生出三种 Procedure：

| Procedure 类型 | 构造方式 | 允许的 auth.type | 典型用途 |
|---------------|---------|-----------------|---------|
| `authedProcedure` | 仅校验 `ctx.user?.id` 存在 | `session` 或 `apiKey` | 通用读写（书签、标签等） |
| `sessionProcedure` | `authedProcedure.use(rejectApiKeyAuth())` | **仅 `session`**，API Key 返回 403 | 敏感操作：创建/撤销 API Key、改密、删号 |
| `createScopedAuthedProcedure(resource)` | `authedProcedure` + scope 检查 | `session` 放行 / `apiKey` 需匹配 scope | 分资源的细粒度鉴权（用户、书签、列表等） |

**`rejectApiKeyAuth` 中间件**（[index.ts](./packages/trpc/index.ts#L163-L175)）：

```typescript
function rejectApiKeyAuth(message = "API keys are not allowed for this endpoint") {
  return t.middleware((opts) => {
    if (opts.ctx.auth?.type === "apiKey") {
      throw new TRPCError({ code: "FORBIDDEN", message });
    }
    return opts.next();
  });
}
```

> **关键边界②**：以下端点只能通过 Web JWT 会话访问，API Key 一概拒绝——
> - `apiKeys.create / regenerate / revoke / list`（全部使用 `sessionProcedure`）
> - `users.changePassword` 使用 `usersProcedure`（createScopedAuthedProcedure），但内部还需再次校验密码
> - `users.deleteAccount` 同理
>
> 这意味着：**持有某设备的 API Key 无法横向扩展权限去创建新的 API Key 或修改账号密码**。即使 API Key 泄露，攻击者也无法进一步接管账号。

#### 3.4.3 状态存储与持久性对比

| 维度 | Web JWT 会话 | API Key 设备凭证 |
|-----|-------------|-----------------|
| **存储介质** | 浏览器 HttpOnly Cookie（由 NextAuth.js 管理） | 客户端本地存储（扩展：Chrome Storage；CLI：全局配置；移动端：SecureStore） |
| **服务端存储** | **无**。JWT 本身为自包含签名令牌，不查数据库 | **强存储**。`apiKeys` 表中每行代表一个活跃凭证 |
| **数据库表** | `sessions` 表存在但未使用（`strategy: "jwt"`） | `apiKeys` 表（含 keyId、keyHash、scopes、lastUsedAt） |
| **有效期** | JWT 由 NextAuth 控制（默认 30 天 Cookie 过期 + JWT 自身过期） | **永不过期**，仅在用户主动删除/重新生成时失效 |
| **签发数量** | 每个浏览器一个 Cookie（不追踪多端） | 无上限，每个设备/用途可独立创建一个 Key |
| **泄露后风险窗口** | JWT 过期前一直有效（无法服务端吊销） | 删除 DB 行后**立即失效**（下次请求查不到） |
| **使用痕迹** | 不记录（无 lastUsedAt） | `lastUsedAt` 10 分钟节流更新（[auth.ts](./packages/trpc/auth.ts#L139-L150)） |

#### 3.4.4 各客户端的凭证选型

| 客户端 | 凭证类型 | 代码位置 | 传输方式 |
|-------|---------|---------|---------|
| Web 浏览器 | JWT Cookie | [auth.ts](./apps/web/server/auth.ts#L181-L183) | HttpOnly Cookie（自动携带） |
| 浏览器扩展 | API Key | [trpc.ts](./apps/browser-extension/src/utils/trpc.ts#L102-L106) | `Authorization: Bearer <apiKey>` |
| CLI | API Key | [trpc.ts](./apps/cli/src/lib/trpc.ts#L16-L19) | `Authorization: Bearer <apiKey>` |
| MCP 服务 | API Key | [shared.ts](./apps/mcp/src/shared.ts#L20-L27) | `Authorization: Bearer <apiKey>` |
| 移动端 | API Key | [session.ts](./apps/mobile/lib/session.ts) | `Authorization: Bearer <apiKey>` |
| Workers（内部） | Impersonated Context | [trpc.ts](./apps/workers/trpc.ts#L10-L13) | 直接构造 `AuthedContext`，不走 HTTP |

> **注意 Workers 的特殊路径**：后台任务通过 `buildImpersonatingAuthedContext(userId)` 直接构造 Context，绕过了 HTTP 层。此时 `ctx.auth` 为 `undefined`（不是 `"session"` 也不是 `"apiKey"`），但由于 `authedProcedure` 只检查 `ctx.user?.id`，所以 Worker 可以正常调用需要认证的端点。不过 Worker 无法调用 `sessionProcedure` 保护的端点（因为 auth 不为 `"session"`），这在设计上是正确的——Worker 不应能创建或撤销 API Key。

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
- 列表路由：[apiKeys.ts](./packages/trpc/routers/apiKeys.ts#L102-L131)
- 前端渲染：[ApiKeySettings.tsx](./apps/web/components/settings/ApiKeySettings.tsx#L19-L78)
- sessionProcedure 定义（拒绝 API Key 访问敏感端点）：[index.ts](./packages/trpc/index.ts#L197)

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
- 创建表单：[AddApiKey.tsx](./apps/web/components/settings/AddApiKey.tsx#L159-L359)
- 创建路由：[apiKeys.ts](./packages/trpc/routers/apiKeys.ts#L36-L60)

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
- 重生成路由：[apiKeys.ts](./packages/trpc/routers/apiKeys.ts#L61-L88)

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
- 注销入口：[ProfileOptions.tsx](./apps/web/components/dashboard/header/ProfileOptions.tsx#L144-L147)
- 注销页面：[logout/page.tsx](./apps/web/app/logout/page.tsx#L9-L26)

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
- 扩展注销：[OptionsPage.tsx](./apps/browser-extension/src/OptionsPage.tsx#L102-L109)
- 删除 Key 路由：[apiKeys.ts](./packages/trpc/routers/apiKeys.ts#L89-L101)

### 5.3 移动端注销

```
调用 useSession().logout()
    ↓
[session.ts:logout]
    │  ├─ 若存在 settings.apiKeyId → 调用 api.apiKeys.revoke()
    │  └─ 清除本地 settings 中的 apiKey 和 apiKeyId
```

关键代码：
- 移动端会话 Hook：[session.ts](./apps/mobile/lib/session.ts#L8-L26)

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

### 5.5 远程撤销实际效果差异 — Web JWT vs API Key

项目对两种凭证的远程吊销能力存在本质差异，这是理解设备会话管理边界的核心。

#### 5.5.1 撤销操作矩阵

| 操作 | 发起方 | 目标凭证 | 是否需要原凭证 | 服务端是否生效 | 客户端是否感知 |
|-----|--------|---------|--------------|--------------|--------------|
| Web 本地 Sign Out | Web 浏览器自身 | 自身 JWT Cookie | 否（用户交互即可） | **否**（仅清除本地 Cookie） | 立即可感知（跳转首页） |
| 删除用户（管理员或自删） | 管理员 / 用户本人 | 该用户所有 JWT + 所有 API Key | 是（管理员会话 / 本人密码） | **是** | JWT：下一次 whoami/settings 查询时被 ValidAccountCheck 拦截；API Key：下次请求立即 401 |
| 撤销某个 API Key | Web 会话（用户本人） | 指定 API Key | 是（Web JWT 会话） | **是**（DELETE 行） | 下次请求立即 401 → 设备显示未登录 |
| 修改用户密码 | Web 会话（用户本人） | — | 是（旧密码） | **否**（不影响已签发 JWT，不影响已有 API Key） | 无感知（当前所有会话继续有效） |
| API Key 重新生成（regenerate） | Web 会话（用户本人） | 指定 API Key | 是（Web JWT 会话） | **是**（UPDATE keyHash） | 旧 Key 下次请求立即 401 |

#### 5.5.2 API Key 撤销的即时失效 — 代码级证据

API Key 的撤销是**同步且即时**的，整条链路没有任何缓存或延迟窗口。以下按代码执行顺序证明：

**步骤 1：撤销操作直接删除数据库行**

[apiKeys.ts](./packages/trpc/routers/apiKeys.ts#L89-L101) 中 `revoke` 过程：

```typescript
revoke: sessionProcedure
  .input(z.object({ id: z.string() }))
  .mutation(async ({ input, ctx }) => {
    const res = await ctx.db
      .delete(apiKeys)
      .where(and(eq(apiKeys.id, input.id), eq(apiKeys.userId, ctx.user.id)));
    if (res.changes == 0) {
      throw new TRPCError({ code: "NOT_FOUND" });
    }
  }),
```

关键事实：
- 使用 `sessionProcedure` 确保只能从 Web 会话发起（防止 API Key 互相撤销）
- 直接执行 `DELETE FROM apiKeys WHERE id = ? AND userId = ?`
- 这是一个同步的 Drizzle ORM 操作，事务提交后数据库中立即不存在该行

**步骤 2：后续请求的认证直接查数据库**

被撤销的设备下次发起请求时，经过 [client.ts](./apps/web/server/api/client.ts#L10-L38) 的 `createContextFromRequest()`：

```typescript
const authorizationHeader = req.headers.get("Authorization");
if (authorizationHeader && authorizationHeader.startsWith("Bearer ")) {
  const token = authorizationHeader.split(" ")[1];
  try {
    const authResult = await authenticateApiKey(token, db);
    // ... 返回认证结果
  } catch {
    // Fallthrough 到 Cookie 认证
  }
}
```

调用 [auth.ts](./packages/trpc/auth.ts#L107-L160) 中的 `authenticateApiKey()`：

```typescript
export async function authenticateApiKey(key: string, database) {
  const { version, keyId, keySecret } = parseApiKey(key);
  // ↓ 每次认证都直接查数据库，无缓存
  const apiKey = await database.query.apiKeys.findFirst({
    where: (k, { eq }) => eq(k.keyId, keyId),
    with: { user: true },
  });

  if (!apiKey) {
    throw new Error("API key not found");  // ← 撤销后立即走这里
  }
  // ... 哈希校验
}
```

关键事实：
- `findFirst` 直接查询 SQLite/PostgreSQL，无任何内存缓存
- 撤销操作（DELETE）和认证查询（SELECT）都走同一个数据库实例
- 没有"软删除"、"标记失效"等延迟机制——行不存在就是不存在

**步骤 3：认证失败后的 fallthrough 行为**

`authenticateApiKey` 抛出异常后，`createContextFromRequest` 的 catch 块执行 fallthrough：

```typescript
catch {
  // Fallthrough to cookie-based auth
}
return createContext(db, ip);
```

`createContext()` 从 Cookie 中读取 JWT。对于扩展/移动设备，请求中不携带 Cookie（跨域），所以 `ctx.user` 为 null。

然后 tRPC 的 [index.ts](./packages/trpc/index.ts#L140-L152) 中 `authedProcedure` 的 `isAuthed` 中间件：

```typescript
.use(function isAuthed(opts) {
  if (!opts.ctx.user?.id) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return opts.next({ ctx: { user } });
});
```

最终返回 HTTP 401。设备侧收到 401 后，通常会清理本地认证状态并跳转登录页。

> **结论**：API Key 从"点击撤销"到"设备请求被拒绝"之间的延迟 = 网络往返时间 + 数据库查询时间（通常 < 50ms）。没有任何异步窗口或缓存不一致。

#### 5.5.3 JWT 撤销的异步延迟窗口 — 代码级证据

**Web JWT 无法被服务端即时吊销**。如果 JWT 被窃取，攻击者在 JWT 自然过期前可以持续使用。

**证据 1：JWT 策略配置 — 无服务端会话表**

[auth.ts](./apps/web/server/auth.ts#L181-L183) 中 NextAuth 明确使用 JWT 策略：

```typescript
session: {
  strategy: "jwt",
},
```

这意味着：
- 不使用数据库 `sessions` 表存储会话
- 所有用户信息编码在 JWT payload 中，由 NextAuth 签名
- 认证时只验证签名，不查数据库（除非业务逻辑主动查）

虽然 [schema.ts](./packages/db/schema.ts#L127-L136) 中存在 `sessions` 表（DrizzleAdapter 要求），但由于 strategy 是 jwt，该表在 Web 端认证中不会被读写。

**证据 2：Context 构造不查用户表**

[client.ts](./apps/web/server/api/client.ts#L41-L64) 的 `createContext()` 直接从 NextAuth 取 session：

```typescript
export const createContext = async (database?, ip?) => {
  const session = await getServerAuthSession();  // ← 仅解码 JWT，不查 DB
  return {
    user: session?.user ?? null,
    auth: session?.user ? { type: "session" } : null,
    db,
    req: { ip },
  };
};
```

`getServerAuthSession()` 由 NextAuth 提供，只做 JWT 签名验证和解码，不访问数据库。

**证据 3：authedProcedure 只检查 ctx.user.id 是否存在**

[index.ts](./packages/trpc/index.ts#L140-L152)：

```typescript
.use(function isAuthed(opts) {
  const user = opts.ctx.user;
  if (!user?.id) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return opts.next({ ctx: { user } });
});
```

只要 JWT 解码出用户 ID，认证就通过。**不验证用户在数据库中是否仍然存在**。

**证据 4：三层防御的延迟性**

项目通过三层防御缓解 JWT 不可吊销的问题，但每一层都有延迟：

```
攻击者持被窃取的 JWT 发起请求
    │
    ├─ [第一层] tRPC authedProcedure
    │     仅检查 ctx.user?.id 存在 → JWT 解码通过 → ✅ 放行
    │     [代码位置：packages/trpc/index.ts#L140-L152]
    │
    ├─ [第二层] Server Component 布局二次校验
    │     dashboard/layout.tsx、reader/layout.tsx、settings/layout.tsx
    │     调用 api.users.settings() → User.fromCtx(ctx) 查数据库
    │     若用户已被删除 → NOT_FOUND → redirect("/logout")
    │     [代码位置：apps/web/app/dashboard/layout.tsx#L32-L52]
    │     [代码位置：packages/trpc/models/users.ts — User.fromCtx]
    │
    └─ [第三层] 前端 ValidAccountCheck
          useQuery(api.users.whoami) → whoami 使用 usersProcedure
          User.fromCtx(ctx) 查数据库 users 表
          若用户不存在 → NOT_FOUND → whoami 抛出 UNAUTHORIZED
          → ValidAccountCheck 捕获 → router.push("/logout")
          [代码位置：apps/web/components/utils/ValidAccountCheck.tsx#L13-L33]
```

> **关键边界**：第二、三层防御都依赖 `User.fromCtx(ctx)` 查询数据库。
> - 如果用户只是执行了"本地 Sign Out"（清除了本地 Cookie，但数据库中 users 行仍存在），则第二、三层防御都不会触发——被窃取的 JWT 仍可正常使用直到过期。
> - 只有当用户被**删除**（deleteAccount）或在数据库层面被**禁用**（当前无禁用字段）时，异步撤销才会生效。
>
> 这就是 `sessions` 表虽然存在但未被使用的安全代价：没有服务端会话白名单/黑名单机制。

**`User.fromCtx` 的关键作用**（[users.ts](./packages/trpc/models/users.ts)）：

```typescript
static async fromCtx(ctx) {
  const user = await ctx.db.query.users.findFirst({
    where: (u, { eq }) => eq(u.id, ctx.user.id),
  });
  if (!user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return new User(ctx, user);
}
```

所有通过 `usersProcedure` 的端点（`whoami`、`settings`、`changePassword`、`deleteAccount` 等）都会走此方法。它是 JWT 与数据库状态的最终一致性校验点。

#### 5.5.4 改密不吊销会话 — 代码级证据

修改密码后，**所有已签发的 JWT 和所有 API Key 继续有效**。

**证据 1：changePassword 只更新 users 表**

[users.ts](./packages/trpc/models/users.ts#L440-L465) 中 `changePassword` 方法：

```typescript
async changePassword(currentPassword: string, newPassword: string) {
  invariant(this.ctx.user.email, "A user always has an email specified");

  // 第一步：验证旧密码
  try {
    const user = await validatePassword(
      this.ctx.user.email,
      currentPassword,
      this.ctx.db,
    );
    invariant(user.id === this.ctx.user.id);
  } catch {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }

  // 第二步：生成新盐 + 新哈希
  const newSalt = generatePasswordSalt();
  await this.ctx.db
    .update(users)
    .set({
      password: await hashPassword(newPassword, newSalt),
      salt: newSalt,
    })
    .where(eq(users.id, this.user.id));

  // 注意：这里没有任何以下操作
  //  - 没有删除 apiKeys 表的记录
  //  - 没有使 sessions 表失效（本来也没用）
  //  - 没有 JWT 版本号/盐值更新
}
```

**证据 2：JWT 验证不依赖密码**

JWT 的签名密钥是 `NEXTAUTH_SECRET`（环境变量），与用户密码无关。密码修改后，已签发的 JWT 仍然可以正常验证通过。

验证链路：
1. 请求到达 → `getServerAuthSession()` → JWT 解码 → 得到 user.id
2. `authedProcedure` 检查 user.id 存在 → 通过
3. 不涉及密码字段

**证据 3：API Key 认证也不依赖密码**

[auth.ts](./packages/trpc/auth.ts#L107-L160) 中 `authenticateApiKey()` 通过 keyId 和 keyHash 校验，与 users 表的 password 字段完全无关：

```typescript
const apiKey = await database.query.apiKeys.findFirst({
  where: (k, { eq }) => eq(k.keyId, keyId),
  with: { user: true },  // ← user 只是顺便查出来填充 ctx
});
// 只校验 keyHash，不校验 user.password
```

因此，修改密码不会导致任何 API Key 失效。

> **设计权衡**：改密不吊销现有会话是便利性优先的选择（用户不需要在所有设备上重新登录）。但这意味着：如果用户因为"怀疑凭证泄露"而改密，已经获得了 JWT 或 API Key 的攻击者仍然可以继续访问。用户必须**手动**去 API Key 列表逐个撤销可疑的设备 Key。

#### 5.5.5 三者集中对照 — 代码证据 / 触发条件 / 安全影响

将 API Key 撤销（即时失效）、JWT 吊销（延迟窗口）、改密不吊销会话三者从代码角度并列对比：

| 对比维度 | API Key 撤销（即时失效） | JWT 吊销（延迟窗口） | 改密不吊销会话 |
|---------|------------------------|---------------------|---------------|
| **核心代码位置** | [apiKeys.ts](./packages/trpc/routers/apiKeys.ts#L89-L101) `revoke` 过程 | [auth.ts](./apps/web/server/auth.ts#L181-L183) `session.strategy: "jwt"` | [users.ts](./packages/trpc/models/users.ts#L440-L465) `changePassword` 方法 |
| **失效判断点** | `authenticateApiKey()` 中 `db.query.apiKeys.findFirst()` | `getServerAuthSession()` 中 JWT 签名验证 | 不涉及失效判断（持续有效） |
| **判断代码** | `if (!apiKey) throw new Error("API key not found")` | `if (!user?.id) throw UNAUTHORIZED`（仅检查 ID 是否解码出来） | 无（密码字段与会话验证完全解耦） |
| **数据库操作** | `DELETE FROM apiKeys WHERE id = ? AND userId = ?` | 无（不查数据库） | `UPDATE users SET password=?, salt=? WHERE id=?` |
| **触发条件** | 用户在设置页点击删除 / 调用 `apiKeys.revoke` | 用户被删除（`deleteAccount` 或管理员删除） / 用户被禁用（当前无禁用字段） | 用户执行 `changePassword` |
| **生效时机** | **立即**（DELETE 提交后，下一个请求直接查不到） | **延迟**（依赖业务代码调用 `User.fromCtx()` 查 DB 时才发现用户不存在） | **永不失效**（已签发凭证与密码无关） |
| **延迟窗口** | 0ms（网络 + DB 查询时间） | 最长 = JWT 自然过期时间（NextAuth 默认 30 天） | 永久（直到 JWT/Key 自身过期或被主动撤销） |
| **影响范围** | 仅被撤销的单个 API Key | 该用户所有已签发的 JWT | 无影响（所有 JWT 和 API Key 继续有效） |
| **攻击者视角** | 拿到 Key → 被撤销 → 立即失效 → 必须重新获取 | 拿到 JWT → 用户改密 → 仍然有效 → 继续使用直到过期 | 拿到 JWT/Key → 用户改密 → 完全不受影响 → 持续可用 |
| **缓解措施** | 本身就是强控制（即时失效） | 1. ValidAccountCheck 前端探针<br>2. Server Component 布局二次校验<br>3. 敏感操作使用 `usersProcedure`（走 `User.fromCtx`） | 无内置缓解<br>→ 需用户手动去 API Key 列表逐个撤销 |
| **用户感知** | 扩展/移动端下次操作立即 401 → 跳转登录页 | 仅在访问 dashboard/reader/settings 等布局时跳 /logout<br>纯 tRPC 接口调用可能不触发 | 完全无感知（所有设备继续正常使用） |
| **对应凭证类型** | API Key（设备凭证） | Web JWT Cookie（浏览器会话） | 两者都不受影响 |

**三条代码路径的对照图：**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        凭证验证时的代码路径对比                                 │
├──────────────────────────┬──────────────────────────┬───────────────────────┤
│   API Key 验证            │   JWT 验证                │   密码修改            │
│  [auth.ts:107-160]       │  [client.ts:41-64]       │  [users.ts:440-465]   │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│  1. parseApiKey(key)     │  1. getServerAuthSession │  1. validatePassword  │
│     解析前缀+keyId+secret│     解码 JWT (不查DB)    │     验证旧密码       │
│                          │                          │                       │
│  2. db.query.apiKeys.    │  2. ctx.user = token.user│  2. generatePasswordSalt│
│     findFirst(keyId)     │     (直接从JWT payload拿) │    生成新盐           │
│     ────────查DB──────── │                          │                       │
│                          │  3. authedProcedure 检查  │  3. hashPassword      │
│  3. if (!apiKey)         │     user.id 是否存在      │     bcrypt 哈希       │
│     throw "not found"    │     (不查DB，只看解码结果)│                       │
│     ← 撤销后立即走这里    │                          │  4. UPDATE users SET   │
│                          │                          │     password + salt    │
│  4. bcrypt/sha256 校验    │  ✅ 只要 JWT 签名有效     │    ← 只改 users 表    │
│     secret vs keyHash    │     就一直有效            │    不碰 apiKeys 表    │
│                          │                          │    不碰 sessions 表   │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│  结论：每次请求都查 DB    │  结论：只查 JWT 签名       │  结论：改密与会话完全  │
│        行不在 = 失效      │        不查数据库          │        解耦，互不影响 │
└──────────────────────────┴──────────────────────────┴───────────────────────┘
```

**安全影响的层级排序（从安全到不安全）：**

1. **API Key 撤销** ⭐⭐⭐⭐⭐ — 即时、确定、无延迟。删除即失效，攻击者拿不到新请求的机会。
2. **JWT 失效（用户被删除）** ⭐⭐ — 依赖业务代码是否调用 `User.fromCtx`。访问 dashboard 等完整页面会被 ValidAccountCheck 拦下，但纯 tRPC 调用（如果不走 `usersProcedure`）可能一直能用。
3. **改密不吊销** ⭐ — 完全没有失效机制。攻击者拿到凭证后，即使用户改密，只要不主动撤销 Key，凭证就一直有效到自然过期。

> **代码证据的核心差异**：三者的根本区别在于**认证判断是否查数据库**。
> - API Key：每次认证都 `findFirst` 查 `apiKeys` 表 → 撤销即时生效
> - JWT：认证只验证签名 → `ctx.user` 来自 JWT payload → 不查 DB → 无法即时吊销
> - 改密：只更新 `users.password` 字段 → JWT 和 API Key 的验证逻辑都不读这个字段 → 完全不影响

---

## 六、安全提示与账户有效性校验

### 6.1 ValidAccountCheck — 前台存活探针

这是一个关键的安全组件，解决 **"JWT 仍有效但用户已被删除/禁用"** 的不一致问题。

挂载位置：
- [SidebarLayout.tsx](./apps/web/components/shared/sidebar/SidebarLayout.tsx)（通过全局布局引入）

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
- [ValidAccountCheck.tsx](./apps/web/components/utils/ValidAccountCheck.tsx#L13-L33)

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
- [dashboard/layout.tsx](./apps/web/app/dashboard/layout.tsx#L32-L52)
- [reader/layout.tsx](./apps/web/app/reader/layout.tsx#L15-L32)
- [settings/layout.tsx](./apps/web/app/settings/layout.tsx#L119-L136)

### 6.3 tRPC 认证中间件

定义位置：[index.ts](./packages/trpc/index.ts#L131-L152)

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
- API 端：由 [auth.ts:authenticateApiKey()](./packages/trpc/auth.ts#L107-L160) 解析 API Key 后注入

Hono API 层进一步封装：
- [api/middlewares/auth.ts](./packages/api/middlewares/auth.ts#L24-L37) — `authMiddleware` 校验 `ctx.user` 存在
- [api/middlewares/auth.ts](./packages/api/middlewares/auth.ts#L39-L59) — `adminAuthMiddleware` 额外校验 `role === "admin"`

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
- 修改密码表单：[ChangePassword.tsx](./apps/web/components/settings/ChangePassword.tsx#L28-L209)
- 修改密码模型方法：[users.ts](./packages/trpc/models/users.ts#L440-L465)

### 6.5 邮箱验证强制登录拦截

在 `signIn` callback 中：

```typescript
// auth.ts
if (serverConfig.auth.emailVerificationRequired && !user.emailVerified) {
  throw new Error("Please verify your email address before signing in");
}
```

前端收到此错误时，跳转到 `/check-email` 页面：
- [CredentialsForm.tsx](./apps/web/components/signin/CredentialsForm.tsx#L85-L88)

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
| [packages/db/schema.ts](./packages/db/schema.ts) | 数据库表定义 |
| [apps/web/server/auth.ts](./apps/web/server/auth.ts) | NextAuth 配置（Web 端登录核心） |
| [apps/web/lib/auth/client.ts](./apps/web/lib/auth/client.ts) | 客户端认证导出层 |
| [packages/trpc/auth.ts](./packages/trpc/auth.ts) | 密码校验、API Key 生成与认证 |
| [packages/trpc/index.ts](./packages/trpc/index.ts) | tRPC 过程定义（authedProcedure 等） |
| [packages/trpc/routers/users.ts](./packages/trpc/routers/users.ts) | 用户相关路由（注册、改密、注销等） |
| [packages/trpc/routers/apiKeys.ts](./packages/trpc/routers/apiKeys.ts) | API Key 路由（设备凭证管理） |
| [packages/trpc/models/users.ts](./packages/trpc/models/users.ts) | User 模型（改密、邮箱验证、重置密码） |
| [packages/api/middlewares/auth.ts](./packages/api/middlewares/auth.ts) | Hono API 认证中间件 |
| [apps/web/app/logout/page.tsx](./apps/web/app/logout/page.tsx) | Web 端注销页面 |
| [apps/web/components/utils/ValidAccountCheck.tsx](./apps/web/components/utils/ValidAccountCheck.tsx) | 账户有效性校验 |
| [apps/web/components/dashboard/header/ProfileOptions.tsx](./apps/web/components/dashboard/header/ProfileOptions.tsx) | 用户菜单（含注销入口） |
| [apps/web/components/signin/CredentialsForm.tsx](./apps/web/components/signin/CredentialsForm.tsx) | 密码登录表单 |
| [apps/web/components/signin/SignInForm.tsx](./apps/web/components/signin/SignInForm.tsx) | 登录页组合组件 |
| [apps/web/components/settings/ChangePassword.tsx](./apps/web/components/settings/ChangePassword.tsx) | 修改密码表单 |
| [apps/web/components/settings/ApiKeySettings.tsx](./apps/web/components/settings/ApiKeySettings.tsx) | API Key 列表（设备列表） |
| [apps/web/components/settings/AddApiKey.tsx](./apps/web/components/settings/AddApiKey.tsx) | 新建 API Key |
| [apps/web/app/dashboard/layout.tsx](./apps/web/app/dashboard/layout.tsx) | Dashboard 布局（含二次认证校验） |
| [apps/web/app/reader/layout.tsx](./apps/web/app/reader/layout.tsx) | Reader 布局（含二次认证校验） |
| [apps/web/app/settings/layout.tsx](./apps/web/app/settings/layout.tsx) | Settings 布局（含二次认证校验） |
| [apps/browser-extension/src/OptionsPage.tsx](./apps/browser-extension/src/OptionsPage.tsx) | 浏览器扩展设置（含注销） |
| [apps/mobile/lib/session.ts](./apps/mobile/lib/session.ts) | 移动端会话管理 |
| [apps/web/server/api/client.ts](./apps/web/server/api/client.ts) | Context 构建（JWT Cookie 与 API Key 两条注入链路） |
| [apps/web/server/api/trpc.ts](./apps/web/server/api/trpc.ts) | Server Component tRPC 代理 |
| [apps/web/app/api/\[\[...route\]\]/route.ts](./apps/web/app/api/%5B%5B...route%5D%5D/route.ts) | Next.js API Route 入口（挂载 Hono） |
| [apps/browser-extension/src/utils/trpc.ts](./apps/browser-extension/src/utils/trpc.ts) | 扩展 tRPC 客户端（Bearer Token 注入） |
| [apps/cli/src/lib/trpc.ts](./apps/cli/src/lib/trpc.ts) | CLI tRPC 客户端（Bearer Token 注入） |
| [apps/mcp/src/shared.ts](./apps/mcp/src/shared.ts) | MCP 服务 tRPC 客户端（Bearer Token 注入） |
| [apps/workers/trpc.ts](./apps/workers/trpc.ts) | Worker 内部上下文构造（Impersonate 模式） |
| [packages/trpc/lib/impersonate.ts](./packages/trpc/lib/impersonate.ts) | Impersonated Context 构造函数 |
| [packages/api/index.ts](./packages/api/index.ts) | Hono API 应用组装（tRPC + REST 路由 |
| [packages/api/middlewares/trpcAdapter.ts](./packages/api/middlewares/trpcAdapter.ts) | tRPC 错误到 HTTP 状态码映射 |
| [packages/api/routes/trpc.ts](./packages/api/routes/trpc.ts) | Hono tRPC 路由（Context 透传） |
