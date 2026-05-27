# Karakeep API Token 系统分析

## 概述

Karakeep 使用 API Key（非 JWT/OAuth）作为外部脚本和客户端的身份认证凭证。系统采用 `ak2_{keyId}_{secret}` 格式，服务端仅存储 keyId 和 secret 的 SHA256 哈希值，明文 key 仅在创建/重新生成时返回给用户一次。整体链路涉及签发、权限收敛、调用注入、撤销回收四大环节。

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

### 2.3 tRPC 与 REST 两条入口的 Scope 校验差异

#### 2.3.1 tRPC 路径：强制自动 Scope 检查

tRPC 所有业务 router 都使用 `createScopedAuthedProcedure(resource)` 包装，**scope 检查是强制自动注入的**：

| 组件 | 机制 | 代码位置 |
|------|------|---------|
| `createScopedAuthedProcedure(resource)` | 根据 operation type 自动映射 access：query→read，mutation→readwrite | `trpc/index.ts:177` |
| `createAdminScopedProcedure(resource)` | scope 检查 + isAdmin 角色检查 | `trpc/index.ts:199` |
| `sessionProcedure` | 完全拒绝 API Key（`rejectApiKeyAuth`），用于 API Key 管理端点 | `trpc/index.ts:197` |

**tRPC scope 检查流程：**
```
请求到达 procedure → 检查 auth.type
  → auth.type === "session"：直接放行（无 scope 限制）
  → auth.type === "apiKey"：
      → 根据 procedure type 确定 access
      → 构造 scope = "{resource}:{access}"
      → apiKeyScopesGrantScope 检查
      → 不通过 → TRPCError(FORBIDDEN)
```

**所有业务 router 均已接入**（通过 grep 验证）：
`bookmarks`, `assets`, `backups`, `feeds`, `highlights`, `lists`, `prompts`, `rules`, `tags`, `users`, `webhooks`, `importSessions`, `subscriptions`

#### 2.3.2 REST 路径：仅特定端点显式 Scope 检查

REST 路径的 scope 检查**不是全局的**，只有特定端点显式添加 `apiKeyScopeMiddleware`：

| 层级 | 机制 | 代码位置 |
|------|------|---------|
| `authMiddleware` | 全局中间件，仅检查 `ctx.user != null`，**不检查 scope** | `api/middlewares/auth.ts:24` |
| `apiKeyScopeMiddleware(resource, access)` | 仅在特定路由上手动添加，检查 apiKey 的 scope | `api/middlewares/apiKeyScopes.ts:14` |

**已发现的 REST scope 检查端点：**

| 端点 | 所需 scope | 代码位置 |
|------|-----------|---------|
| `POST /bookmarks/singlefile` | `assets:readwrite` + `bookmarks:readwrite` | `api/routes/bookmarks.ts:113` |
| `POST /assets` | `assets:readwrite` | `api/routes/assets.ts:17` |
| `GET /assets/:assetId` | `assets:read` | `api/routes/assets.ts:43` |

**重要发现：** 大多数 REST 端点（如 `GET /bookmarks`, `POST /bookmarks`, `GET /bookmarks/search`, `GET /bookmarks/check-url` 等）**没有显式的 scope 检查**。只要通过 `authMiddleware`（即用户存在），无论 auth.type 是 apiKey 还是 session，都可以访问。

#### 2.3.3 Scope 检查路径对比

| 维度 | tRPC 路径 | REST 路径 |
|------|----------|----------|
| Scope 检查时机 | Procedure 定义时自动注入 | 特定路由手动添加 |
| 覆盖范围 | 所有业务端点 | 仅 3 个端点有检查 |
| Access 映射 | 自动根据 query/mutation 映射 | 显式指定 |
| Session 用户 | 不受 scope 限制 | 不受 scope 限制 |
| API Key 用户 | 所有端点强制 scope 检查 | 大部分端点无 scope 检查 |

### 2.4 Scope 边界总结

| 层级 | 机制 | 代码位置 |
|------|------|---------|
| 签发默认 | 默认 `fullaccess`，可显式指定 | `apiKeys.ts:48`, `apiKeys.ts:185` |
| tRPC 资源 | `createScopedAuthedProcedure` 自动映射 read/readwrite，**强制检查** | `trpc/index.ts:177` |
| tRPC 管理员 | `createAdminScopedProcedure` + 角色检查 | `trpc/index.ts:199` |
| tRPC Key 管理 | `sessionProcedure` 禁止 apiKey 调用 | `trpc/index.ts:197` |
| REST 认证 | `authMiddleware` 仅检查用户存在，**不检查 scope** | `api/middlewares/auth.ts:24` |
| REST 特定路由 | `apiKeyScopeMiddleware` 显式指定，**仅 3 个端点** | `api/middlewares/apiKeyScopes.ts:14` |
| 全局拒绝 | `rejectApiKeyAuth` 中间件 | `trpc/index.ts:163` |

---

## 三、撤销与回收策略

### 3.1 撤销操作

**硬删除（Hard Delete）**

`revoke` 端点（`apiKeys.ts:89`）执行数据库硬删除：

```sql
DELETE FROM apiKey WHERE id = ? AND userId = ?
```

特点：
- 删除后记录完全消失，`validate` 立即失败
- 操作不可逆
- 需要 `sessionProcedure`（API Key 自身无法撤销自己）

### 3.2 重新生成（Regenerate）

`regenerate` 端点（`apiKeys.ts:61`）是"撤销+重发"：

1. 根据 `id` + `userId` 查找已有记录
2. 生成新的 `keyId` + `keyHash`
3. 更新该记录
4. **旧 keyId 被替换，旧 key 立即失效**

数据库操作是 `UPDATE` 而非 `DELETE + INSERT`，保持 `id` 不变，保留 name/scopes/createdAt。

### 3.3 级联回收

用户删除时的级联：`apiKeys.userId` 设置了 `references(() => users.id, { onDelete: "cascade" })`，用户删除时所有关联 API Key 自动清理。

### 3.4 运行时回收 — 请求时实时验证

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

### 3.5 使用追踪

`lastUsedAt` 字段以 10 分钟为节流周期更新（`auth.ts:140`）：

```ts
const tenMinutesAgo = new Date(Date.now() - 10 * 60 * 1000);
if (!apiKey.lastUsedAt || apiKey.lastUsedAt < tenMinutesAgo) {
  // fire-and-forget, 不阻塞认证响应
  database.update(apiKeys).set({ lastUsedAt: new Date() })...
}
```

**策略：** 更新为"即发即忘"（async 不 await），避免数据库写入阻塞认证流程。

### 3.6 撤销策略总结

| 操作 | 效果 | 可恢复 |
|------|------|--------|
| `revoke` | 硬删除，立即失效 | 否 |
| `regenerate` | keyId 替换，旧 key 失效 | 否 |
| 用户删除 | 级联删除所有 API Key | 否 |

---

## 四、外部脚本调用系统完整链路

### 4.1 整体架构

```
外部客户端 (CLI / 浏览器扩展 / 移动端 / 第三方脚本)
    │
    │  HTTP 请求
    │  Header: Authorization: Bearer ak2_xxx_yyy
    ▼
Web Server (Next.js API Route)
    │  apps/web/app/api/[[...route]]/route.ts
    │  → nextAuth 中间件 → createContextFromRequest()
    ▼
认证与上下文注入
    │  apps/web/server/api/client.ts
    │  → authenticateApiKey()  [packages/trpc/auth.ts]
    │  → ✅ 成功 → auth.type = "apiKey"
    │  → ❌ 失败 → 静默回退 → auth.type = "session" (如果有有效 cookie)
    ▼
Hono App (packages/api)
    │  → authMiddleware (检查 ctx.user != null)
    │  → REST 路由: apiKeyScopeMiddleware (仅特定端点)
    │  → tRPC 路由: createScopedAuthedProcedure (所有业务端点)
    ▼
业务逻辑执行
```

### 4.2 Bearer 失败时的会话回退机制

#### 4.2.1 回退条件

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

#### 4.2.2 回退路径

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

#### 4.2.3 安全影响

**1. 混淆代理攻击（Confused Deputy）**

如果用户已登录 Web 应用（有有效的 session cookie），攻击者可以构造带有**无效 Bearer token** 的请求：

- 正常场景：`Authorization: Bearer <valid-api-key>` → auth.type = "apiKey" → 受 scope 限制
- 攻击场景：`Authorization: Bearer <invalid-or-expired-key>` → 回退到 session → auth.type = "session" → **不受 scope 限制**

攻击者可以利用用户的 session 权限执行超出 API Key 授权范围的操作。

**2. Scope 检查绕过**

回退到 session 后，`auth.type` 变为 `"session"`，所有 scope 检查中间件都会跳过：

- tRPC 的 `createScopedAuthedProcedure` 检查 `auth.type !== "apiKey"` 时直接放行
- REST 的 `apiKeyScopeMiddleware` 检查 `auth.type !== "apiKey"` 时直接放行

这意味着 scope 边界仅在 Bearer token **验证成功** 时生效。

**3. 信息泄露防护（双刃剑）**

静默回退不返回 401，攻击者无法区分：
- "API Key 不存在/无效"
- "API Key 有效但权限不足"
- "请求根本没有 API Key"

这在一定程度上防止了 API Key 存在性探测，但也增加了调试难度。

**4. 对纯 API 客户端的影响**

对于没有 cookie 的纯 API 客户端（如 CLI、服务器脚本），回退会导致 `getServerAuthSession()` 返回 null，最终 `authMiddleware` 返回 401。这是预期行为，但错误信息不明确。

### 4.3 Token 获取方式

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

### 4.4 Token 注入与传递

**浏览器扩展**（`apps/browser-extension/src/utils/trpc.ts`）：

```ts
httpLink({
  url: `${address}/api/trpc`,
  headers: {
    Authorization: `Bearer ${apiKey}`,
    ...customHeaders,
  },
})
```

**移动端**（类似 tRPC client 注入）。

**CLI**（`apps/cli/src/lib/trpc.ts`）：

```
从 ~/.config/karakeep/config.json 读取 apiKey
→ 创建 tRPC client 时注入 Authorization header
```

**单文件上传（singlefile）**：

```ts
// apps/browser-extension/src/utils/singlefile.ts
headers = { Authorization: `Bearer ${settings.apiKey}` }
fetch(`${apiUrl}/api/v1/bookmarks/singlefile`, { method: "POST", headers, body: formData })
```

### 4.5 完整调用时序图（tRPC 路径）

```
[外部脚本]
    │
    │ 1. GET /api/trpc/bookmarks.list
    │    Authorization: Bearer ak2_abc123_def456
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
[Hono tRPC Adapter]
    │  trpcServer → 从 c.var.ctx 获取上下文
    │
    ▼
[tRPC 调用链]
    │  router.bookmarks.list (createScopedAuthedProcedure("bookmarks"))
    │    ├── 检查 auth.type === "apiKey" ✓
    │    ├── opts.type === "query" → access = "read"
    │    ├── scope = "bookmarks:read"
    │    ├── apiKeyScopesGrantScope(scopes, "bookmarks:read")
    │    │   ├── fullaccess? → 直接通过
    │    │   ├── bookmarks:read? → 通过
    │    │   └── bookmarks:readwrite? → 通过 (隐含 read)
    │    └── 执行业务逻辑
    │
    ▼
[响应]
    │ 返回书签列表数据
```

### 4.6 完整调用时序图（REST 路径，无 scope 检查的端点）

```
[外部脚本]
    │
    │ 1. POST /api/v1/bookmarks
    │    Authorization: Bearer ak2_abc123_def456
    │    Body: { "url": "https://example.com" }
    │
    ▼
[Next.js Route Handler]
    │  → createContextFromRequest
    │  → ✅ API Key 验证成功 → auth.type = "apiKey"
    │
    ▼
[Hono App]
    │  authMiddleware → 检查 ctx.user != null ✓
    │
    ▼
[业务路由]
    │  POST /bookmarks（无 apiKeyScopeMiddleware）
    │  → 直接调用 c.var.api.bookmarks.createBookmark
    │  → ❗ 无 scope 检查，只要认证通过即可访问
    │
    ▼
[响应]
    │ 返回创建的书签
```

---

## 五、安全设计要点

### 5.1 存储安全

- **仅存哈希**：数据库只存 `keyHash`（SHA256），不存明文 secret
- **keyId 分离**：keyId 是可公开标识（用于查找），与 secret 独立生成
- **V1→V2 迁移**：V1 使用 bcrypt（计算慢，更抗暴力），V2 使用 SHA256（性能好）

### 5.2 传输安全

- **Authorization: Bearer** 标准格式，支持 HTTPS 下的 TLS 传输
- CORS 配置允许 `Authorization` 和 `Content-Type` 头
- Web Server 中 API Key 失败时静默回退到 session，不泄露 key 是否有效

### 5.3 权限模型

- **tRPC 路径**：所有业务端点强制 scope 检查，权限边界清晰
- **REST 路径**：大部分端点无 scope 检查，API Key 拥有等同于 session 的权限
- **默认签发 `fullaccess`**：向后兼容，但支持显式指定细粒度 scope
- **`sessionProcedure`**：防止 API Key 管理 API Key（防止提权递归）
- **管理员 scope 命名空间隔离**：`admin:` 前缀

### 5.4 回退机制的安全权衡

| 设计选择 | 优点 | 风险 |
|---------|------|------|
| Bearer 失败静默回退 | 同一端点同时支持 API Key 和 Session，用户体验好 | 混淆代理攻击、scope 检查绕过 |
| Session 用户不受 scope 限制 | Web 界面使用简单 | 回退时绕过 scope 边界 |
| 不返回 401 区分 key 无效/权限不足 | 防止 key 存在性探测 | 调试困难，用户无法知道 key 是否正确 |

### 5.5 撤销即时性

- **无缓存**：每次请求实时查库验证，撤销即时生效
- **硬删除**：revoke 是 DELETE 操作，记录彻底消失
- **级联删除**：用户删除自动清理所有 key

### 5.6 限速保护

| 端点 | 窗口 | 最大请求 |
|------|------|---------|
| `exchange` | 15 分钟 | 10 |
| `validate` | 1 分钟 | 30 |
| 全局 public | 1 分钟 | 1000 |
| 全局 authed | 1 分钟 | 3000 |
| `assets.upload` | 1 分钟 | 30 |

---

## 六、代码索引

| 功能 | 文件路径 | 关键符号 |
|------|---------|---------|
| Key 生成/验证 | `packages/trpc/auth.ts` | `generateApiKey`, `authenticateApiKey`, `regenerateApiKey`, `parseApiKey` |
| Key 管理路由 | `packages/trpc/routers/apiKeys.ts` | `apiKeysAppRouter` (create/revoke/regenerate/list/exchange/validate) |
| Scope 类型定义 | `packages/shared/types/apiKeys.ts` | `API_KEY_SCOPE_RESOURCES`, `apiKeyScopesGrantScope`, `getApiKeyScope` |
| tRPC Scope 中间件 | `packages/trpc/index.ts` | `createScopedAuthedProcedure`, `sessionProcedure`, `rejectApiKeyAuth` |
| Hono Scope 中间件 | `packages/api/middlewares/apiKeyScopes.ts` | `apiKeyScopeMiddleware` |
| Hono 认证中间件 | `packages/api/middlewares/auth.ts` | `authMiddleware`（仅检查用户存在） |
| Web 上下文注入 | `apps/web/server/api/client.ts` | `createContextFromRequest`（Bearer 回退逻辑） |
| Web Session 认证 | `apps/web/server/auth.ts` | `getServerAuthSession`, NextAuth 配置 |
| 数据库 Schema | `packages/db/schema.ts` | `apiKeys` table |
| 浏览器扩展登录 | `apps/browser-extension/src/SignInPage.tsx` | exchange 调用 |
| 移动端登录 | `apps/mobile/app/signin.tsx` | exchange 调用 |
| 浏览器扩展 tRPC | `apps/browser-extension/src/utils/trpc.ts` | `initializeClients`, Bearer header 注入 |
| CLI 配置 | `apps/cli/src/commands/auth.ts`, `apps/cli/src/lib/config.ts` | `auth init`, `~/.config/karakeep/config.json` |
| 测试用例 | `packages/trpc/routers/apiKeys.test.ts` | 完整生命周期/scope 强制/兼容性测试 |
