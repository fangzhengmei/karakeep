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
- 用户模型：[packages/trpc/models/users.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/packages/trpc/models/users.ts)

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

> 注意：`profile()` 回调返回的 `user` 是**内存对象**，此时还未写入数据库。它会作为后续 `signIn` 回调的输入。

---

## 3. Provider 回调处理全流程

NextAuth 的路由入口 [apps/web/app/api/auth/[...nextauth]/route.tsx](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/app/api/auth/%5B...nextauth%5D/route.tsx) 将 GET/POST 全部交给 `authHandler`（由 `NextAuth(authOptions)` 创建）。

### 3.1 回调处理总览

OAuth 回调到达后的完整处理顺序：

```
Provider 回调
  │
  ├─ 1. 校验 state / PKCE
  ├─ 2. 用 code 交换 access_token + id_token
  ├─ 3. 调用 provider.profile() 构造内存 user 对象
  │
  ├─ 4. 调用 callbacks.signIn()  ← 应用层拦截点
  │     │
  │     ├─ 4a. 按 email 查询本地 users 表（应用层主动查，代码 L196-L199）
  │     ├─ 4b. 禁用注册检查、邮箱验证检查等
  │     ├─ 抛错 / return false → 终止，重定向到 error 页
  │     └─ return true → 继续
  │
  ├─ 5. callbackHandler 账号查找与绑定 ← NextAuth 内部逻辑 (callback-handler.js)
  │     │
  │     ├─ 5a. getUserByAccount(provider, providerAccountId)
  │     │    │   查 accounts 表复合主键，确认外部账号是否已绑定
  │     │    ├─ 找到账号 → 拿到关联的本地 user → 跳到步骤 6
  │     │    └─ 未找到账号 → 新外部账号，继续 5b
  │     │
  │     └─ 5b. 新外部账号处理（无已登录 session 时）
  │          │
  │          ├─ getUserByEmail(email)  ← 无论 allowDangerousEmailAccountLinking 值为何，都会执行
  │          │
  │          ├─ [A] getUserByEmail 找到本地用户
  │          │    ├─ allowDangerousEmailAccountLinking = true
  │          │    │    └─ 复用该用户: user = userByEmail → linkAccount() → 成功
  │          │    └─ allowDangerousEmailAccountLinking = false (默认)
  │          │         └─ 抛 AccountNotLinkedError → OAuthAccountNotLinked
  │          │            （不会尝试 createUser，在 getUserByEmail 发现冲突时就直接报错）
  │          │
  │          └─ [B] getUserByEmail 未找到本地用户
  │               └─ createUser() + linkAccount() → 新用户注册成功
  │
  ├─ 6. 调用 jwt 回调（JWT 策略）
  ├─ 7. 调用 session 回调
  └─ 8. Set-Cookie + 重定向到 callbackUrl
```

**关键理解（按代码事实）：**
- 两次 users 表查询发生在**不同位置**：
  1. `signIn` 回调中（L196-L199）：**应用层主动查询**，用于禁用注册判断、登录日志记录
  2. NextAuth 内部（仅当 `allowDangerousEmailAccountLinking=true` 时）：框架内部调用 `getUserByEmail`，用于自动按邮箱关联
- `signIn` 回调发生在 Adapter 账号查找/创建**之前**。它只决定"这个 email 能不能登录"，不检查"这个外部账号是否已绑定"
- `allowDangerousEmailAccountLinking=false`（默认）时，NextAuth**绝不**按 email 查找已有用户，直接尝试创建新用户，这是触发账号未关联错误的根源

### 3.2 signIn 回调详解

[apps/web/server/auth.ts#L191-L252](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts#L191-L252)

```
signIn 回调（入参：credUser, credentials, profile）
  │
  ├─ 提取 email (credUser.email 或 profile.email)
  │    └─ 无 email → 抛出错误 "Provider didn't provide an email during signin"
  │
  ├─ 按 email 查询本地 users 表（开发者主动查询，非 NextAuth 传入）
  │
  ├─ [分支 1] Credentials 登录 (credentials 存在)
  │    ├─ 本地无此用户 → 记日志 user.login_failed + throw "Invalid credentials"
  │    ├─ 开启邮箱验证但未验证 → 记日志 user.login_failed + throw "Please verify..."
  │    └─ 通过 → 记日志 user.login + return true
  │
  └─ [分支 2] OAuth 登录 (credentials 不存在)
       ├─ 本地无此用户 && DISABLE_SIGNUPS=true
       │    └─ 记日志 user.signup 失败 (failure_reason: signups_disabled)
       │         + throw "Signups are disabled in server config"
       ├─ 本地已有此用户
       │    └─ 记日志 user.login + return true
       └─ 本地无此用户 && 允许注册
            └─ return true（放行，交由 NextAuth 后续创建用户）
```

**重要细节：**
- `credUser` 是 `profile()` 回调返回的内存对象，不是数据库中的用户
- 回调内部**主动查询数据库**来判断用户是否存在，这是应用层自己的逻辑
- `disableSignups` 的判断粒度是"按 email 是否存在"，不是"按 account 是否存在"

### 3.3 Adapter 账号查找与绑定（NextAuth 内部）

signIn 返回 `true` 后，NextAuth 进入内部账号处理流程。项目使用基于 `@auth/drizzle-adapter` 的自定义 Adapter：[apps/web/server/auth.ts#L88-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts#L88-L112)

#### 第一步：按外部账号查找

```
getUserByAccount(provider, providerAccountId)
  │
  └─ SELECT * FROM accounts
     WHERE provider = ? AND providerAccountId = ?
     JOIN users ON users.id = accounts.userId
```

- **命中** → 这是一个已绑定的外部账号，直接使用关联的本地用户
- **未命中** → 这是一个新的外部账号，进入下一步处理

#### 第二步：新外部账号处理

根据 `allowDangerousEmailAccountLinking` 配置走不同分支：

```
新外部账号
  │
  ├─ [分支 A] allowDangerousEmailAccountLinking = true
  │    │
  │    ├─ 按 email 查找本地用户：getUserByEmail(email)
  │    │    │
  │    │    ├─ 找到用户 → 账号绑定
  │    │    │    └─ linkAccount(userId, accountInfo)
  │    │    │         INSERT INTO accounts (...)
  │    │    │
  │    │    └─ 未找到用户 → 创建用户 + 绑定
  │    │         ├─ createUser(userInfo)
  │    │         │    INSERT INTO users (...)
  │    │         └─ linkAccount(userId, accountInfo)
  │    │              INSERT INTO accounts (...)
  │    │
  │    └─ 结果：已有账号登录 / 新用户注册 + 绑定
  │
  └─ [分支 B] allowDangerousEmailAccountLinking = false (默认)
       │
       ├─ 直接创建新用户 + 绑定
       │    ├─ createUser(userInfo)
       │    │    INSERT INTO users (...)
       │    │    └─ [异常] email 唯一约束冲突 → 抛出错误
       │    └─ linkAccount(userId, accountInfo)
       │         INSERT INTO accounts (...)
       │
       └─ 结果：正常情况下新用户注册成功
              如果 email 已存在 → 登录失败（账号未绑定）
```

### 3.4 自定义 Adapter 的 createUser

`CustomProvider` 包装了 DrizzleAdapter，并重写了 `createUser` 方法：

```
CustomProvider.createUser(user)
  │
  ├─ 调用 User.createRaw(db, {...}) 创建本地用户
  │    ├─ name: normalizeSafeDisplayName(user.name)
  │    ├─ email: user.email
  │    ├─ emailVerified: user.emailVerified
  │    └─ 角色分配在 User.createRaw 内部处理
  │         └─ SELECT count(*) FROM users == 0 ? "admin" : "user"
  │
  ├─ 记日志 user.signup 事件 (auth.provider: "oauth")
  │
  └─ 返回创建的用户对象
```

**User.createRaw 核心逻辑** 在 [packages/trpc/models/users.ts#L94-L145](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/packages/trpc/models/users.ts#L94-L145)：

```
createRaw 事务内：
  1. 统计用户总数，第一个用户自动为 admin
  2. INSERT user 记录，附带默认配额
  3. 捕获 UNIQUE 约束冲突 → 抛 TRPCError "Email is already taken"
```

> 注意：`profile()` 回调中也做了角色判断（admin / firstUser），但 `createUser` 内部会再次判断。两者一致，都以"第一个用户为 admin"为原则。

### 3.5 账号关联表（accounts）

OAuth 外部账号与本地用户的映射存储在 `accounts` 表 [packages/db/schema.ts#L103-L125](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/packages/db/schema.ts#L103-L125)：

| 字段 | 说明 |
|------|------|
| `userId` | 关联本地 users.id（级联删除） |
| `type` | `oauth` \| `credentials` 等 |
| `provider` | Provider ID（如 `"custom"`） |
| `providerAccountId` | Provider 返回的用户 ID（即 OIDC 的 sub） |
| `refresh_token` / `access_token` | OAuth 令牌 |
| `expires_at` / `token_type` / `scope` | 令牌元数据 |
| `id_token` | OIDC ID Token |
| `session_state` | 会话状态 |

**主键：** `(provider, providerAccountId)` 复合主键

这确保了**同一 Provider 的同一外部用户只能绑定到一个本地账号**。但一个本地用户可以绑定多个不同 Provider 的账号。

**外键：** `userId` → `users.id`，`ON DELETE CASCADE`，删除用户时自动清理所有关联账号。

### 3.6 账号绑定安全策略详解

`allowDangerousEmailAccountLinking` 是关键的安全开关：

| 配置值 | 行为 | 安全风险 |
|--------|------|----------|
| `false`（默认） | 新外部账号总是创建新用户，绝不按 email 自动关联 | 低；但如果用户已有本地账号，用同一 email 的 OAuth 登录会创建重复账号失败 |
| `true` | 新外部账号先按 email 查找本地用户，找到就自动绑定 | 中高；如果 Provider 没有验证 email，攻击者可能伪造 email 来接管已有账号 |

**默认策略下的异常场景：**
- 用户先用邮箱注册了本地账号（或其他 Provider）
- 后来用同一个 email 的新 OAuth Provider 登录
- 因为 `allowDangerousEmailAccountLinking = false`
- NextAuth 不会自动将新 OAuth 账号绑定到已有用户
- 会尝试创建新用户 → email 唯一约束冲突 → 登录失败
- 用户看到错误页面，但原有账号不受影响

---

## 4. Session 创建与 JWT 策略

项目配置 `session: { strategy: "jwt" }`，不使用数据库 Session。

### 4.1 JWT 回调

[apps/web/server/auth.ts#L253-L264](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/auth.ts#L253-L264)

```
jwt({ token, user })
  │
  └─ 如果 user 对象存在（仅在登录/注册时传入）
       └─ 将用户信息注入 token:
            token.user = {
              id, name, email, image, role
            }
  │
  └─ 返回 token（后续刷新时 user 为 undefined，保留已有 token.user）
```

**关键点：** `user` 参数仅在首次登录（含 OAuth 回调成功）时由 NextAuth 传入。后续 JWT 刷新时只传递已有的 `token`，这意味着用户信息在登录时被**快照**到 JWT 中。

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
2. **JWT 过期时间**：默认 30 天（可通过 `jwt.maxAge` 配置覆盖）
3. **Session 刷新**：每次调用 `getServerSession()` 或前端 `useSession()` 触发网络请求时，NextAuth 会检查 JWT 是否需要滚动刷新
4. **滚动刷新**：当 JWT 剩余寿命少于一半时，会自动签发新 JWT 并更新 Cookie

> 注意：本项目未配置自定义 `jwt.maxAge`，使用 NextAuth 默认值。由于 JWT 中存储了用户信息快照，**用户角色变更、权限变更等不会自动反映到已有 Session 中**，需用户重新登录才会生效。

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

这是一个重要的"逃生通道"：如果 OAuth 配置有问题或 Provider 不可用，用户不会被困在无限重定向循环中。

### 6.2 signIn 回调错误

`signIn` 回调抛出 Error 时，NextAuth 会：
1. 中断登录流程
2. 重定向到配置的 `pages.error`（本项目为 `/signin`）
3. URL 附带 `?error=xxx` 参数

| 错误场景 | failure_reason 日志 | 错误消息 |
|----------|---------------------|----------|
| Credentials 用户不存在 | `invalid_credentials` | `Invalid credentials` |
| Credentials 密码错误 | `invalid_credentials` | （由 validatePassword 抛出） |
| Credentials 邮箱未验证 | `email_not_verified` | `Please verify your email address before signing in` |
| OAuth 注册被禁用 | `signups_disabled` | `Signups are disabled in server config` |
| Provider 未返回 email | 无 | `Provider didn't provide an email during signin` |

### 6.3 账号绑定异常（重点）

这是 OAuth 登录中最容易被忽略的异常分支。

#### 场景 A：新外部账号 + email 已存在 + 禁止邮箱关联

**触发条件：**
- `allowDangerousEmailAccountLinking = false`（默认）
- 该 (provider, providerAccountId) 在 accounts 表中不存在
- 但 profile.email 在 users 表中已存在（可能是本地账号或其他 Provider）

**处理路径：**

```
1. signIn 回调
   │
   ├─ 按 email 查询到用户存在
   ├─ 记录 user.login 日志（注意：此时还没真正登录成功）
   └─ return true（放行）

2. NextAuth 内部账号处理
   │
   ├─ getUserByAccount() → 未找到（新外部账号）
   │
   ├─ allowDangerousEmailAccountLinking = false
   │    └─ 直接 createUser()
   │
   ├─ createUser() 执行 INSERT INTO users
   │    └─ Email 唯一约束冲突（SQLITE_CONSTRAINT_UNIQUE）
   │
   └─ 抛出错误 → 登录失败

3. 错误回退
   │
   ├─ 重定向到 /signin?error=OAuthAccountNotLinked
   └─ 用户看到错误（但前端未做特殊错误展示）
```

**注意：** 在这个场景下，`signIn` 回调已经记录了 `user.login` 事件日志，但实际上登录最终失败了。这是一个日志与实际结果不一致的边缘情况。

#### 场景 B：新外部账号 + 禁用注册

**触发条件：**
- `DISABLE_SIGNUPS = true`
- 该 email 在 users 表中不存在

**处理路径：**

```
1. signIn 回调
   │
   ├─ 按 email 查询 → 用户不存在
   ├─ disableSignups = true
   ├─ 记录 user.signup 失败日志 (failure_reason: signups_disabled)
   └─ throw Error("Signups are disabled in server config")

2. 错误回退
   │
   └─ 重定向到 /signin?error=...
```

**注意：** 这种情况在 signIn 阶段就被拦截了，不会走到 Adapter 的 createUser 步骤。

#### 场景 C：新外部账号 + email 已存在 + 允许邮箱关联

**触发条件：**
- `allowDangerousEmailAccountLinking = true`
- accounts 表中无此账号，但 users 表中有此 email

**处理路径：**

```
1. signIn 回调 → 查到用户存在 → 记 login 日志 → return true

2. NextAuth 内部账号处理
   │
   ├─ getUserByAccount() → 未找到
   │
   ├─ allowDangerousEmailAccountLinking = true
   │    ├─ getUserByEmail() → 找到用户
   │    └─ linkAccount() → INSERT INTO accounts
   │
   └─ 账号绑定成功

3. jwt / session 回调 → 登录成功
```

这是"账号自动绑定"的正常路径。

### 6.4 API Key 认证失败回退

[apps/web/server/api/client.ts#L17-L35](file:///d:/fz/0601-2/solo-dogfeeding/code/43-karakeep/apps/web/server/api/client.ts#L17-L35)

```
Authorization 头存在且为 Bearer
  │
  ├─ authenticateApiKey() 成功 → 使用 API Key 身份
  └─ authenticateApiKey() 失败（try/catch 捕获）
       └─ 静默回退到 Cookie-based Session 认证
```

**设计意图：** API Key 验证失败时不会报错，而是尝试继续使用 Cookie 会话。这种设计避免了 API Key 过期或无效时，已有登录态的浏览器用户被意外登出。

**潜在问题：** 如果 API Key 是错误格式或已过期，调用方可能期望得到明确的错误提示，但实际上回退到了（可能未登录的）Cookie 身份，返回 UNAUTHORIZED 而非"API Key 无效"。

### 6.5 用户注册异常

在 `User.createRaw` 中：
- **Email 重复**（`SQLITE_CONSTRAINT_UNIQUE`）→ 抛出 TRPCError `BAD_REQUEST` + "Email is already taken"
- 其他 DB 错误 → 抛出 TRPCError `INTERNAL_SERVER_ERROR` + "Something went wrong"

对于 OAuth 流程中的 `createUser`，如果 email 重复（场景 6.3.1），DrizzleAdapter 捕获错误后由 NextAuth 处理，最终表现为登录失败。

### 6.6 Provider 超时 / 不可用

配置 `OAUTH_TIMEOUT`（默认 3500ms）控制 OAuth HTTP 请求超时。超时会导致登录失败，回退到登录页并显示错误。

### 6.7 Demo Mode 保护

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

注意：登出只清除本地 Session Cookie，不会通知 OAuth Provider 登出（无 SLO / 单点登出）。

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

## 9. 完整时序图（按代码事实，含异常分支）

```
用户浏览器                  NextAuth (Web)              OAuth Provider         DB (users+accounts)
    │                           │                           │                  │
    │ 1. 点击 "Sign in with X"  │                           │                  │
    │─────────────────────────>│                           │                  │
    │                           │                           │                  │
    │                           │ 2. 生成 state/pkce        │                  │
    │                           │    重定向到授权端点        │                  │
    │<─────────────────────────│                           │                  │
    │                           │                           │                  │
    │ 3. 重定向到 Provider       │                           │                  │
    │─────────────────────────────────────────────────────>│                  │
    │                           │                           │                  │
    │                           │                           │ 4. 用户登录/授权  │
    │                           │                           │                  │
    │<─────────────────────────────────────────────────────│                  │
    │                           │                           │                  │
    │ 5. 带 code 回调 /api/auth/callback/custom            │                  │
    │─────────────────────────>│                           │                  │
    │                           │                           │                  │
    │                           │ 6. 校验 state + pkce       │                  │
    │                           │    code → access_token     │                  │
    │                           │──────────────────────────>│                  │
    │                           │                           │                  │
    │                           │<──────────────────────────│                  │
    │                           │                           │                  │
    │                           │ 7. 取 userinfo / 解析 id_token             │
    │                           │──────────────────────────>│                  │
    │                           │                           │                  │
    │                           │<──────────────────────────│                  │
    │                           │                           │                  │
    │                           │ 8. 调用 provider.profile() │                  │
    │                           │    构造内存 user 对象      │                  │
    │                           │                           │                  │
    │                           │ 9. 调用 callbacks.signIn() │                  │
    │                           │    [L196-L199] 查 users 表 (按 email)       │
    │                           │────────────────────────────────────────────>│
    │                           │<────────────────────────────────────────────│
    │                           │                           │                  │
    │   ┌─────────────────────────────────────────────────┐                  │
    │   │ 【异常分支 1】新用户 + 禁用注册                    │                  │
    │   │  !user && disableSignups → true                  │                  │
    │   │  记 signup 失败日志 + throw Error                 │                  │
    │   │  重定向 /signin?error=Signups+are+disabled...    │                  │
    │<─────────────────────────┤                           │                  │
    │   └─────────────────────────────────────────────────┘                  │
    │                           │                           │                  │
    │   ┌─────────────────────────────────────────────────┐                  │
    │   │ 【正常继续】signIn 返回 true                      │                  │
    │   │  如果 user 存在，已记录 user.login 日志           │                  │
    │   └─────────────────────────────────────────────────┘                  │
    │                           │                           │                  │
    │                           │ 10. Adapter: getUserByAccount               │
    │                           │     查 accounts 表 (provider+accountId)    │
    │                           │────────────────────────────────────────────>│
    │                           │<────────────────────────────────────────────│
    │                           │                           │                  │
    │                           │     找到账号 → 已绑定 → 跳步骤 13            │
    │                           │     未找到 → 新外部账号 → 继续步骤 11        │
    │                           │                           │                  │
    │                           │ 11. 新账号处理分支         │                  │
    │                           │                           │                  │
    │   ┌─────────────────────────────────────────────────┐                  │
    │   │ 【分支 A】allowDangerousEmailLinking=true        │                  │
    │   │  NextAuth 再查 users 表 (getUserByEmail)         │                  │
    │   │  ├─ 找到 → linkAccount() 绑定账号                │                  │
    │   │  └─ 没找到 → createUser + linkAccount            │                  │
    │   └─────────────────────────────────────────────────┘                  │
    │                           │                           │                  │
    │   ┌─────────────────────────────────────────────────┐                  │
    │   │ 【分支 B】allowDangerousEmailLinking=false(默认)  │                  │
    │   │  NextAuth 不查 users 表，直接 createUser()        │                  │
    │   │  ├─ 正常 → INSERT users + INSERT accounts        │                  │
    │   │  └─ email 冲突 → 抛 OAuthAccountNotLinked        │                  │
    │   │     重定向 /signin?error=OAuthAccountNotLinked   │                  │
    │<─────────────────────────┤                           │                  │
    │   │    ⚠ signIn 已记 login 日志，但实际登录失败        │                  │
    │   └─────────────────────────────────────────────────┘                  │
    │                           │                           │                  │
    │                           │ 12. 账号绑定/创建成功       │                  │
    │                           │                           │                  │
    │                           │ 13. jwt 回调               │                  │
    │                           │     token.user = {...}     │                  │
    │                           │                           │                  │
    │                           │ 14. session 回调           │                  │
    │                           │     session.user = token.user               │
    │                           │                           │                  │
    │                           │ 15. Set-Cookie: next-auth.session-token     │
    │<─────────────────────────│     重定向到 callbackUrl   │                  │
    │                           │                           │                  │
    │ 16. 访问受保护页面         │                           │                  │
    │─────────────────────────>│                           │                  │
    │                           │                           │                  │
    │                           │ 17. getServerAuthSession() │                  │
    │                           │     从 Cookie 解析 JWT     │                  │
    │                           │     构建 tRPC Context      │                  │
```

**时序图关键标注（按代码事实）：**
- **两次 users 表查询**：步骤 9（signIn 回调，应用层主动查）和步骤 11A（NextAuth 内部，仅当允许邮箱关联时）
- **三次查询的不同对象**：步骤 9 查 `users`（按 email），步骤 10 查 `accounts`（按 provider+accountId），步骤 11A 再查 `users`（按 email）
- **日志不一致点**：步骤 9 中如果 email 已存在会记录 `user.login`，但步骤 11B 中可能因 email 冲突而失败
- **错误回退路径**：两种异常分支都重定向到 `/signin?error=xxx`，触发 OAuthAutoRedirect 的逃生通道
