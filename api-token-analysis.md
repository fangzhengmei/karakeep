# Karakeep API Token 系统分析

## 概述

Karakeep 使用 API Key（非 JWT/OAuth）作为外部脚本和客户端的身份认证凭证。系统采用 `ak2_{keyId}_{secret}` 格式，服务端仅存储 keyId 和 secret 的 SHA256 哈希值，明文 key 仅在创建/重新生成时返回给用户一次。整体链路涉及签发、权限收敛、调用注入、撤销回收四大环节。

---

## 一、Token 签发流程

### 1.1 签发入口

系统提供两条签发通道，均定义在 `packages/trpc/routers/apiKeys.ts`：

| 通道 | 方法 | 认证方式 | 使用场景 |
|------|------|---------|---------|
| `create` | `sessionProcedure` + `mutation` | Web Session | Web 界面创建 |
| `exchange` | `publicProcedure` + `mutation` | email + password | 浏览器扩展"自制 OAuth" |

**exchange 通道的限速保护：** 15 分钟内最多 10 次请求（`createRateLimitMiddleware`）。

### 1.2 Key 生成算法

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

### 1.3 数据库 Schema

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

### 1.4 V1/V2 兼容性

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

### 2.3 tRPC 层的 Scope 检查

在 `packages/trpc/index.ts` 中实现了两层权限收敛：

**(a) `createScopedAuthedProcedure(resource)`** — 普通资源范围：

```
authedProcedure → 检查 auth.type === "apiKey"
  → 若不是 apiKey（即 session），直接放行
  → 若是 apiKey，根据 operation type 自动映射 access：
      query → "read"
      mutation → "readwrite"
  → 构造 scope = "{resource}:{access}"
  → 调用 hasRequiredApiKeyScopes 检查
```

**(b) `createAdminScopedProcedure(resource)`** — 管理员范围：

```
authedProcedure → apiKey scope 检查 → isAdmin 角色检查
```

**(c) `sessionProcedure`** — 完全拒绝 API Key：

```
authedProcedure → rejectApiKeyAuth() 中间件
  → 若 auth.type === "apiKey"，抛出 FORBIDDEN
```

`sessionProcedure` 用于 API Key 管理端点（create/regenerate/revoke/list），防止 API Key 自我管理的递归权限。

### 2.4 Hono REST 层的 Scope 检查

`packages/api/middlewares/apiKeyScopes.ts` 的 `apiKeyScopeMiddleware(resource, access)`：

```
auth.type !== "apiKey" → 放行（session 用户不受 scope 限制）
auth.type === "apiKey" → 构造 scope → apiKeyScopesGrantScope 检查
  → 失败则抛出 HTTP 403
```

与 tRPC 不同，REST 路由**显式指定 access 级别**（不是根据 HTTP method 推断），例如：

```ts
app.post("/", apiKeyScopeMiddleware("assets", "readwrite"), ...)
app.get("/", apiKeyScopeMiddleware("bookmarks", "read"), ...)
```

### 2.5 Scope 边界总结

| 层级 | 机制 | 代码位置 |
|------|------|---------|
| 签发默认 | 默认 `fullaccess`，可显式指定 | `apiKeys.ts:48`, `apiKeys.ts:185` |
| tRPC 资源 | `createScopedAuthedProcedure` 自动映射 read/readwrite | `trpc/index.ts:177` |
| tRPC 管理员 | `createAdminScopedProcedure` + 角色检查 | `trpc/index.ts:199` |
| tRPC Key 管理 | `sessionProcedure` 禁止 apiKey 调用 | `trpc/index.ts:197` |
| REST 路由 | `apiKeyScopeMiddleware` 显式指定 | `api/middlewares/apiKeyScopes.ts:14` |
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
外部客户端 (CLI / 浏览器扩展 / 第三方脚本)
    │
    │  HTTP 请求
    │  Header: Authorization: Bearer ak2_xxx_yyy
    ▼
Web Server (Next.js API Route)
    │  apps/web/app/api/[[...route]]/route.ts
    │  → createContextFromRequest()
    ▼
认证与上下文注入
    │  apps/web/server/api/client.ts
    │  → authenticateApiKey()  [packages/trpc/auth.ts]
    ▼
Hono App (packages/api)
    │  → authMiddleware / apiKeyScopeMiddleware
    ▼
tRPC Router (packages/trpc/routers)
    │  → createScopedAuthedProcedure 做 scope 检查
    ▼
业务逻辑执行
```

### 4.2 Token 获取方式

**方式 A：Web 界面创建（Session 认证）**

```
用户登录 Web → Settings → API Keys → 创建
  → POST /api/trpc/apiKeys.create
  → sessionProcedure（cookie/session 认证）
  → 返回明文 key（仅此一次）
```

**方式 B：浏览器扩展 Exchange（密码认证）**

```
扩展首次使用 → 用户输入 email + password
  → POST /api/trpc/apiKeys.exchange
  → publicProcedure（无需前置认证）
  → validatePassword(email, password)
  → 检查 emailVerificationRequired
  → 返回明文 key（扩展存储在 chrome.storage.sync）
```

**方式 C：CLI 手动配置**

```
用户从 Web 界面复制 key → 运行 `karakeep auth init`
  → 交互输入 serverAddr + apiKey
  → 写入 ~/.config/karakeep/config.json (权限 0o600)
```

### 4.3 Token 注入与传递

**浏览器扩展**（`apps/browser-extension/src/utils/trpc.ts`）：

```ts
// 创建 tRPC 客户端时注入
httpLink({
  url: `${address}/api/trpc`,
  headers: {
    Authorization: `Bearer ${apiKey}`,
    ...customHeaders,
  },
})
```

**CLI**（`apps/cli/src/lib/trpc.ts`）：

```
从 ~/.config/karakeep/config.json 读取 apiKey
→ 创建 tRPC client 时注入 Authorization header
```

**单文件上传（singlefile）**：

```ts
// apps/browser-extension/src/utils/singlefile.ts
headers = { Authorization: `Bearer ${settings.apiKey}` }
fetch(`${apiUrl}/api/v1/bookmarks`, { method: "POST", headers, body: formData })
```

### 4.4 Web Server 侧的认证注入

`apps/web/server/api/client.ts` 的 `createContextFromRequest` 是核心入口：

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
      // API key 验证失败 → 回退到 cookie session 认证
    }
  }
  return createContext(db, ip); // 走 session 路径
}
```

**关键设计：** API Key 验证失败时静默回退到 session 认证，不是返回 401。这意味着同一个端点可以同时支持两种认证方式。

### 4.5 Scope 检查执行时机

Scope 检查在**每次请求**的两个阶段进行：

**阶段一：tRPC Procedure 级（隐式自动）**

```
请求到达 router → createScopedAuthedProcedure("bookmarks")
  → opts.type === "query" → access = "read"
  → scope = "bookmarks:read"
  → apiKeyScopesGrantScope(auth.scopes, "bookmarks:read")
  → 不通过 → TRPCError(FORBIDDEN, "API key is missing required scope: bookmarks:read")
```

**阶段二：Hono Middleware 级（显式手动）**

```
请求到达 REST 路由 → apiKeyScopeMiddleware("bookmarks", "readwrite")
  → auth.type === "apiKey"
  → scope = "bookmarks:readwrite"
  → apiKeyScopesGrantScope(auth.scopes, "bookmarks:readwrite")
  → 不通过 → HTTPException(403, "API key is missing required scope: bookmarks:readwrite")
```

### 4.6 完整调用时序图

```
[外部脚本]
    │
    │ 1. GET /api/trpc/bookmarks.list
    │    Authorization: Bearer ak2_abc123_def456
    │
    ▼
[Next.js Route Handler]
    │  route.ts → createContextFromRequest(rawRequest)
    │
    ▼
[Token 解析]
    │  提取 Bearer token → authenticateApiKey(key, db)
    │    ├── parseApiKey: 校验格式 (ak2_xxx_yyy)
    │    ├── DB 查询 keyId
    │    ├── SHA256 比对 secret
    │    ├── 更新 lastUsedAt (10min 节流)
    │    └── 返回 { user, apiKey: { keyId, scopes } }
    │
    ▼
[上下文注入]
    │  ctx = { user, auth: { type: "apiKey", keyId, scopes }, db }
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

### 5.3 权限最小化

- 默认签发 `fullaccess`（向后兼容），但支持显式指定细粒度 scope
- `sessionProcedure` 防止 API Key 管理 API Key（防止提权递归）
- 管理员 scope 与普通 scope 命名空间隔离（`admin:` 前缀）

### 5.4 撤销即时性

- **无缓存**：每次请求实时查库验证，撤销即时生效
- **硬删除**：revoke 是 DELETE 操作，记录彻底消失
- **级联删除**：用户删除自动清理所有 key

### 5.5 限速保护

| 端点 | 窗口 | 最大请求 |
|------|------|---------|
| `exchange` | 15 分钟 | 10 |
| `validate` | 1 分钟 | 30 |
| 全局 public | 1 分钟 | 1000 |
| 全局 authed | 1 分钟 | 3000 |

---

## 六、代码索引

| 功能 | 文件路径 | 关键符号 |
|------|---------|---------|
| Key 生成/验证 | `packages/trpc/auth.ts` | `generateApiKey`, `authenticateApiKey`, `regenerateApiKey`, `parseApiKey` |
| Key 管理路由 | `packages/trpc/routers/apiKeys.ts` | `apiKeysAppRouter` (create/revoke/regenerate/list/exchange/validate) |
| Scope 类型定义 | `packages/shared/types/apiKeys.ts` | `API_KEY_SCOPE_RESOURCES`, `apiKeyScopesGrantScope`, `getApiKeyScope` |
| tRPC Scope 中间件 | `packages/trpc/index.ts` | `createScopedAuthedProcedure`, `sessionProcedure`, `rejectApiKeyAuth` |
| Hono Scope 中间件 | `packages/api/middlewares/apiKeyScopes.ts` | `apiKeyScopeMiddleware` |
| Web 上下文注入 | `apps/web/server/api/client.ts` | `createContextFromRequest` |
| 数据库 Schema | `packages/db/schema.ts` | `apiKeys` table |
| 浏览器扩展 | `apps/browser-extension/src/utils/trpc.ts` | `initializeClients`, Bearer header 注入 |
| CLI 配置 | `apps/cli/src/commands/auth.ts`, `apps/cli/src/lib/config.ts` | `auth init`, `~/.config/karakeep/config.json` |
| REST 认证 | `packages/api/middlewares/auth.ts` | `authMiddleware`, `unauthedMiddleware` |
| 测试用例 | `packages/trpc/routers/apiKeys.test.ts` | 完整生命周期/scope 强制/兼容性测试 |
