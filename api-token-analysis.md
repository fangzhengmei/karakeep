# Karakeep API Token 系统分析

## 概述

Karakeep 使用 API Key（非 JWT/OAuth）作为外部脚本和客户端的身份认证凭证。系统采用 `ak2_{keyId}_{secret}` 格式，服务端仅存储 keyId 和 secret 的 SHA256 哈希值，明文 key 仅在创建/重新生成时返回给用户一次。整体链路涉及签发、权限收敛、调用注入、撤销回收四大环节。

**核心发现（修正后）**：tRPC 和 REST 两条入口**共享相同的 scope 检查逻辑**。REST 路由通过 tRPC caller 间接触发 `createScopedAuthedProcedure` 的 scope 检查，不存在绕过。

---

## 一、Token 签发流程

### 1.1 签发入口

系统提供两条签发通道，均定义在 `packages/trpc/routers/apiKeys.ts`：

| 通道 | 方法 | 认证方式 | 使用场景 |
|------|------|---------|---------|
| `create` | `sessionProcedure` + `mutation` | Web Session | Web 界面创建（需登录） |
| `exchange` | `publicProcedure` + `mutation` | email + password | 浏览器扩展/移动端"自制 OAuth" |

**exchange 通道的限速保护：** 15 分钟内最多 10 次请求（`createRateLimitMiddleware`）。

### 1.2 exchange 的实际使用边界

`exchange` 是为**无法使用 Web Session** 的客户端设计的"用户名密码换 API Key"机制。各客户端的使用情况：

| 客户端 | 是否使用 exchange | 替代方式 |
|--------|------------------|---------|
| **Web 端** | ❌ 不使用 | Web 界面调用 `apiKeys.create`（`sessionProcedure`，需登录） |
| **浏览器扩展** | ✅ 使用（email+password） | 也支持直接粘贴 API Key（`apiKeys.validate`） |
| **移动端** | ✅ 使用（email+password） | 也支持直接粘贴 API Key（`apiKeys.validate`） |
| **CLI** | ❌ 不使用 | 用户手动从 Web 界面复制 key 到配置文件 |

代码注释明确说明：`// Exchange the username and password with an API key. Homemade oAuth. This is used by the extension.`

**移动端使用细节**（`apps/mobile/app/signin.tsx`）：
- 提供两种登录方式：Password（调用 exchange）和 API Key（直接粘贴调用 validate）
- exchange 调用时 `keyName` 格式：`Mobile App: (${randStr})`

**浏览器扩展使用细节**（`apps/browser-extension/src/SignInPage.tsx`）：
- 同样提供两种登录方式
- exchange 调用时 `keyName` 格式：`Browser extension: (${randStr})`

### 1.3 Key 生成算法

核心逻辑在 `packages/trpc/auth.ts` 的 `generateApiKeySecret()`：

```
plain = "ak2" + "_" + keyId + "_" + secret
```

| 字段 | 生成方式 | 长度 |
|------|---------|------|
| `keyId` | `randomBytes(10).toString("hex")` | 20 hex 字符 |
| `secret` | `randomBytes(16).toString("hex")` | 32 hex 字符 |
| `keyHash` | `sha256(secret).digest("base64")` | — |

数据库存储：`keyId`（唯一索引）+ `keyHash`（SHA256），**不存储明文 secret**。

### 1.4 数据库 Schema

定义在 `packages/db/schema.ts` 的 `apiKeys` 表（`apiKey` 表名）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | text PK | 内部记录 ID |
| `name` | text | 用户自定义名称（同用户下唯一） |
| `createdAt` | integer (timestamp) | 创建时间 |
| `lastUsedAt` | integer (timestamp) | 最近使用时间（10 分钟节流更新） |
| `keyId` | text UNIQUE | 用于查找的公开标识 |
| `keyHash` | text | secret 的 SHA256 哈希 |
| `scopes` | text (JSON) | 授权范围数组 |
| `userId` | text FK | 所属用户（`onDelete: cascade`） |

### 1.5 V1/V2 兼容性

`authenticateApiKey` 中存在两套验证逻辑：

- **V1 (`ak1_`)**：使用 bcrypt 哈希（`$2a$10$...`），比较 `bcrypt.compare(keySecret, hash)`
- **V2 (`ak2_`)**：使用 SHA256，直接比较 `sha256(keySecret).digest("base64") == hash`

V2 的优势：性能更高（bcrypt 计算量大），哈希不可反向。

---

## 二、权限收敛与范围裁剪机制

### 2.1 Scope 模型

定义在 `packages/shared/types/apiKeys.ts`，采用 **资源 + 访问级别** 的二元结构：

**普通资源范围：**

```
{resource}:{access}
```

| 资源 | 说明 |
|------|------|
| `assets` | 文件/媒体资产 |
| `backups` | 备份管理 |
| `bookmarks` | 书签 |
| `feeds` | RSS 订阅 |
| `highlights` | 高亮标注 |
| `lists` | 列表 |
| `prompts` | AI 提示词 |
| `rules` | 自动化规则 |
| `tags` | 标签 |
| `users` | 用户信息 |
| `webhooks` | Webhook |
| `importSessions` | 导入会话（UI 隐藏） |
| `subscriptions` | 订阅（UI 隐藏） |

**访问级别：**

| 级别 | 说明 |
|------|------|
| `read` | 只读 |
| `readwrite` | 读写（隐含包含 read） |

**管理员范围：**

```
admin:{resource}:{access}
```

| 资源 | 说明 |
|------|------|
| `admin:bookmarks` | 管理员书签操作 |
| `admin:jobs` | 后台任务管理 |
| `admin:system` | 系统级操作 |
| `admin:users` | 用户管理 |

**完全访问：**

- `fullaccess`：一个特殊 scope，包含所有资源的所有操作，**默认签发时的默认值**。

### 2.2 Scope 授权判定逻辑

核心函数 `apiKeyScopesGrantScope(grantedScopes, requiredScope)`（`packages/shared/types/apiKeys.ts:91`）：

```
1. 如果 grantedScopes 包含 "fullaccess" → 直接授权
2. 如果 grantedScopes 包含 requiredScope → 授权
3. 如果 requiredScope 是 ":read"，检查是否有对应的 ":readwrite" → 授权
4. 否则 → 拒绝
```

**关键设计：** `readwrite` 权限隐含 `read` 权限，即"包含式"授权。

---

## 三、REST 入口鉴权流程深度分析

### 3.1 整体架构

REST 入口的鉴权是一个**多层级、多中间件协同**的流程，最终通过 tRPC caller 间接触发 scope 检查。

```
HTTP 请求 (REST)
    │
    ▼
[Step 1] Next.js Route Handler
    │  apps/web/app/api/[[...route]]/route.ts
    │  nextAuth 中间件 → createContextFromRequest(req)
    ▼
[Step 2] Token 解析与上下文创建
    │  apps/web/server/api/client.ts
    │  → Authorization: Bearer {key} 提取
    │  → authenticateApiKey(key, db)
    │    ✅ 成功 → ctx.auth = { type: "apiKey", keyId, scopes }
    │    ❌ 失败 → 回退到 session 认证
    ▼
[Step 3] Hono 中间件层
    │  packages/api/middlewares/auth.ts
    │  → authMiddleware: 检查 ctx.user != null
    │  → ✅ 通过：c.set("api", createCaller(c.get("ctx")))
    │       【关键】创建 tRPC caller，绑定当前上下文
    ▼
[Step 4] REST 路由处理
    │  packages/api/routes/bookmarks.ts
    │  → 调用 c.var.api.bookmarks.getBookmarks(params)
    │       【关键】通过 tRPC caller 调用 procedure
    ▼
[Step 5] tRPC Procedure 层（Scope 校验生效处）
    │  packages/trpc/routers/bookmarks.ts
    │  → bookmarksProcedure = createScopedAuthedProcedure("bookmarks")
    │  → Scope 检查中间件执行
    │    • auth.type === "session" → 直接放行
    │    • auth.type === "apiKey" → apiKeyScopesGrantScope 检查
    ▼
[Step 6] 业务逻辑执行
```

### 3.2 Route 层中间件详解

**authMiddleware**（`packages/api/middlewares/auth.ts:24`）是所有 REST 业务路由的前置中间件：

```ts
export const authMiddleware = createMiddleware<{
  Variables: {
    ctx: AuthedContext;
    api: ReturnType<typeof createCaller>;
  };
}>(async (c, next) => {
  if (!c.var.ctx || !c.var.ctx.user || c.var.ctx.user === null) {
    throw new HTTPException(401, { message: "Unauthorized" });
  }
  // 【关键】创建 tRPC caller，绑定当前上下文（包含 auth.type 和 scopes）
  c.set("api", createCaller(c.get("ctx")));
  await next();
});
```

**核心作用：**
1. 验证用户已认证（`ctx.user != null`）
2. 创建 tRPC caller 并绑定到 `c.var.api`
3. caller 携带完整的上下文信息，包括 `auth.type` 和 `auth.scopes`

### 3.3 createCaller 到 tRPC procedure 的调用过程

`createCaller` 是 tRPC 官方提供的 `createCallerFactory`（`packages/trpc/index.ts:92`）：

```ts
export const createCallerFactory = t.createCallerFactory;
```

**调用链详解：**

1. **创建 caller**：`createCaller(ctx)` 创建一个代理对象，可直接调用 router 上的 procedure
2. **调用 procedure**：`c.var.api.bookmarks.getBookmarks(params)` 触发以下流程：
   ```
   caller.bookmarks.getBookmarks(params)
       ↓
   tRPC 内部调用 router.bookmarks.getBookmarks
       ↓
   执行 procedure 的中间件链（从外到内）
       ↓
   1. createScopedAuthedProcedure("bookmarks") → Scope 检查
   2. createBookmarksQueriedMiddleware() → 埋点
   3. 输入验证 (zod)
   4. 业务逻辑
   ```

**关键代码证据**（`packages/trpc/routers/bookmarks.ts:70, 951`）：

```ts
const bookmarksProcedure = createScopedAuthedProcedure("bookmarks");

export const bookmarksAppRouter = router({
  getBookmarks: bookmarksProcedure
    .use(createBookmarksQueriedMiddleware())
    .input(zGetBookmarksRequestSchema)
    .output(zGetBookmarksResponseSchema)
    .query(async ({ input, ctx }) => {
      // 业务逻辑
    }),
  // ... 所有其他 procedure 都使用 bookmarksProcedure
});
```

### 3.4 Scope 校验的生效层级

**Scope 校验在 tRPC procedure 层生效**，无论通过 tRPC 直接调用还是通过 REST caller 间接调用，都会执行相同的检查：

```
createScopedAuthedProcedure("bookmarks")
    │
    ▼
中间件执行逻辑：
  if (opts.ctx.auth.type !== "apiKey") {
    return opts.next();  // session 用户直接放行
  }
  
  // API Key 用户需要检查 scope
  const access = opts.type === "query" ? "read" : "readwrite";
  const scope = `${resource}:${access}`;
  
  if (!apiKeyScopesGrantScope(opts.ctx.auth.scopes, scope)) {
    throw new TRPCError({
      code: "FORBIDDEN",
      message: `API key is missing required scope: ${scope}`,
    });
  }
  
  return opts.next();
```

**重要结论**：
- ✅ REST 路由通过 `c.var.api.*` 调用时，**必然触发 tRPC procedure 的 scope 检查**
- ✅ tRPC 和 REST 两条入口**共享完全相同的 scope 检查逻辑**
- ❌ 不存在"REST 路由绕过 scope 检查"的情况

### 3.5 apiKeyScopeMiddleware 的作用

`apiKeyScopeMiddleware`（`packages/api/middlewares/apiKeyScopes.ts`）是**额外的冗余检查**，仅在 3 个端点显式添加：

| 端点 | 所需 scope | 说明 |
|------|-----------|------|
| `POST /bookmarks/singlefile` | `assets:readwrite` + `bookmarks:readwrite` | 上传单文件 |
| `POST /assets` | `assets:readwrite` | 上传资产 |
| `GET /assets/:assetId` | `assets:read` | 获取资产 |

**为什么是冗余的？**
- 这些路由最终也会通过 `c.var.api.*` 调用 tRPC procedure
- tRPC procedure 已经执行了相同的 scope 检查
- `apiKeyScopeMiddleware` 只是提前在路由层做了一次相同的检查

**可能的设计意图**：
1. 提前拒绝无效请求，减少不必要的 tRPC 调用
2. 在路由层明确标注权限要求，提高代码可读性

---

## 四、tRPC 与 REST 两条入口的异同点

### 4.1 对比表

| 维度 | tRPC 路径 (`/api/trpc/*`) | REST 路径 (`/api/v1/*`) |
|------|---------------------------|-------------------------|
| **入口 URL** | `/api/trpc/{router}.{procedure}` | `/api/v1/{resource}` |
| **上下文创建** | `createContextFromRequest` 统一处理 | `createContextFromRequest` 统一处理 |
| **认证方式** | Bearer Token / Session Cookie | Bearer Token / Session Cookie |
| **用户认证检查** | tRPC 内部 `authedProcedure` | Hono `authMiddleware` |
| **tRPC caller 创建** | tRPC 适配器内部创建 | `authMiddleware` 显式创建 |
| **Scope 检查时机** | procedure 中间件直接执行 | 通过 caller 间接触发 procedure 中间件 |
| **Scope 检查逻辑** | `createScopedAuthedProcedure` | `createScopedAuthedProcedure`（完全相同） |
| **路由级 Scope 检查** | 无（procedure 层已覆盖） | `apiKeyScopeMiddleware`（仅 3 个端点，冗余） |
| **Session 用户** | 不受 scope 限制 | 不受 scope 限制 |
| **API Key 用户** | 所有端点强制 scope 检查 | 所有端点强制 scope 检查（通过 caller） |
| **Bearer 失败回退** | 静默回退到 session | 静默回退到 session |
| **支持的客户端** | Web 端、CLI、扩展、移动端 | 扩展、移动端、第三方脚本 |

### 4.2 核心相同点

1. **共享认证链路**：两条路径都经过 `createContextFromRequest` 处理 Bearer Token 和 Session
2. **共享 Scope 检查**：最终都执行 `createScopedAuthedProcedure` 的相同逻辑
3. **共享业务逻辑**：调用相同的 tRPC procedure 执行业务操作
4. **相同回退行为**：Bearer Token 失败时都静默回退到 Session 认证
5. **相同撤销机制**：API Key 撤销对两条路径同时生效

### 4.3 核心不同点

1. **调用方式**：
   - tRPC：客户端直接调用 procedure，类型安全
   - REST：客户端通过 HTTP 方法调用，再通过 caller 转发到 procedure

2. **中间件层级**：
   - tRPC：所有逻辑在 tRPC 内部完成
   - REST：先经过 Hono 中间件，再进入 tRPC 层

3. **错误返回格式**：
   - tRPC：标准 tRPC 错误格式（JSON-RPC）
   - REST：HTTP 状态码 + JSON 响应

4. **冗余检查**：
   - tRPC：无冗余检查
   - REST：3 个端点有 `apiKeyScopeMiddleware` 冗余检查

---

## 五、Bearer 失败时的会话回退机制

### 5.1 回退条件

核心代码在 `apps/web/server/api/client.ts` 的 `createContextFromRequest`：

```ts
export async function createContextFromRequest(req: Request) {
  const authorizationHeader = req.headers.get("Authorization");
  if (authorizationHeader && authorizationHeader.startsWith("Bearer ")) {
    const token = authorizationHeader.split(" ")[1];
    try {
      const authResult = await authenticateApiKey(token, db);
      return {
        user: authResult.user,
        auth: { type: "apiKey", keyId: ..., scopes: ... },
        db, req: { ip },
      };
    } catch {
      // API key 验证失败 → 静默吞掉异常，回退到 cookie session 认证
    }
  }
  return createContext(db, ip); // 走 session 路径
}
```

**触发回退的场景：**
1. Bearer token 格式错误（不是 `ak{1,2}_{keyId}_{secret}` 格式）
2. keyId 在数据库中不存在
3. secret 哈希比对失败
4. 其他 `authenticateApiKey` 抛出的异常

### 5.2 回退路径

```
Bearer token 验证失败
    ↓
catch 块静默吞掉异常
    ↓
调用 createContext(db, ip)
    ↓
createContext 调用 getServerAuthSession()
    ↓
从 Next.js cookie 中读取 session（如果有）
    ↓
session 有效 → ctx.auth = { type: "session" }
session 无效 → ctx.auth = null，后续 authMiddleware 返回 401
```

### 5.3 安全影响（修正后）

**之前的错误结论**：回退会绕过 scope 检查

**修正后的正确结论**：

1. **Scope 检查不会被绕过**：
   - 回退到 session 后，`auth.type = "session"`
   - `createScopedAuthedProcedure` 对 session 用户直接放行（设计如此）
   - 这不是"绕过"，而是 session 用户本来就不受 scope 限制

2. **混淆代理攻击风险仍然存在**：
   - 攻击者构造带有**无效 Bearer token** 的请求
   - 如果用户已登录（有有效 session cookie），请求会以 session 身份执行
   - 攻击者可以利用用户的 session 权限执行操作
   - 但这是 Session 认证的固有风险，不是 scope 检查绕过

3. **权限变化对比**：

| 场景 | auth.type | 权限模型 |
|------|-----------|---------|
| Bearer token 有效 | `apiKey` | 受 scope 限制 |
| Bearer token 无效 + 有效 session | `session` | 完整用户权限（不受 scope 限制） |
| Bearer token 无效 + 无 session | `null` | 401 未认证 |

4. **信息泄露防护（双刃剑）**：
   - 静默回退不返回 401，攻击者无法区分：
     - "API Key 不存在/无效"
     - "API Key 有效但权限不足"
     - "请求根本没有 API Key"
   - 防止 API Key 存在性探测，但增加调试难度

5. **对纯 API 客户端的影响**：
   - 没有 cookie 的纯 API 客户端（如 CLI、服务器脚本）
   - Bearer 失败后 `getServerAuthSession()` 返回 null
   - 最终 `authMiddleware` 返回 401
   - 错误信息不明确，用户无法知道是 key 无效还是其他问题

---

## 六、撤销与回收策略

### 6.1 撤销操作

**硬删除（Hard Delete）**

`revoke` 端点（`apiKeys.ts:89`）执行数据库硬删除：

```sql
DELETE FROM apiKey WHERE id = ? AND userId = ?
```

特点：
- 删除后记录完全消失，`validate` 立即失败
- 操作不可逆
- 需要 `sessionProcedure`（API Key 自身无法撤销自己）

### 6.2 重新生成（Regenerate）

`regenerate` 端点（`apiKeys.ts:61`）是"撤销+重发"：

1. 根据 `id` + `userId` 查找已有记录
2. 生成新的 `keyId` + `keyHash`
3. 更新该记录
4. **旧 keyId 被替换，旧 key 立即失效**

数据库操作是 `UPDATE` 而非 `DELETE + INSERT`，保持 `id` 不变，保留 name/scopes/createdAt。

### 6.3 级联回收

用户删除时的级联：`apiKeys.userId` 设置了 `references(() => users.id, { onDelete: "cascade" })`，用户删除时所有关联 API Key 自动清理。

### 6.4 运行时回收 — 请求时实时验证

API Key **无会话缓存**，每次请求都执行完整验证链：

```
HTTP 请求 → Authorization: Bearer {key}
  → createContextFromRequest (web server)
  → authenticateApiKey(key, db)
  → parseApiKey (格式校验)
  → DB 查询 keyId → 未找到则抛错
  → 哈希比对 (V1: bcrypt / V2: sha256) → 不匹配则抛错
  → 返回 user + apiKey.scopes
```

这意味着 **撤销操作即时生效**，无需等待 token 过期或缓存刷新。

### 6.5 使用追踪

`lastUsedAt` 字段以 10 分钟为节流周期更新（`auth.ts:140`）：

```ts
const tenMinutesAgo = new Date(Date.now() - 10 * 60 * 1000);
if (!apiKey.lastUsedAt || apiKey.lastUsedAt < tenMinutesAgo) {
  // fire-and-forget, 不阻塞认证响应
  database.update(apiKeys).set({ lastUsedAt: new Date() })...
}
```

**策略：** 更新为"即发即忘"（async 不 await），避免数据库写入阻塞认证流程。

### 6.6 撤销策略总结

| 操作 | 效果 | 可恢复 |
|------|------|--------|
| `revoke` | 硬删除，立即失效 | 否 |
| `regenerate` | keyId 替换，旧 key 失效 | 否 |
| 用户删除 | 级联删除所有 API Key | 否 |

---

## 七、外部脚本调用系统完整链路

### 7.1 tRPC 路径完整时序

```
[外部脚本]
    │
    │ 1. POST /api/trpc/bookmarks.getBookmarks
    │    Authorization: Bearer ak2_abc123_def456
    │    Body: { "input": { ... } }
    │
    ▼
[Next.js Route Handler]
    │  route.ts → nextAuth 中间件
    │  → createContextFromRequest(rawRequest)
    │
    ▼
[Token 解析]
    │  提取 Bearer token → authenticateApiKey(key, db)
    │    ├── parseApiKey: 校验格式 (ak2_xxx_yyy)
    │    ├── DB 查询 keyId
    │    ├── SHA256 比对 secret
    │    ├── 更新 lastUsedAt (10min 节流)
    │    └── ✅ 返回 { user, apiKey: { keyId, scopes } }
    │       ❌ 失败 → 回退到 session 认证
    │
    ▼
[上下文注入]
    │  ctx = { user, auth: { type: "apiKey", keyId, scopes }, db }
    │
    ▼
[tRPC 适配器]
    │  trpcServer → fetchRequestHandler
    │  → 从 c.var.ctx 获取上下文
    │
    ▼
[tRPC 调用链]
    │  router.bookmarks.getBookmarks
    │    ├── bookmarksProcedure = createScopedAuthedProcedure("bookmarks")
    │    │   ├── 检查 auth.type === "apiKey" ✓
    │    │   ├── opts.type === "query" → access = "read"
    │    │   ├── scope = "bookmarks:read"
    │    │   └── apiKeyScopesGrantScope 检查 ✓
    │    ├── createBookmarksQueriedMiddleware()
    │    ├── 输入验证 (zod)
    │    └── 执行业务逻辑
    │
    ▼
[响应]
    │ 返回 tRPC 格式响应
```

### 7.2 REST 路径完整时序

```
[外部脚本]
    │
    │ 1. GET /api/v1/bookmarks?limit=20
    │    Authorization: Bearer ak2_abc123_def456
    │
    ▼
[Next.js Route Handler]
    │  → createContextFromRequest
    │  → ✅ API Key 验证成功 → auth.type = "apiKey"
    │
    ▼
[Hono App]
    │  authMiddleware
    │    ├── 检查 ctx.user != null ✓
    │    └── 【关键】c.set("api", createCaller(c.get("ctx")))
    │
    ▼
[REST 路由处理]
    │  GET /bookmarks 路由
    │    ├── zValidator 验证 query 参数
    │    └── 【关键】调用 c.var.api.bookmarks.getBookmarks(searchParams)
    │
    ▼
[tRPC Procedure 层]
    │  caller 触发 bookmarks.getBookmarks procedure
    │    ├── bookmarksProcedure = createScopedAuthedProcedure("bookmarks")
    │    │   ├── 检查 auth.type === "apiKey" ✓
    │    │   ├── opts.type === "query" → access = "read"
    │    │   ├── scope = "bookmarks:read"
    │    │   └── apiKeyScopesGrantScope 检查 ✓
    │    ├── createBookmarksQueriedMiddleware()
    │    ├── 输入验证 (zod)
    │    └── 执行业务逻辑
    │
    ▼
[响应]
    │ 返回 REST 格式响应 (JSON + pagination)
```

### 7.3 Token 获取方式

**方式 A：Web 界面创建（Session 认证）**

```
用户登录 Web → Settings → API Keys → 创建
  → POST /api/trpc/apiKeys.create
  → sessionProcedure（cookie/session 认证）
  → 返回明文 key（仅此一次）
```

**方式 B：浏览器扩展/移动端 Exchange（密码认证）**

```
扩展/移动端首次使用 → 用户输入 email + password
  → POST /api/trpc/apiKeys.exchange
  → publicProcedure（无需前置认证）
  → validatePassword(email, password)
  → 检查 emailVerificationRequired
  → 返回明文 key（存储在本地）
```

**方式 C：直接粘贴 API Key**

```
用户从 Web 界面复制 key → 粘贴到扩展/移动端
  → POST /api/trpc/apiKeys.validate
  → 验证成功后存储在本地
```

**方式 D：CLI 手动配置**

```
用户从 Web 界面复制 key → 运行 `karakeep auth init`
  → 交互输入 serverAddr + apiKey
  → 写入 ~/.config/karakeep/config.json (权限 0o600)
```

---

## 八、安全设计要点

### 8.1 存储安全

- **仅存哈希**：数据库只存 `keyHash`（SHA256），不存明文 secret
- **keyId 分离**：keyId 是可公开标识（用于查找），与 secret 独立生成
- **V1→V2 迁移**：V1 使用 bcrypt（计算慢，更抗暴力），V2 使用 SHA256（性能好）

### 8.2 传输安全

- **Authorization: Bearer** 标准格式，支持 HTTPS 下的 TLS 传输
- CORS 配置允许 `Authorization` 和 `Content-Type` 头
- Web Server 中 API Key 失败时静默回退到 session，不泄露 key 是否有效

### 8.3 权限模型（修正后）

- **统一的 Scope 检查**：tRPC 和 REST 路径共享相同的 `createScopedAuthedProcedure` 检查
- **无绕过可能**：所有业务 procedure 都被 scope 中间件包装，REST 通过 caller 间接触发
- **默认签发 `fullaccess`**：向后兼容，但支持显式指定细粒度 scope
- **`sessionProcedure`**：防止 API Key 管理 API Key（防止提权递归）
- **管理员 scope 命名空间隔离**：`admin:` 前缀

### 8.4 回退机制的安全权衡

| 设计选择 | 优点 | 风险 |
|---------|------|------|
| Bearer 失败静默回退 | 同一端点同时支持 API Key 和 Session，用户体验好 | 混淆代理攻击（利用用户 session） |
| Session 用户不受 scope 限制 | Web 界面使用简单 | 回退时权限从"受 scope 限制"变为"完整权限" |
| 不返回 401 区分 key 无效/权限不足 | 防止 key 存在性探测 | 调试困难，用户无法知道 key 是否正确 |

### 8.5 撤销即时性

- **无缓存**：每次请求实时查库验证，撤销即时生效
- **硬删除**：revoke 是 DELETE 操作，记录彻底消失
- **级联删除**：用户删除自动清理所有 key

### 8.6 限速保护

| 端点 | 窗口 | 最大请求 |
|------|------|---------|
| `exchange` | 15 分钟 | 10 |
| `validate` | 1 分钟 | 30 |
| 全局 public | 1 分钟 | 1000 |
| 全局 authed | 1 分钟 | 3000 |
| `assets.upload` | 1 分钟 | 30 |
| `bookmarks.createBookmark` | 1 分钟 | 30 |

---

## 九、代码索引

| 功能 | 文件路径 | 关键符号 |
|------|---------|---------|
| Key 生成/验证 | `packages/trpc/auth.ts` | `generateApiKey`, `authenticateApiKey`, `regenerateApiKey`, `parseApiKey` |
| Key 管理路由 | `packages/trpc/routers/apiKeys.ts` | `apiKeysAppRouter` (create/revoke/regenerate/list/exchange/validate) |
| Scope 类型定义 | `packages/shared/types/apiKeys.ts` | `API_KEY_SCOPE_RESOURCES`, `apiKeyScopesGrantScope`, `getApiKeyScope` |
| tRPC Scope 中间件 | `packages/trpc/index.ts` | `createScopedAuthedProcedure`, `sessionProcedure`, `rejectApiKeyAuth`, `createCallerFactory` |
| Hono 认证中间件 | `packages/api/middlewares/auth.ts` | `authMiddleware`（创建 tRPC caller） |
| Hono Scope 中间件 | `packages/api/middlewares/apiKeyScopes.ts` | `apiKeyScopeMiddleware`（冗余检查） |
| Web 上下文注入 | `apps/web/server/api/client.ts` | `createContextFromRequest`（Bearer 回退逻辑） |
| Web Session 认证 | `apps/web/server/auth.ts` | `getServerAuthSession`, NextAuth 配置 |
| 数据库 Schema | `packages/db/schema.ts` | `apiKeys` table |
| 浏览器扩展登录 | `apps/browser-extension/src/SignInPage.tsx` | exchange 调用 |
| 移动端登录 | `apps/mobile/app/signin.tsx` | exchange 调用 |
| 浏览器扩展 tRPC | `apps/browser-extension/src/utils/trpc.ts` | `initializeClients`, Bearer header 注入 |
| CLI 配置 | `apps/cli/src/commands/auth.ts`, `apps/cli/src/lib/config.ts` | `auth init`, `~/.config/karakeep/config.json` |
| tRPC 路由示例 | `packages/trpc/routers/bookmarks.ts` | `bookmarksProcedure`, `bookmarksAppRouter` |
| REST 路由示例 | `packages/api/routes/bookmarks.ts` | `c.var.api.bookmarks.getBookmarks` 调用 |
| 测试用例 | `packages/trpc/routers/apiKeys.test.ts` | 完整生命周期/scope 强制/兼容性测试 |
