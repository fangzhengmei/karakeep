# Karakeep OAuth/OIDC SSO 登录流程详解

## 整体架构概览

Karakeep 使用 **NextAuth.js (Auth.js)** 作为认证框架，采用 **JWT Session** 策略。认证体系支持两种登录方式：
1. **Credentials（本地账号密码）**
2. **OAuth/OIDC（自定义 SSO Provider）**

关键文件：
- 认证配置：[apps/web/server/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts)
- NextAuth 路由：[apps/web/app/api/auth/[...nextauth]/route.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/app/api/auth/%5B...nextauth%5D/route.tsx)
- 数据库 Schema：[packages/db/schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/packages/db/schema.ts)
- 服务端配置：[packages/shared/config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/packages/shared/config.ts)

---

## 1. 登录入口（前端发起）

### 1.1 登录页面渲染

登录页面 [apps/web/app/signin/page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/app/signin/page.tsx) 首先通过 `getServerAuthSession()` 检查用户是否已登录，已登录则直接重定向到首页。

```
用户访问 /signin
  │
  ├─ getServerAuthSession() 检查 Session
  │    ├─ 已登录 → redirect("/")
  │    └─ 未登录 → 渲染 SignInForm
  │
  └─ SignInForm 组件
       ├─ OAuthAutoRedirect（自动跳转逻辑）
       ├─ CredentialsForm（本地账号密码表单）
       └─ SignInProviderButton（OAuth 登录按钮）
```

### 1.2 OAuth 自动重定向

组件 [apps/web/components/signin/OAuthAutoRedirect.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/components/signin/OAuthAutoRedirect.tsx) 满足以下条件时自动跳转 OAuth Provider：

| 条件 | 配置项 | 来源 |
|------|--------|------|
| 启用 OAuth 自动跳转 | `oauthAutoRedirect` | `OAUTH_AUTO_REDIRECT=true` |
| 禁用密码登录 | `disablePasswordAuth` | `DISABLE_PASSWORD_AUTH=true` |
| 存在 OAuth Provider | 配置了 `OAUTH_WELLKNOWN_URL` | 环境变量 |
| 当前无 error 参数 | URL 不含 `?error=` | 查询参数 |

满足条件时，自动调用客户端的 `signIn(providerId)`。

### 1.3 手动点击 OAuth 按钮

组件 [apps/web/components/signin/SignInProviderButton.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/components/signin/SignInProviderButton.tsx) 点击后：

```typescript
signIn(provider.id, { callbackUrl: "/" });
```

客户端 auth 库封装在 [apps/web/lib/auth/client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/lib/auth/client.ts)，本质是 re-export `next-auth/react` 的 `signIn` 函数。

---

## 2. OAuth Provider 配置

在 [apps/web/server/auth.ts#L146-L175](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts#L146-L175) 中，当 `serverConfig.auth.oauth.wellKnownUrl` 存在时，动态注册一个自定义 OAuth Provider：

| 配置项 | 环境变量 | 默认值 |
|--------|----------|--------|
| `wellKnown` | `OAUTH_WELLKNOWN_URL` | - |
| `clientId` | `OAUTH_CLIENT_ID` | - |
| `clientSecret` | `OAUTH_CLIENT_SECRET` | - |
| `scope` | `OAUTH_SCOPE` | `openid email profile` |
| `name` | `OAUTH_PROVIDER_NAME` | `Custom Provider` |
| `timeout` | `OAUTH_TIMEOUT` | `3500` ms |
| `allowDangerousEmailAccountLinking` | `OAUTH_ALLOW_DANGEROUS_EMAIL_ACCOUNT_LINKING` | `false` |

安全校验：使用 `checks: ["pkce", "state"]`，启用 PKCE 和 State 校验。

### profile 回调

Provider 返回用户信息后，`profile()` 函数处理用户角色分配：

```typescript
async profile(profile: Record<string, string>) {
  const [admin, firstUser] = await Promise.all([
    isAdmin(profile.email),     // 检查该 email 是否为已有 admin
    isFirstUser(),              // 检查 user 表是否为空
  ]);

  return {
    id: profile.sub,            // OIDC subject 标识
    name: normalizeSafeDisplayName(profile.name),
    email: profile.email,
    role: admin || firstUser ? "admin" : "user",
  };
}
```

**角色分配规则：**
- 如果是系统中**第一个用户**（user 表为空）→ `admin`
- 如果该 email 对应用户已有 `admin` 角色 → `admin`
- 其他情况 → `user`

---

## 3. Provider 回调处理流程

NextAuth 的路由入口 [apps/web/app/api/auth/[...nextauth]/route.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/app/api/auth/%5B...nextauth%5D/route.tsx) 将 GET/POST 全部交给 `authHandler`（由 `NextAuth(authOptions)` 创建）。

### 3.1 signIn 回调

当 Provider 回调成功后，会触发 [apps/web/server/auth.ts#L191-L252](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts#L191-L252) 的 `signIn` 回调：

```
signIn 回调
  │
  ├─ 提取 email (credUser.email 或 profile.email)
  │    └─ 无 email → 抛出错误 "Provider didn't provide an email"
  │
  ├─ 按 email 查询本地 users 表
  │
  ├─ [分支1] Credentials 登录 (credentials 存在)
  │    ├─ 本地无此用户 → 记日志 + throw "Invalid credentials"
  │    ├─ 开启邮箱验证但未验证 → 记日志 + throw "Please verify your email..."
  │    └─ 通过 → 记日志 user.login + return true
  │
  └─ [分支2] OAuth 登录 (credentials 不存在)
       ├─ 本地无此用户 && DISABLE_SIGNUPS=true
       │    └─ 记日志 user.signup 失败 + throw "Signups are disabled"
       ├─ 本地已有此用户
       │    └─ 记日志 user.login + return true
       └─ 本地无此用户 && 允许注册
            └─ return true (触发 createUser)
```

### 3.2 账号绑定 / 首次注册

当 `signIn` 返回 `true` 且是 OAuth 新用户时，NextAuth 会调用 Adapter 的 `createUser`。项目使用自定义的 `CustomProvider()` Adapter：

[apps/web/server/auth.ts#L88-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts#L88-L112)

```
Adapter 包装了 DrizzleAdapter，重写了 createUser：
  │
  └─ CustomProvider.createUser(user)
       │
       ├─ 调用 User.createRaw(db, {...}) 创建本地用户
       │    ├─ name: normalizeSafeDisplayName(user.name)
       │    ├─ email: user.email
       │    ├─ emailVerified: user.emailVerified
       │    └─ 角色分配由 User.createRaw 内部处理
       │         └─ userCount == 0 ? "admin" : "user"
       │
       └─ 记日志 user.signup 事件
```

**User.createRaw 核心逻辑** 在 [packages/trpc/models/users.ts#L94-L145](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/packages/trpc/models/users.ts#L94-L145)：

```
createRaw 事务内：
  1. 统计用户总数，第一个用户自动为 admin
  2. INSERT user 记录，附带默认配额
  3. 捕获 UNIQUE 约束冲突（email 重复）→ 返回 "Email is already taken"
```

### 3.3 账号关联表

OAuth 账号与本地用户的映射存储在 `accounts` 表 [packages/db/schema.ts#L103-L125](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/packages/db/schema.ts#L103-L125)：

| 字段 | 说明 |
|------|------|
| `userId` | 关联本地 users.id（级联删除） |
| `type` | `oauth` \| `credentials` 等 |
| `provider` | Provider ID（如 `"custom"`） |
| `providerAccountId` | Provider 返回的用户 ID（sub） |
| `refresh_token` / `access_token` | OAuth 令牌 |
| `expires_at` / `token_type` / `scope` | 令牌元数据 |
| `id_token` | OIDC ID Token |

**主键：** `(provider, providerAccountId)` 复合主键，确保同一 Provider 的同一外部用户只能绑定到一个本地账号。

**账号绑定安全策略：**
- 默认 `allowDangerousEmailAccountLinking = false`
- 当此配置为 `false` 时，如果 OAuth 返回的 email 已存在于本地但未通过该 Provider 绑定，NextAuth 默认**不会自动按 email 关联**，而是尝试创建新用户（会因 email 唯一约束失败）
- 设为 `true` 时，允许通过 email 将 OAuth 登录自动关联到已有本地账号（存在安全风险，需确保 Provider 已验证 email）

---

## 4. Session 创建与 JWT 策略

项目配置 `session: { strategy: "jwt" }`，不使用数据库 Session。

### 4.1 JWT 回调

[apps/web/server/auth.ts#L253-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts#L253-L264)

```
jwt({ token, user })
  │
  └─ 如果 user 对象存在（仅在登录/注册时存在）
       └─ 将用户信息注入 token:
            token.user = {
              id, name, email, image, role
            }
  │
  └─ 返回 token（后续请求中 user 为 undefined，保留已有 token.user）
```

**关键点：** `user` 参数仅在首次登录（含 OAuth 回调）时由 NextAuth 传入。后续刷新 JWT 时只传递已有的 `token`，这意味着用户信息在登录时被**快照**到 JWT 中。

### 4.2 Session 回调

[apps/web/server/auth.ts#L265-L268](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts#L265-L268)

```
session({ session, token })
  │
  └─ session.user = { ...token.user }
       (将 JWT 中的用户信息同步到 Session 对象)
```

### 4.3 Session 在前端的使用

根组件 [apps/web/lib/providers.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/lib/providers.tsx) 通过 `<SessionProvider session={session}>` 将会话注入 React Context。

客户端通过 `useSession()` hook（re-export from next-auth）访问会话。

---

## 5. Session 刷新与服务端认证

### 5.1 服务端获取 Session

`getServerAuthSession()` 封装 `getServerSession(authOptions)`，从请求 Cookie 中读取并验证 JWT。

主要使用点：
- **页面级鉴权**：如 `/signin` 检查是否已登录
- **tRPC Context**：[apps/web/server/api/client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/api/client.ts)

### 5.2 tRPC Context 构建

[apps/web/server/api/client.ts#L10-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/api/client.ts)

```
createContextFromRequest(req)
  │
  ├─ 检查 Authorization: Bearer <token>
  │    ├─ 存在 → 尝试 API Key 认证 authenticateApiKey()
  │    │    ├─ 成功 → 返回 apiKey 类型 Context
  │    │    └─ 失败 → 回退到 Cookie Session 认证
  │    └─ 不存在 → 继续 Cookie Session 认证
  │
  └─ createContext()
       │
       ├─ getServerAuthSession() 从 Cookie 解析 JWT
       └─ 返回 session 类型 Context
            ctx.user = session.user (如果已登录)
            ctx.auth = { type: "session" }
```

### 5.3 tRPC 认证中间件

[packages/trpc/index.ts#L131-L152](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/packages/trpc/index.ts#L131-L152)

```
authedProcedure
  │
  ├─ 速率限制（60s 内 3000 请求）
  │
  └─ isAuthed 中间件
       │
       ├─ 检查 ctx.user.id 是否存在
       ├─ 不存在 → 抛 TRPCError UNAUTHORIZED
       └─ 存在 → 将 user 注入 ctx 并继续
```

### 5.4 Session 自动刷新机制

由于使用 JWT 策略，NextAuth.js 的行为：
1. **JWT 存储在 HttpOnly Cookie** 中，默认名为 `next-auth.session-token`
2. **JWT 过期时间**：默认 30 天（可通过配置覆盖）
3. **Session 刷新**：每次调用 `getServerSession()` 或前端 `useSession()` 触发网络请求时，NextAuth 会检查 JWT 是否需要滚动刷新
4. **滚动刷新**：如果启用了 `jwt.maxAge`，当 JWT 剩余寿命少于一半时，会自动签发新 JWT 并更新 Cookie

> 注意：本项目未配置自定义 `jwt.maxAge`，使用 NextAuth 默认值。由于 JWT 中存储了用户信息快照，**用户角色变更等不会自动反映到已有 Session 中**，需用户重新登录才会生效。

---

## 6. 异常回退与错误处理

### 6.1 OAuth 自动重定向回退

在 [OAuthAutoRedirect.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/components/signin/OAuthAutoRedirect.tsx) 中：

```
shouldRedirect = oauthAutoRedirect 
              && disablePasswordAuth 
              && oauthProviderId 存在 
              && !hasError  // URL 有 error 参数时停止自动跳转
```

当 OAuth 流程出错（如用户取消授权、Provider 返回错误），URL 会带 `?error=xxx` 参数，此时**停止自动跳转**，显示完整登录页让用户选择其他方式。

### 6.2 signIn 回调错误

`signIn` 回调抛出 Error 时，NextAuth 会：
1. 中断登录流程
2. 重定向到配置的 `pages.error`（本项目为 `/signin`）
3. URL 附带 `?error=xxx` 参数，前端可据此展示错误信息

| 错误场景 | failure_reason 日志 |
|----------|---------------------|
| Credentials 用户不存在 | `invalid_credentials` |
| Credentials 密码错误 | `invalid_credentials` |
| Credentials 邮箱未验证 | `email_not_verified` |
| OAuth 注册被禁用 | `signups_disabled` |
| Provider 未返回 email | 直接抛错，无特定 reason |

### 6.3 API Key 认证失败回退

[apps/web/server/api/client.ts#L17-L35](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/api/client.ts#L17-L35)

```
Authorization 头存在且为 Bearer
  │
  ├─ authenticateApiKey() 成功 → 使用 API Key 身份
  └─ authenticateApiKey() 失败（try/catch）
       └─ 静默回退到 Cookie-based Session 认证
```

**注意：** API Key 验证失败时不会报错，而是尝试继续使用 Cookie 会话。这种设计避免了 API Key 过期导致已有登录态的用户被意外登出。

### 6.4 用户注册异常

在 `User.createRaw` 中：
- **Email 重复**（`SQLITE_CONSTRAINT_UNIQUE`）→ 抛出 `"Email is already taken"`
- 其他 DB 错误 → 抛出 `"Something went wrong"`

### 6.5 Demo Mode 保护

[tRPC procedure 中间件](packages/trpc/index.ts#L95-L104)：Demo 模式下所有 mutation 被禁止，抛出 `FORBIDDEN`。

---

## 7. 登出流程

[apps/web/app/logout/page.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/app/logout/page.tsx)

```
用户访问 /logout
  │
  ├─ signOut({ redirect: false, callbackUrl: "/" })
  │    └─ NextAuth 清除 JWT Cookie
  │
  ├─ clearHistory() 清除本地搜索历史
  │
  └─ router.push("/") 跳转到首页
```

---

## 8. 关键配置开关汇总

| 环境变量 | 类型 | 默认 | 说明 |
|----------|------|------|------|
| `NEXTAUTH_SECRET` | string | 必填 | JWT 签名密钥 |
| `NEXTAUTH_URL` | url | `http://localhost:3000` | 回调 URL 基础地址 |
| `DISABLE_SIGNUPS` | bool | `false` | 禁用新用户注册（OAuth 和本地） |
| `DISABLE_PASSWORD_AUTH` | bool | `false` | 禁用密码登录（只允许 OAuth） |
| `EMAIL_VERIFICATION_REQUIRED` | bool | `false` | 本地账号必须邮箱验证后才能登录 |
| `OAUTH_WELLKNOWN_URL` | url | - | OIDC Discovery 端点，设置即启用 OAuth |
| `OAUTH_CLIENT_ID` | string | - | OAuth Client ID |
| `OAUTH_CLIENT_SECRET` | string | - | OAuth Client Secret |
| `OAUTH_SCOPE` | string | `openid email profile` | OAuth Scope |
| `OAUTH_PROVIDER_NAME` | string | `Custom Provider` | 登录按钮显示名 |
| `OAUTH_AUTO_REDIRECT` | bool | `false` | 自动跳转 OAuth Provider |
| `OAUTH_ALLOW_DANGEROUS_EMAIL_ACCOUNT_LINKING` | bool | `false` | 允许按 email 自动关联已有账号 |
| `OAUTH_TIMEOUT` | number | `3500` | OAuth HTTP 请求超时（ms） |

---

## 9. 完整时序图（首次 OAuth 注册）

```
用户浏览器                  NextAuth (Web)              OAuth Provider         DB
    │                           │                           │                  │
    │ 1. 点击 "Sign in with X"  │                           │                  │
    │─────────────────────────>│                           │                  │
    │                           │ 2. 生成 state/pkce        │                  │
    │                           │    重定向到授权端点        │                  │
    │<─────────────────────────│                           │                  │
    │ 3. 重定向                 │                           │                  │
    │─────────────────────────────────────────────────────>│                  │
    │                           │                           │ 4. 用户登录/同意  │
    │                           │                           │                  │
    │<─────────────────────────────────────────────────────│                  │
    │ 5. 带 code 回调 /api/auth/callback/custom            │                  │
    │─────────────────────────>│                           │                  │
    │                           │ 6. code → token           │                  │
    │                           │──────────────────────────>│                  │
    │                           │                           │                  │
    │                           │<──────────────────────────│                  │
    │                           │ 7. 用 access_token 取用户信息               │
    │                           │──────────────────────────>│                  │
    │                           │                           │                  │
    │                           │<──────────────────────────│                  │
    │                           │                           │                  │
    │                           │ 8. signIn 回调            │                  │
    │                           │    检查 email 是否存在    │                  │
    │                           │    SELECT * FROM users WHERE email=?        │
    │                           │────────────────────────────────────────────>│
    │                           │<────────────────────────────────────────────│
    │                           │                           │                  │
    │                           │ 9a. 新用户 → createUser   │                  │
    │                           │    INSERT users + INSERT accounts           │
    │                           │────────────────────────────────────────────>│
    │                           │<────────────────────────────────────────────│
    │                           │                           │                  │
    │                           │ 9b. 老用户 → 仅更新 login 日志                │
    │                           │                           │                  │
    │                           │ 10. jwt 回调              │                  │
    │                           │     将 user 写入 token    │                  │
    │                           │                           │                  │
    │                           │ 11. Set-Cookie JWT        │                  │
    │<─────────────────────────│     重定向到 callbackUrl   │                  │
    │                           │                           │                  │
    │ 12. 访问受保护页面         │                           │                  │
    │─────────────────────────>│                           │                  │
    │                           │ 13. getServerAuthSession()│                  │
    │                           │     从 Cookie 解析 JWT    │                  │
    │                           │     构建 tRPC Context     │                  │
```
