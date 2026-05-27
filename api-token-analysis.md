# Karakeep API Token 系统分析

## 概述

Karakeep 使用 API Key（非 JWT/OAuth）作为外部脚本和客户端的身份认证凭证。系统采用 `ak2_{keyId}_{secret}` 格式，服务端仅存储 keyId 和 secret 的 SHA256 哈希值，明文 key 仅在创建/重新生成时返回给用户一次。整体链路涉及签发、权限收敛、调用注入、撤销回收四大环节。

**核心架构（修正后）**：tRPC 和 REST 两条入口**大部分情况**共享相同的 scope 检查逻辑。但有 **3 个特殊端点**（`POST /assets`、`GET /assets/:assetId`、`POST /bookmarks/singlefile` 中的 `uploadAsset`）不经过 tRPC caller，其 scope 保护**完全依赖 route 层的 `apiKeyScopeMiddleware`**。

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

## 三、REST 入口鉴权流程：Route 层与 Procedure 层职责划分

### 3.1 整体架构

REST 入口的鉴权是一个**多层级、多中间件协同**的流程。**大多数** REST 路由通过 tRPC caller 间接触发 scope 检查，但**少数特殊端点**直接调用底层函数，其 scope 保护完全依赖 route 层中间件。

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
[Step 3] Hono Route 层中间件
    │  packages/api/middlewares/auth.ts
    │  → authMiddleware: 检查 ctx.user != null
    │  → ✅ 通过：c.set("api", createCaller(c.get("ctx")))
    │       【关键】创建 tRPC caller，绑定当前上下文
    ▼
[Step 4] Route 层 Scope 检查（可选，仅特殊端点）
    │  packages/api/middlewares/apiKeyScopes.ts
    │  → apiKeyScopeMiddleware(resource, access)
    │  → 仅在 assets 和 singlefile 端点添加
    ▼
[Step 5] Route 层输入验证
    │  packages/api/routes/*.ts
    │  → zValidator: 验证 query/body/params
    ▼
[Step 6] REST 路由处理（分两种路径）
    │
    ├─[路径 A] 标准业务路由 → c.var.api.{resource}.{operation}(params)
    │   通过 tRPC caller 调用 procedure → 触发 procedure 层 scope 检查
    │
    └─[路径 B] 特殊端点 → 直接调用底层函数 (uploadAsset / Asset.fromId)
        不经过 tRPC caller → scope 保护仅依赖 Step 4 的 route 层检查
    ▼
[Step 7a] tRPC Procedure 层（路径 A）
    │  → createScopedAuthedProcedure(resource) → Scope 检查
    │  → ensure*Ownership → 资源所有权检查
    │  → 业务逻辑
    │
[Step 7b] 资源访问校验（路径 B）
    │  → Asset.fromId().canUserView() / ensureCanView() → 资源访问检查
    │  → 业务逻辑
    ▼
[Step 8] 响应格式化
    │  → REST 路由可能进行响应格式转换
    │  → 返回 JSON 响应
```

### 3.2 Route 层与 Procedure 层职责划分

#### 3.2.1 Route 层职责

Route 层（Hono 路由 + 中间件）负责：

| 职责 | 实现 | 说明 |
|------|------|------|
| **用户认证检查** | `authMiddleware` | 检查 `ctx.user != null`，确保请求已认证 |
| **tRPC caller 创建** | `c.set("api", createCaller(c.get("ctx")))` | 将当前上下文绑定到 caller，使 procedure 能访问 auth 信息 |
| **Route 层 Scope 检查** | `apiKeyScopeMiddleware` | **仅特殊端点需要**：不经过 tRPC caller 的操作，scope 保护完全依赖此中间件 |
| **输入验证** | `zValidator` | 验证 HTTP 请求的 query/body/params |
| **响应格式化** | 路由处理函数 | 进行响应格式转换（如分页适配、base64 编码） |
| **速率限制** | `createRateLimitMiddleware` | 部分端点添加速率限制 |

#### 3.2.2 Procedure 层职责

Procedure 层（tRPC router + 中间件）负责：

| 职责 | 实现 | 说明 |
|------|------|------|
| **Scope 检查** | `createScopedAuthedProcedure(resource)` | **标准路由的唯一 scope 保护**：检查 API Key 的 scope 是否满足操作要求 |
| **资源所有权检查** | `ensure*Ownership` 中间件 | 确保用户拥有被操作的资源 |
| **业务逻辑执行** | procedure handler | 执行业务操作（数据库查询/修改） |
| **事件日志** | `createEventLogMiddleware` | 记录操作事件 |
| **管理员角色检查** | `createAdminScopedProcedure(resource)` | 检查用户角色是否为 admin |

#### 3.2.3 两条路径的关键差异

| 维度 | 路径 A（标准路由） | 路径 B（特殊端点） |
|------|-------------------|------------------|
| **Scope 检查位置** | Procedure 层（`createScopedAuthedProcedure`） | Route 层（`apiKeyScopeMiddleware`） |
| **Scope 检查是否可绕过** | 不可绕过（caller 携带完整上下文） | **理论上可绕过**（如果 route 层忘记添加中间件） |
| **调用方式** | `c.var.api.resource.operation()` | `uploadAsset()` / `Asset.fromId()` |
| **tRPC procedure 层** | 经过 | 不经过 |
| **资源所有权检查** | `ensure*Ownership` 中间件 | `Asset.fromId().canUserView()` 内联检查 |
| **涉及端点** | 除 assets 相关以外的所有标准路由 | `POST /assets`, `GET /assets/:assetId`, `POST /bookmarks/singlefile`（uploadAsset 部分） |

---

## 四、Assets 相关端点执行路径深度分析

### 4.1 POST /assets — 完整执行路径

**文件**: `packages/api/routes/assets.ts:15-42`

```
HTTP POST /assets
Authorization: Bearer ak2_xxx_yyy
Content-Type: multipart/form-data
```

**执行步骤（按顺序）：**

| 步骤 | 中间件/函数 | 代码位置 | 作用 |
|------|------------|---------|------|
| 1 | `authMiddleware` | `middlewares/auth.ts:24` | 检查 `ctx.user != null`，创建 tRPC caller |
| 2 | `apiKeyScopeMiddleware("assets", "readwrite")` | `middlewares/apiKeyScopes.ts:14` | **唯一的 scope 保护**：检查 auth.type === "apiKey" 时 scopes 是否包含 `assets:readwrite` |
| 3 | `createRateLimitMiddleware({...})` | `middlewares/rateLimit.ts` | 速率限制：1 分钟 30 次 |
| 4 | `zValidator("form", ...)` | `@hono/zod-validator` | 验证 form 数据包含 `file` 或 `image` 字段 |
| 5 | `uploadAsset(user, db, body)` | `utils/upload.ts:42` | **直接调用**，不经过 tRPC caller |

**Step 5 详解 — `uploadAsset` 内部：**

```ts
// packages/api/utils/upload.ts:42-143
export async function uploadAsset(user, db, formData) {
  // 1. 文件类型检测（安全检查）
  const detectedType = await fileTypeFromBlob(data);
  
  // 2. 类型白名单检查
  if (!SUPPORTED_UPLOAD_ASSET_TYPES.has(contentType)) { ... }
  
  // 3. 大小限制检查
  if (data.size > MAX_UPLOAD_SIZE_BYTES) { ... }
  
  // 4. 存储配额检查
  await QuotaService.checkStorageQuota(db, user.id, data.size);
  
  // 5. 写入临时文件 → 保存到对象存储 → 数据库插入
  const [assetDb] = await db.insert(assets).values({
    id: newAssetId(),
    userId: user.id,    // 直接使用 ctx.user.id
    contentType, size, fileName,
  }).returning();
  
  await saveAssetFromFile({ ... });
  return { assetId, contentType, size, fileName };
}
```

**关键发现：**
- ❌ **不经过 tRPC caller**：`uploadAsset` 直接操作数据库，不调用 `c.var.api.*`
- ❌ **不经过 tRPC procedure 层**：没有 `createScopedAuthedProcedure` 的 scope 检查
- ✅ **唯一 scope 保护**：Step 2 的 `apiKeyScopeMiddleware("assets", "readwrite")`
- ✅ **无资源所有权检查**：创建操作不需要所有权检查（用户创建自己的资产）
- ⚠️ **如果移除 Step 2 的中间件**：任何已认证用户（包括 scope 受限的 API Key）都能上传资产

### 4.2 GET /assets/:assetId — 完整执行路径

**文件**: `packages/api/routes/assets.ts:43-50`

```
HTTP GET /assets/{assetId}
Authorization: Bearer ak2_xxx_yyy
```

**执行步骤（按顺序）：**

| 步骤 | 中间件/函数 | 代码位置 | 作用 |
|------|------------|---------|------|
| 1 | `authMiddleware` | `middlewares/auth.ts:24` | 检查 `ctx.user != null`，创建 tRPC caller |
| 2 | `apiKeyScopeMiddleware("assets", "read")` | `middlewares/apiKeyScopes.ts:14` | **唯一的 scope 保护**：检查 scopes 是否包含 `assets:read` |
| 3 | `Asset.fromId(ctx, assetId)` | `trpc/models/assets.ts:28` | **直接调用**，不经过 tRPC caller |
| 4 | `asset.ensureCanView()` | `trpc/models/assets.ts:253` | 资源访问校验 |
| 5 | `serveAsset(c, assetId, asset.asset.userId)` | `utils/assets.ts` | 返回文件内容 |

**Step 3-4 详解 — 资源访问校验：**

```ts
// packages/trpc/models/assets.ts:28-50
static async fromId(ctx, id) {
  const assetdb = await ctx.db.query.assets.findFirst({ where: eq(assets.id, id) });
  if (!assetdb) { throw TRPCError(NOT_FOUND); }
  const asset = new Asset(ctx, assetdb);
  
  // 内联资源访问检查
  if (!(await asset.canUserView())) {
    throw TRPCError(NOT_FOUND);
  }
  return asset;
}

// canUserView 逻辑 (line 225-251):
async canUserView() {
  // 规则 1: 资产所有者可查看
  if (this.asset.userId === this.ctx.user.id) { return true; }
  // 规则 2: 头像始终公开
  if (this.asset.assetType === "avatar") { return true; }
  // 规则 3: 如果资产属于书签，检查书签访问权限
  if (this.asset.bookmarkId) {
    await BareBookmark.bareFromId(this.ctx, this.asset.bookmarkId);
    return true;
  }
  return false;
}
```

**关键发现：**
- ❌ **不经过 tRPC caller**：`Asset.fromId` 直接查询数据库
- ❌ **不经过 tRPC procedure 层**：没有 `createScopedAuthedProcedure` 的 scope 检查
- ✅ **唯一 scope 保护**：Step 2 的 `apiKeyScopeMiddleware("assets", "read")`
- ✅ **有资源所有权/访问检查**：`canUserView()` 三层规则
- ⚠️ **安全依赖**：scope 检查 + 资源访问检查形成双重防护

### 4.3 POST /bookmarks/singlefile — 混合执行路径

**文件**: `packages/api/routes/bookmarks.ts:111-203`

```
HTTP POST /bookmarks/singlefile?ifexists=skip
Authorization: Bearer ak2_xxx_yyy
Content-Type: multipart/form-data
```

**执行步骤（按顺序）：**

| 步骤 | 中间件/函数 | 代码位置 | 作用 |
|------|------------|---------|------|
| 1 | `authMiddleware` | `middlewares/auth.ts:24` | 检查用户，创建 tRPC caller |
| 2 | `apiKeyScopeMiddleware("assets", "readwrite")` | `middlewares/apiKeyScopes.ts:14` | 资产上传的 scope 保护 |
| 3 | `apiKeyScopeMiddleware("bookmarks", "readwrite")` | `middlewares/apiKeyScopes.ts:14` | 书签创建的 scope 保护 |
| 4 | `zValidator("query", ...)` | `@hono/zod-validator` | 验证 `ifexists` 参数 |
| 5 | `zValidator("form", ...)` | `@hono/zod-validator` | 验证 `url` + `file` |
| 6 | `uploadAsset(user, db, form)` | `utils/upload.ts:42` | **路径 B：直接调用**，scope 依赖 Step 2 |
| 7 | `c.var.api.bookmarks.createBookmark(...)` | tRPC caller → procedure | **路径 A：触发 procedure 层 scope 检查** |
| 8 | `c.var.api.assets.replaceAsset(...)` | tRPC caller → procedure | **路径 A：触发 procedure 层 scope 检查** |
| 9 | `c.var.api.assets.attachAsset(...)` | tRPC caller → procedure | **路径 A：触发 procedure 层 scope 检查** |
| 10 | `c.var.api.bookmarks.recrawlBookmark(...)` | tRPC caller → procedure | **路径 A：触发 procedure 层 scope 检查** |

**关键发现：**
- ⚠️ **混合路径**：Step 6 (`uploadAsset`) 走路径 B（直接调用），Steps 7-10 走路径 A（tRPC caller）
- ✅ **双重 scope 检查**：
  - Step 2 `apiKeyScopeMiddleware("assets", "readwrite")` 保护 `uploadAsset`
  - Step 3 `apiKeyScopeMiddleware("bookmarks", "readwrite")` 保护后续书签操作
  - Steps 7-10 的 tRPC caller 又会触发 procedure 层的 `bookmarksProcedure` 和 `assetsProcedure` scope 检查
- 实际上 Step 3 和后续 procedure 层的 scope 检查是**冗余的**（双重检查），但 Step 2 对 `uploadAsset` 是**必需的**

### 4.4 三个端点的 Scope 保护机制对比

| 端点 | Scope 检查层级 | 是否经过 tRPC caller | 是否经过 Procedure 层 | 资源访问检查 | 安全等级 |
|------|--------------|---------------------|----------------------|------------|---------|
| `POST /assets` | **仅 Route 层** (`apiKeyScopeMiddleware`) | ❌ | ❌ | 不需要（创建操作） | ⚠️ 单点防护 |
| `GET /assets/:assetId` | **仅 Route 层** (`apiKeyScopeMiddleware`) | ❌ | ❌ | ✅ `canUserView()` 三层规则 | ✅ 双重防护 |
| `POST /bookmarks/singlefile` - uploadAsset | **仅 Route 层** (`apiKeyScopeMiddleware`) | ❌ | ❌ | 不需要（创建操作） | ⚠️ 单点防护 |
| `POST /bookmarks/singlefile` - createBookmark | Route 层 + Procedure 层 | ✅ | ✅ | ✅ `ensureBookmarkOwnership` | ✅ 双重防护 |
| `POST /bookmarks/singlefile` - replace/attachAsset | Route 层 + Procedure 层 | ✅ | ✅ | ✅ `ensureBookmarkOwnership` + `ensureOwnership` | ✅ 多重防护 |

---

## 五、REST 路由分类与校验职责

### 5.1 标准业务路由（authMiddleware + tRPC caller → Procedure 层 Scope 检查）

这些路由的 scope 检查**完全由 tRPC procedure 层执行**，Route 层无需额外 scope 中间件：

| 路由文件 | Procedure 定义 | Scope 检查位置 | 额外资源所有权检查 |
|---------|---------------|---------------|------------------|
| `bookmarks.ts` (除 singlefile) | `bookmarksProcedure = createScopedAuthedProcedure("bookmarks")` | Procedure 层 | `ensureBookmarkOwnership`（update/delete） |
| `tags.ts` | `tagsProcedure = createScopedAuthedProcedure("tags")` | Procedure 层 | `ensureTagOwnership`（get/delete/update） |
| `lists.ts` | `listsProcedure = createScopedAuthedProcedure("lists")` | Procedure 层 | `ensureListAtLeastViewer/Owner`（edit/delete） |
| `feeds.ts` | `feedsProcedure = createScopedAuthedProcedure("feeds")` | Procedure 层 | `ensureFeedOwnership`（get/update/delete/fetch） |
| `highlights.ts` | `highlightsProcedure = createScopedAuthedProcedure("highlights")` | Procedure 层 | `ensureHighlightOwnership`（get/update/delete） |
| `backups.ts` | `backupsProcedure = createScopedAuthedProcedure("backups")` | Procedure 层 | 无 |
| `users.ts` | `usersProcedure = createScopedAuthedProcedure("users")` | Procedure 层 | 无 |
| `webhooks.ts` | `webhooksProcedure = createScopedAuthedProcedure("webhooks")` | Procedure 层 | `ensureWebhookOwnership`（update/delete） |

**关键代码模式**（以 `tags.ts` 为例）：

```ts
// Route 层：仅做输入验证，通过 c.var.api.tags.* 调用
.post("/", zValidator("json", zCreateTagRequestSchema), async (c) => {
  const body = c.req.valid("json");
  const tags = await c.var.api.tags.create(body);  // ✅ 通过 caller → 触发 procedure scope 检查
  return c.json(tags, 201);
})

// Procedure 层：scope 检查由 createScopedAuthedProcedure 自动执行
const tagsProcedure = createScopedAuthedProcedure("tags");
```

### 5.2 特殊端点（Route 层 Scope 检查 → 直接调用底层函数）

这些端点**不经过 tRPC caller**，scope 保护完全依赖 Route 层：

| 端点 | Route 层 Scope 检查 | 直接调用的函数 | 资源访问检查 |
|------|---------------------|--------------|------------|
| `POST /assets` | `apiKeyScopeMiddleware("assets", "readwrite")` | `uploadAsset()` | 不需要 |
| `GET /assets/:assetId` | `apiKeyScopeMiddleware("assets", "read")` | `Asset.fromId()` + `asset.ensureCanView()` | ✅ `canUserView()` |
| `POST /bookmarks/singlefile` (uploadAsset 部分) | `apiKeyScopeMiddleware("assets", "readwrite")` | `uploadAsset()` | 不需要 |

### 5.3 管理员路由（adminAuthMiddleware + tRPC caller）

这些路由需要**管理员 scope + 角色检查**：

| 路由文件 | Scope 检查位置 | 角色检查 |
|---------|---------------|---------|
| `admin.ts` | `createAdminScopedProcedure("bookmarks"/"jobs"/"system"/"users")` | `adminAuthMiddleware` 检查 `user.role === "admin"` |

### 5.4 特殊认证路由

这些路由使用独立的认证机制，**不经过标准的 API Key / Session 认证链路**：

| 路由文件 | 认证方式 | 说明 |
|---------|---------|------|
| `webhooks/stripe` | Stripe 签名验证 | 独立的 webhook 认证 |
| `rss/lists/:listId` | `unauthedMiddleware` + 公共 token | 公共 RSS 订阅，使用 list token |
| `public/assets/:assetId` | `unauthedMiddleware` + 签名 token | 公共资产访问，使用签名 token |
| `metrics` | `bearerAuth` + 独立 token | Prometheus 指标，使用独立配置 token |
| `health` | 无认证 | 健康检查 |
| `version` | 无认证 | 版本信息 |

### 5.5 仅 tRPC 访问的资源

这些资源**没有 REST 路由**，仅通过 tRPC 路径访问：

| 资源 | Router 文件 | Procedure 类型 |
|------|-------------|---------------|
| `apiKeys` | `apiKeys.ts` | `sessionProcedure`（仅 Web 界面） |
| `rules` | `rules.ts` | `createScopedAuthedProcedure("rules")` |
| `prompts` | `prompts.ts` | `createScopedAuthedProcedure("prompts")` |
| `importSessions` | `importSessions.ts` | `createScopedAuthedProcedure("importSessions")` |
| `subscriptions` | `subscriptions.ts` | `createScopedAuthedProcedure("subscriptions")` |

---

## 六、tRPC 与 REST 两条入口的异同点

### 6.1 对比表

| 维度 | tRPC 路径 (`/api/trpc/*`) | REST 路径 (`/api/v1/*`) |
|------|---------------------------|-------------------------|
| **入口 URL** | `/api/trpc/{router}.{procedure}` | `/api/v1/{resource}` |
| **上下文创建** | `createContextFromRequest` 统一处理 | `createContextFromRequest` 统一处理 |
| **认证方式** | Bearer Token / Session Cookie | Bearer Token / Session Cookie |
| **用户认证检查** | tRPC 内部 `authedProcedure` | Hono `authMiddleware` |
| **tRPC caller 创建** | tRPC 适配器内部创建 | `authMiddleware` 显式创建 |
| **标准端点 Scope 检查** | Procedure 层直接执行 | 通过 caller 间接触发 Procedure 层检查 |
| **特殊端点 Scope 检查** | N/A（所有 tRPC 端点都经过 Procedure 层） | **Route 层 `apiKeyScopeMiddleware`（唯一保护）** |
| **资源所有权检查** | `ensure*Ownership` 中间件 | 路径 A：`ensure*Ownership` 中间件；路径 B：`Asset.fromId().canUserView()` |
| **路由级 Scope 检查** | 无（procedure 层已覆盖） | 特殊端点需要，标准端点冗余 |
| **输入验证** | tRPC 内置 zod 验证 | Hono `zValidator` |
| **响应格式** | tRPC 标准格式 | 可能有额外格式化（分页适配等） |
| **Session 用户** | 不受 scope 限制 | 不受 scope 限制 |
| **API Key 用户** | 所有端点强制 scope 检查 | 路径 A：强制检查；路径 B：依赖 Route 层检查 |
| **Bearer 失败回退** | 静默回退到 session | 静默回退到 session |
| **支持的客户端** | Web 端、CLI、扩展、移动端 | 扩展、移动端、第三方脚本 |

### 6.2 核心相同点

1. **共享认证链路**：两条路径都经过 `createContextFromRequest` 处理 Bearer Token 和 Session
2. **共享 Scope 检查逻辑**：路径 A 最终都执行 `createScopedAuthedProcedure` 的相同逻辑
3. **共享业务逻辑**：路径 A 调用相同的 tRPC procedure 执行业务操作
4. **相同回退行为**：Bearer Token 失败时都静默回退到 Session 认证
5. **相同撤销机制**：API Key 撤销对两条路径同时生效

### 6.3 核心不同点

1. **调用方式**：
   - tRPC：客户端直接调用 procedure，类型安全
   - REST：客户端通过 HTTP 方法调用，路径 A 通过 caller 转发到 procedure

2. **特殊端点的 Scope 保护**：
   - tRPC：所有端点都经过 Procedure 层 scope 检查
   - REST：3 个特殊端点的部分操作**仅依赖 Route 层** scope 中间件

3. **错误返回格式**：
   - tRPC：标准 tRPC 错误格式（JSON-RPC）
   - REST：HTTP 状态码 + JSON 响应

4. **响应格式化**：
   - tRPC：返回原始 procedure 输出
   - REST：可能有额外格式化（如分页适配、base64 编码 cursor）

---

## 七、Bearer 失败时的会话回退机制

### 7.1 回退条件

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

### 7.2 回退路径

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

### 7.3 权限边界与安全影响（重新分析）

#### 7.3.1 回退不是 Scope 绕过，而是权限模型切换

**关键理解**：回退到 session 后 `auth.type = "session"`，`createScopedAuthedProcedure` 和 `apiKeyScopeMiddleware` 都跳过检查。这不是"绕过"，而是**设计如此的权限模型切换**：

- **Session 用户**：代表"已登录的用户本人"，拥有完整的用户权限，不受 scope 限制
- **API Key 用户**：代表"被授权的第三方客户端"，权限受 scope 限制
- 两种认证方式的权限模型本来就不同

#### 7.3.2 回退对不同端点的安全影响

| 端点类型 | 回退前（auth.type = "apiKey"） | 回退后（auth.type = "session"） | 安全影响 |
|---------|------------------------------|-------------------------------|---------|
| **标准路由（路径 A）** | Scope 受限（Procedure 层检查） | 完整用户权限（跳过 scope） | 权限模型切换（设计如此） |
| **POST /assets（路径 B）** | Scope 受限（Route 层 `apiKeyScopeMiddleware`） | 完整用户权限（跳过 scope） | 权限模型切换（设计如此） |
| **GET /assets/:assetId（路径 B）** | Scope 受限 + 资源访问检查 | 完整用户权限 + 资源访问检查 | 权限模型切换（设计如此） |
| **POST /bookmarks/singlefile** | 双重 scope 检查 + 资源访问检查 | 完整用户权限 + 资源访问检查 | 权限模型切换（设计如此） |

#### 7.3.3 混淆代理攻击风险

```
攻击者视角：
1. 受害者已登录 Web 应用（有有效 session cookie）
2. 攻击者诱导受害者访问恶意页面
3. 恶意页面发送请求：POST /api/v1/assets
   Header: Authorization: Bearer <无效的 API Key>
   Body: multipart/form-data (恶意文件)
   
4. 服务器处理：
   → Bearer token 无效 → 回退到 session
   → session 有效 → auth.type = "session"
   → apiKeyScopeMiddleware 跳过（auth.type !== "apiKey"）
   → 以受害者身份上传资产

风险：攻击者可以利用受害者的 session 权限执行操作
缓解：这是所有基于 cookie 的 Web 应用的固有风险（CSRF）
```

**严重程度评估**：中。这是 Session 认证的固有风险，不是 scope 检查机制的漏洞。

#### 7.3.4 权限变化对比表

| 场景 | auth.type | 权限模型 | Scope 检查 | 资源所有权检查 |
|------|-----------|---------|-----------|--------------|
| Bearer token 有效 | `apiKey` | 受 scope 限制 | ✅ 执行 | ✅ 执行 |
| Bearer token 无效 + 有效 session | `session` | 完整用户权限 | ❌ 跳过（设计如此） | ✅ 执行 |
| Bearer token 无效 + 无 session | `null` | 未认证 | N/A | N/A |
| 无 Bearer token + 有效 session | `session` | 完整用户权限 | ❌ 跳过（设计如此） | ✅ 执行 |
| 无 Bearer token + 无 session | `null` | 未认证 | N/A | N/A |

#### 7.3.5 对纯 API 客户端的影响

对于没有 cookie 的纯 API 客户端（如 CLI、服务器脚本）：
- Bearer 失败后 `getServerAuthSession()` 返回 null
- 最终 `authMiddleware` 返回 401
- 错误信息不明确，用户无法知道是 key 无效还是其他问题

---

## 八、撤销与回收策略

### 8.1 撤销操作

**硬删除（Hard Delete）**

`revoke` 端点（`apiKeys.ts:89`）执行数据库硬删除：

```sql
DELETE FROM apiKey WHERE id = ? AND userId = ?
```

特点：
- 删除后记录完全消失，`validate` 立即失败
- 操作不可逆
- 需要 `sessionProcedure`（API Key 自身无法撤销自己）

### 8.2 重新生成（Regenerate）

`regenerate` 端点（`apiKeys.ts:61`）是"撤销+重发"：

1. 根据 `id` + `userId` 查找已有记录
2. 生成新的 `keyId` + `keyHash`
3. 更新该记录
4. **旧 keyId 被替换，旧 key 立即失效**

数据库操作是 `UPDATE` 而非 `DELETE + INSERT`，保持 `id` 不变，保留 name/scopes/createdAt。

### 8.3 级联回收

用户删除时的级联：`apiKeys.userId` 设置了 `references(() => users.id, { onDelete: "cascade" })`，用户删除时所有关联 API Key 自动清理。

### 8.4 运行时回收 — 请求时实时验证

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

### 8.5 使用追踪

`lastUsedAt` 字段以 10 分钟为节流周期更新（`auth.ts:140`）：

```ts
const tenMinutesAgo = new Date(Date.now() - 10 * 60 * 1000);
if (!apiKey.lastUsedAt || apiKey.lastUsedAt < tenMinutesAgo) {
  // fire-and-forget, 不阻塞认证响应
  database.update(apiKeys).set({ lastUsedAt: new Date() })...
}
```

**策略：** 更新为"即发即忘"（async 不 await），避免数据库写入阻塞认证流程。

### 8.6 撤销策略总结

| 操作 | 效果 | 可恢复 |
|------|------|--------|
| `revoke` | 硬删除，立即失效 | 否 |
| `regenerate` | keyId 替换，旧 key 失效 | 否 |
| 用户删除 | 级联删除所有 API Key | 否 |

---

## 九、外部脚本调用系统完整链路

### 9.1 tRPC 路径完整时序

```
[外部脚本]
    │
    │ 1. POST /api/trpc/tags.create
    │    Authorization: Bearer ak2_abc123_def456
    │    Body: { "input": { "name": "my-tag" } }
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
    │  router.tags.create
    │    ├── tagsProcedure = createScopedAuthedProcedure("tags")
    │    │   ├── 检查 auth.type === "apiKey" ✓
    │    │   ├── opts.type === "mutation" → access = "readwrite"
    │    │   ├── scope = "tags:readwrite"
    │    │   └── apiKeyScopesGrantScope 检查 ✓
    │    ├── createEventLogMiddleware("tag.create")
    │    ├── 输入验证 (zod) → zCreateTagRequestSchema
    │    └── 执行业务逻辑 → Tag.create(ctx, input)
    │
    ▼
[响应]
    │ 返回 tRPC 格式响应
```

### 9.2 REST 路径完整时序（标准业务路由，路径 A）

```
[外部脚本]
    │
    │ 1. POST /api/v1/tags
    │    Authorization: Bearer ak2_abc123_def456
    │    Body: { "name": "my-tag" }
    │
    ▼
[Next.js Route Handler]
    │  → createContextFromRequest
    │  → ✅ API Key 验证成功 → auth.type = "apiKey"
    │
    ▼
[Hono authMiddleware]
    │    ├── 检查 ctx.user != null ✓
    │    └── 【关键】c.set("api", createCaller(c.get("ctx")))
    │
    ▼
[Route 层输入验证]
    │  zValidator("json", zCreateTagRequestSchema) ✓
    │
    ▼
[REST 路由处理]
    │  POST /tags 路由处理函数
    │    └── 【关键】调用 c.var.api.tags.create(body) → 进入 tRPC
    │
    ▼
[tRPC Procedure 层]
    │  caller 触发 tags.create procedure
    │    ├── tagsProcedure = createScopedAuthedProcedure("tags")
    │    │   ├── 检查 auth.type === "apiKey" ✓
    │    │   ├── opts.type === "mutation" → access = "readwrite"
    │    │   ├── scope = "tags:readwrite"
    │    │   └── apiKeyScopesGrantScope 检查 ✓
    │    ├── createEventLogMiddleware("tag.create")
    │    ├── 输入验证 (zod)
    │    └── 执行业务逻辑 → Tag.create(ctx, input)
    │
    ▼
[响应格式化]
    │  return c.json(tags, 201)
```

### 9.3 REST 路径完整时序（POST /assets，路径 B）

```
[外部脚本]
    │
    │ 1. POST /api/v1/assets
    │    Authorization: Bearer ak2_abc123_def456
    │    Body: multipart/form-data (file)
    │
    ▼
[Next.js Route Handler]
    │  → createContextFromRequest
    │  → ✅ API Key 验证成功 → auth.type = "apiKey"
    │
    ▼
[Hono authMiddleware]
    │    ├── 检查 ctx.user != null ✓
    │    └── c.set("api", createCaller(c.get("ctx")))  [caller 被创建但未使用]
    │
    ▼
[Route 层 Scope 检查]
    │  apiKeyScopeMiddleware("assets", "readwrite")
    │    ├── auth.type === "apiKey" ✓
    │    ├── scope = "assets:readwrite"
    │    └── apiKeyScopesGrantScope 检查 ✓
    │
    ▼
[速率限制]
    │  createRateLimitMiddleware → 1 分钟 30 次 ✓
    │
    ▼
[输入验证]
    │  zValidator("form", ...) → 验证包含 file 或 image ✓
    │
    ▼
[直接调用底层函数 — 不经过 tRPC]
    │  uploadAsset(c.var.ctx.user, c.var.ctx.db, body)
    │    ├── 文件类型检测（fileTypeFromBlob）
    │    ├── 类型白名单检查
    │    ├── 大小限制检查
    │    ├── 存储配额检查
    │    ├── 写入临时文件
    │    ├── db.insert(assets).values({ userId: user.id, ... })
    │    └── saveAssetFromFile → 对象存储
    │
    ▼
[响应]
    │  return c.json({ assetId, contentType, size, fileName }, 201)
```

### 9.4 Token 获取方式

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

## 十、安全设计要点

### 10.1 存储安全

- **仅存哈希**：数据库只存 `keyHash`（SHA256），不存明文 secret
- **keyId 分离**：keyId 是可公开标识（用于查找），与 secret 独立生成
- **V1→V2 迁移**：V1 使用 bcrypt（计算慢，更抗暴力），V2 使用 SHA256（性能好）

### 10.2 传输安全

- **Authorization: Bearer** 标准格式，支持 HTTPS 下的 TLS 传输
- CORS 配置允许 `Authorization` 和 `Content-Type` 头
- Web Server 中 API Key 失败时静默回退到 session，不泄露 key 是否有效

### 10.3 权限模型（修正后）

- **路径 A（标准路由）**：tRPC 和 REST 路径共享相同的 `createScopedAuthedProcedure` 检查，无绕过可能
- **路径 B（特殊端点）**：scope 保护**完全依赖 Route 层** `apiKeyScopeMiddleware`，不经过 tRPC Procedure 层
  - `POST /assets`：单点防护（仅 Route 层 scope 检查）
  - `GET /assets/:assetId`：双重防护（Route 层 scope 检查 + 资源访问检查）
  - `POST /bookmarks/singlefile` (uploadAsset)：单点防护（仅 Route 层 scope 检查）
- **两层防护**：Scope 检查 + 资源所有权检查（ensureOwnership / canUserView）
- **默认签发 `fullaccess`**：向后兼容，但支持显式指定细粒度 scope
- **`sessionProcedure`**：防止 API Key 管理 API Key（防止提权递归）
- **管理员 scope 命名空间隔离**：`admin:` 前缀

### 10.4 apiKeyScopeMiddleware 的真实作用

**之前的错误结论**：认为 `apiKeyScopeMiddleware` 是冗余检查

**修正后的正确理解**：

| 端点 | `apiKeyScopeMiddleware` 的作用 |
|------|-------------------------------|
| `POST /assets` | **必需的唯一 scope 保护** — `uploadAsset` 不经过 tRPC caller |
| `GET /assets/:assetId` | **必需的唯一 scope 保护** — `Asset.fromId` 不经过 tRPC caller |
| `POST /bookmarks/singlefile` (assets 部分) | **必需的唯一 scope 保护** — `uploadAsset` 不经过 tRPC caller |
| `POST /bookmarks/singlefile` (bookmarks 部分) | **冗余检查** — `c.var.api.bookmarks.*` 会触发 Procedure 层检查 |

### 10.5 回退机制的安全权衡

| 设计选择 | 优点 | 风险 |
|---------|------|------|
| Bearer 失败静默回退 | 同一端点同时支持 API Key 和 Session，用户体验好 | 混淆代理攻击（利用用户 session），权限从受限变为完整 |
| Session 用户不受 scope 限制 | Web 界面使用简单 | 回退时权限模型切换（设计如此，但可能被误解为绕过） |
| 不返回 401 区分 key 无效/权限不足 | 防止 key 存在性探测 | 调试困难，API 客户端无法知道 key 是否正确 |

### 10.6 撤销即时性

- **无缓存**：每次请求实时查库验证，撤销即时生效
- **硬删除**：revoke 是 DELETE 操作，记录彻底消失
- **级联删除**：用户删除自动清理所有 key

### 10.7 限速保护

| 端点 | 窗口 | 最大请求 |
|------|------|---------|
| `exchange` | 15 分钟 | 10 |
| `validate` | 1 分钟 | 30 |
| 全局 public | 1 分钟 | 1000 |
| 全局 authed | 1 分钟 | 3000 |
| `assets.upload` | 1 分钟 | 30 |
| `bookmarks.createBookmark` | 1 分钟 | 30 |

---

## 十一、代码索引

| 功能 | 文件路径 | 关键符号 |
|------|---------|---------|
| Key 生成/验证 | `packages/trpc/auth.ts` | `generateApiKey`, `authenticateApiKey`, `regenerateApiKey`, `parseApiKey` |
| Key 管理路由 | `packages/trpc/routers/apiKeys.ts` | `apiKeysAppRouter` (create/revoke/regenerate/list/exchange/validate) |
| Scope 类型定义 | `packages/shared/types/apiKeys.ts` | `API_KEY_SCOPE_RESOURCES`, `apiKeyScopesGrantScope`, `getApiKeyScope` |
| tRPC Scope 中间件 | `packages/trpc/index.ts` | `createScopedAuthedProcedure`, `createAdminScopedProcedure`, `sessionProcedure`, `rejectApiKeyAuth`, `createCallerFactory` |
| Hono 认证中间件 | `packages/api/middlewares/auth.ts` | `authMiddleware`（创建 tRPC caller）, `adminAuthMiddleware`, `unauthedMiddleware` |
| Hono Route 层 Scope 中间件 | `packages/api/middlewares/apiKeyScopes.ts` | `apiKeyScopeMiddleware`（**路径 B 端点的唯一 scope 保护**） |
| Assets REST 路由 | `packages/api/routes/assets.ts` | `POST /`, `GET /:assetId`（路径 B，直接调用底层函数） |
| Assets 上传工具 | `packages/api/utils/upload.ts` | `uploadAsset`（直接操作数据库，不经过 tRPC） |
| Assets 模型 | `packages/trpc/models/assets.ts` | `Asset.fromId`, `Asset.canUserView`, `Asset.ensureCanView`, `Asset.ensureOwnership` |
| Bookmarks REST 路由 | `packages/api/routes/bookmarks.ts` | `POST /singlefile`（混合路径：uploadAsset 直接调用 + 其余走 tRPC caller） |
| Web 上下文注入 | `apps/web/server/api/client.ts` | `createContextFromRequest`（Bearer 回退逻辑） |
| Web Session 认证 | `apps/web/server/auth.ts` | `getServerAuthSession`, NextAuth 配置 |
| 数据库 Schema | `packages/db/schema.ts` | `apiKeys` table, `assets` table |
| 浏览器扩展登录 | `apps/browser-extension/src/SignInPage.tsx` | exchange 调用 |
| 移动端登录 | `apps/mobile/app/signin.tsx` | exchange 调用 |
| 浏览器扩展 tRPC | `apps/browser-extension/src/utils/trpc.ts` | `initializeClients`, Bearer header 注入 |
| CLI 配置 | `apps/cli/src/commands/auth.ts`, `apps/cli/src/lib/config.ts` | `auth init`, `~/.config/karakeep/config.json` |
| REST 路由（标准业务，路径 A） | `packages/api/routes/tags.ts`, `lists.ts`, `feeds.ts`, `highlights.ts`, `backups.ts`, `users.ts`, `webhooks.ts` | `c.var.api.*` 调用 → Procedure 层 scope 检查 |
| REST 路由（管理员） | `packages/api/routes/admin.ts` | `adminAuthMiddleware` |
| REST 路由（特殊认证） | `packages/api/routes/webhooks.ts`, `rss.ts`, `public/`, `metrics.ts` | 独立认证机制 |
| tRPC Router 示例 | `packages/trpc/routers/bookmarks.ts`, `tags.ts`, `lists.ts`, `feeds.ts`, `highlights.ts` | `createScopedAuthedProcedure`, `ensure*Ownership` |
| tRPC Assets Router | `packages/trpc/routers/assets.ts` | `assetsProcedure`, `list`, `attachAsset`, `replaceAsset`, `detachAsset` |
| tRPC Admin Router | `packages/trpc/routers/admin.ts` | `createAdminScopedProcedure` |
| 测试用例 | `packages/trpc/routers/apiKeys.test.ts` | 完整生命周期/scope 强制/兼容性测试 |
