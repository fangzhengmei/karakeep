# Karakeep 限流与防滥用入口边界分析

本文档梳理 Karakeep 项目中与限流、防滥用相关的所有代码路径，包括 IP 识别、代理头信任、用户配额、缓存状态与不同入口的拦截逻辑。

---

## 1. 整体架构概览

Karakeep 的防滥用体系分为五层：

| 层级 | 作用 | 核心模块 |
|------|------|----------|
| IP 识别层 | 获取真实客户端 IP，用于限流 Key | `request-ip` 库 |
| 限流插件层 | 提供可插拔的限流存储后端（内存/Redis） | `PluginManager` + RateLimit 插件 |
| 全局限流中间件 | 在 tRPC/Hono 入口对所有请求做基础限流 | `publicProcedure` / `authedProcedure` |
| 接口级限流 | 对敏感接口做更严格的独立限流 | 各 router 中的 `createRateLimitMiddleware` |
| 业务配额层 | 对用户资源（书签数、存储空间、爬取能力）做配额限制 | `QuotaService` + 数据库字段 |
| 爬虫层限流 | 对外部网站域名的爬取频率做限制 | `checkDomainRateLimit` |
| 人机验证层 | Cloudflare Turnstile CAPTCHA | `verifyTurnstileToken` |

---

## 2. IP 识别与代理头信任

### 2.1 IP 获取位置

所有请求的 IP 提取集中在一个入口：

**文件**: [apps/web/server/api/client.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/web/server/api/client.ts#L10-L64)

```typescript
import requestIp from "request-ip";

export async function createContextFromRequest(req: Request) {
  const ip = requestIp.getClientIp({
    headers: Object.fromEntries(req.headers.entries()),
  });
  // ...
  return { /* ctx */ req: { ip } };
}

export const createContext = async (database?, ip?) => {
  if (ip === undefined) {
    const hdrs = await headers();
    ip = requestIp.getClientIp({
      headers: Object.fromEntries(hdrs.entries()),
    });
  }
  return { /* ctx */ req: { ip } };
};
```

### 2.2 代理头信任机制

项目使用第三方库 [`request-ip`](https://www.npmjs.com/package/request-ip) 提取 IP。该库默认按以下优先级检查请求头（从高到低）：

1. `X-Client-IP`
2. `X-Forwarded-For`（取最左边/第一个 IP）
3. `X-Real-IP`
4. `X-Cluster-Client-IP`
5. `X-Forwarded`
6. `Forwarded-For`
7. `Forwarded`
8. （回退到）socket 的 `remoteAddress`

**⚠️ 安全注意事项**:
- 项目未对 `request-ip` 配置可信代理范围（`trustProxy`），默认会信任所有代理头
- 若部署在未受信任的反向代理后，攻击者可通过伪造 `X-Forwarded-For` 头绕过 IP 限流
- 建议在生产部署前配置可信代理列表

### 2.3 Context 传递链

IP 在 context 中的类型定义：

**文件**: [packages/trpc/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/index.ts#L44-L60)

```typescript
export interface Context {
  user: User | null;
  auth?: RequestAuth;
  db: typeof db;
  req: {
    ip: string | null;  // IP 可能为 null（无法识别时）
  };
}
```

Web 应用通过 Next.js route handler 将 context 注入 Hono API：

**文件**: [apps/web/app/api/[[...route]]/route.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/web/app/api/[[...route]]/route.ts#L11-L27)

```typescript
export const nextAuth = createMiddleware(async (c, next) => {
  const ctx = await createContextFromRequest(c.req.raw);
  c.set("ctx", ctx);
  await next();
});

const app = new Hono().basePath("/api").use(nextAuth).route("/", allApp);
```

---

## 3. 限流插件系统与缓存状态

### 3.1 插件架构

限流功能通过插件系统提供，支持热插拔和多后端。

**插件类型定义**: [packages/shared/plugins.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/plugins.ts#L9-L21)

```typescript
export enum PluginType {
  Search = "search",
  Queue = "queue",
  RateLimit = "ratelimit",  // 限流插件类型
  VectorStore = "vectorstore",
}
```

**限流客户端接口**: [packages/shared/ratelimiting.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/ratelimiting.ts#L3-L40)

```typescript
export interface RateLimitConfig {
  name: string;          // 限流规则名称（组成 Key 的一部分）
  windowMs: number;      // 时间窗口（毫秒）
  maxRequests: number;   // 窗口内最大请求数
}

export interface RateLimitClient {
  checkRateLimit(config: RateLimitConfig, key: string): RateLimitResult | Promise<RateLimitResult>;
  reset(config: RateLimitConfig, key: string): void | Promise<void>;
  clear(): void | Promise<void>;
}
```

### 3.2 插件加载与优先级

**文件**: [packages/shared-server/src/plugins.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared-server/src/plugins.ts#L16-L43)

```typescript
export async function loadAllPlugins() {
  // ...
  // Rate limiters (order matters - last one wins)
  await import("@karakeep/plugins/ratelimit-memory");
  await import("@karakeep/plugins/ratelimit-redis");
  // 最后加载的 Redis 插件优先级更高
}
```

`PluginManager.getClient()` 返回**最后注册**的插件客户端：

**文件**: [packages/shared/plugins.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/plugins.ts#L70-L78)

```typescript
static async getClient<T extends PluginType>(type: T): Promise<PluginTypeMap[T] | null> {
  const providers = PluginManager.providersFor(type);
  if (providers.length === 0) return null;
  return await providers[providers.length - 1]!.provider.getClient();
  //                        ^ 取最后一个
}
```

### 3.3 内存存储后端（开发/降级用）

**文件**: [packages/plugins/ratelimit-memory/src/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/plugins/ratelimit-memory/src/index.ts#L1-L86)

实现特点：
- 使用 `Map<string, RateLimitEntry>` 作为存储
- 固定窗口算法（非滑动窗口）
- 1% 概率触发过期清理（概率性清理，避免每次检查都遍历）
- **不支持分布式部署**：多实例间限流状态不共享
- Key 格式：`${config.name}:${key}`

### 3.4 Redis 存储后端（生产用）

**文件**: [packages/plugins/ratelimit-redis/src/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/plugins/ratelimit-redis/src/index.ts#L1-L228)

#### 3.4.1 核心算法：Lua 脚本 + ZSET 滑动窗口

使用 Redis 有序集合（ZSET）实现精确的滑动窗口限流：

```lua
-- 1. 删除时间窗口外的旧条目 (ZREMRANGEBYSCORE)
-- 2. 统计当前窗口内请求数 (ZCARD)
-- 3. 若未超限：添加新条目（时间戳 + 自增序号保证唯一）
-- 4. 设置 Key 过期时间
-- 5. 若超限：返回最旧条目的时间以计算 resetInSeconds
```

Key 设计：
- 主 Key：`ratelimit:v1:${config.name}:${key}` — ZSET，存时间戳
- 辅助 Key：`ratelimit:v1:${config.name}:${key}:seq` — 计数器，生成唯一成员

#### 3.4.2 故障处理：Fail-Open 策略

**文件**: [packages/plugins/ratelimit-redis/src/index.ts#L96-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/plugins/ratelimit-redis/src/index.ts#L96-L103)

```typescript
} catch (error) {
  // On Redis error, fail open (allow the request)
  failOpenLog(
    "warn",
    `Rate limiter failed open due to Redis error: ${error}`,
  );
  return { allowed: true };
}
```

- Redis 不可用时 **放行所有请求**（fail-open）
- 使用 `throttledLogger` 限流日志（30秒内不重复打印）
- 此策略保证可用性，但在 Redis 故障时段会失去限流保护

#### 3.4.3 连接管理与重连

**文件**: [packages/plugins/ratelimit-redis/src/index.ts#L145-L227](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/plugins/ratelimit-redis/src/index.ts#L145-L227)

- `disableOfflineQueue: true`：禁用离线队列，避免请求堆积
- `reconnectStrategy: () => 3000`：每 3 秒尝试重连
- 连接失败后 5 秒退避（`RETRY_BACKOFF_MS = 5_000`），避免频繁重试
- 单例模式 + Promise 去重初始化

---

## 4. 全局限流配置与 Key 生成策略

### 4.1 总开关

**文件**: [packages/shared/config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/config.ts#L181-L182)

```typescript
// 环境变量
RATE_LIMITING_ENABLED: stringBool("false"),  // 默认关闭！
```

`serverConfig.rateLimiting.enabled` 为 `false` 时，所有限流中间件直接放行。

### 4.2 tRPC 全局默认限流

**文件**: [packages/trpc/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/index.ts#L122-L152)

```typescript
// 公开接口限流：每分钟 1000 次
export const publicProcedure = procedure.use(
  createRateLimitMiddleware({
    name: "globalPublic",
    windowMs: 60 * 1000,
    maxRequests: 1000,
  }),
);

// 已认证接口限流：每分钟 3000 次
export const authedProcedure = procedure
  .use(
    createRateLimitMiddleware({
      name: "globalAuthed",
      windowMs: 60 * 1000,
      maxRequests: 3000,
    }),
  )
  .use(function isAuthed(opts) { /* 认证检查 */ });
```

### 4.3 Key 生成策略

tRPC 限流 Key 格式（[packages/trpc/lib/rateLimit.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/lib/rateLimit.ts#L38-L39)）：

```typescript
const userSegment = opts.ctx.user?.id ? `:user:${opts.ctx.user.id}` : "";
const key = `${ip}${userSegment}:${opts.path}`;
```

Hono API 限流 Key 格式（[packages/api/middlewares/rateLimit.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/api/middlewares/rateLimit.ts#L29-L30)）：

```typescript
const userSegment = c.var.ctx.user?.id ? `:user:${c.var.ctx.user.id}` : "";
const key = `${ip}${userSegment}:${config.name}`;
```

Key 策略说明：

| 场景 | Key 示例 |
|------|----------|
| 未登录用户访问 tRPC | `192.168.1.1:bookmarks.list` |
| 已登录用户（ID 为 u123）访问 tRPC | `192.168.1.1:user:u123:bookmarks.list` |
| Hono API 资产上传 | `192.168.1.1:user:u123:assets.upload` |

**关键特性**：
- 已登录用户的限流 Key 绑定 `user.id`，**不受 IP 变化影响**（但仍带 IP 前缀）
- IP 为 `null` 时（无法识别），限流中间件直接跳过（`return next()`）

---

## 5. 各接口级独立限流

除了全局默认限流，敏感接口还叠加了更严格的独立限流：

### 5.1 tRPC Router 层面限流

| 接口 | 配置名 | 窗口 | 最大请求 | 文件 |
|------|--------|------|----------|------|
| 用户注册 `users.create` | `users.create` | 60秒 | 3次 | [routers/users.ts#L34-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L34-L38) |
| 修改密码 `users.changePassword` | `users.changePassword` | 15分钟 | 5次 | [routers/users.ts#L120-L124](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L120-L124) |
| 邮箱验证 `users.verifyEmail` | `users.verifyEmail` | 5分钟 | 10次 | [routers/users.ts#L220-L224](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L220-L224) |
| 重发验证邮件 `users.resendVerificationEmail` | `users.resendVerificationEmail` | 5分钟 | 3次 | [routers/users.ts#L238-L242](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L238-L242) |
| 忘记密码 `users.forgotPassword` | `users.forgotPassword` | 15分钟 | 3次 | [routers/users.ts#L261-L265](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L261-L265) |

### 5.2 Hono REST API 层面限流

| 接口 | 配置名 | 窗口 | 最大请求 | 文件 |
|------|--------|------|----------|------|
| 资产上传 `POST /api/assets` | `assets.upload` | 60秒 | 30次 | [routes/assets.ts#L18-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/api/routes/assets.ts#L18-L22) |

### 5.3 限流执行流程

tRPC 限流中间件流程（[packages/trpc/lib/rateLimit.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/lib/rateLimit.ts#L12-L51)）：

```
请求进入
  │
  ├─ rateLimiting.enabled === false? ──是──► 放行
  │
  ├─ IP 为 null? ──是──► 放行
  │
  ├─ RateLimitClient 未初始化? ──是──► 放行
  │
  ├─ 生成 Key (IP + userID + path/config.name)
  │
  ├─ 调用 checkRateLimit()
  │     │
  │     ├─ 允许 ──► 继续执行
  │     │
  │     └─ 拒绝 ──► 抛出 TRPCError(TOO_MANY_REQUESTS)
  │                    HTTP 429
```

---

## 6. 用户配额系统（业务级防滥用）

### 6.1 配额类型与数据库字段

**文件**: [packages/db/schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/db/schema.ts#L45-L50)

```typescript
// users 表字段
bookmarkQuota: integer("bookmarkQuota"),           // 书签数量上限 (null = 无限)
storageQuota: integer("storageQuota"),             // 存储空间上限字节 (null = 无限)
browserCrawlingEnabled: integer("browserCrawlingEnabled", { mode: "boolean" }),  // 浏览器爬取开关
```

### 6.2 环境变量默认配额

**文件**: [packages/shared/config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/config.ts#L194-L199)

```typescript
FREE_QUOTA_BOOKMARK_LIMIT: z.coerce.number().optional(),
FREE_QUOTA_ASSET_SIZE_BYTES: z.coerce.number().optional(),
FREE_BROWSER_CRAWLING_ENABLED: optionalStringBool(),
PAID_QUOTA_BOOKMARK_LIMIT: z.coerce.number().optional(),
PAID_QUOTA_ASSET_SIZE_BYTES: z.coerce.number().optional(),
PAID_BROWSER_CRAWLING_ENABLED: optionalStringBool(),
```

通过 Stripe 订阅区分免费/付费用户，具体分配逻辑在 `subscriptions` router。

### 6.3 QuotaService 实现

**文件**: [packages/shared-server/src/services/quotaService.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared-server/src/services/quotaService.ts#L1-L96)

#### 书签配额检查

```typescript
static async canCreateBookmark(db, userId) {
  // 读取用户 bookmarkQuota
  // 统计当前 bookmarks 数量
  // 超过则返回 { result: false, error: "..." }
}
```

#### 存储配额检查 + Approval Token 机制

```typescript
static async checkStorageQuota(db, userId, requestedSize): Promise<QuotaApproved> {
  // 1. 读取用户 storageQuota (null = 无限制)
  // 2. 汇总 assets 表中该用户所有资产大小
  // 3. currentUsage + requestedSize > quota 时抛出 StorageQuotaError
  // 4. 通过后返回 QuotaApproved token
}
```

**QuotaApproved Token**（[packages/shared/storageQuota.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/storageQuota.ts#L1-L19)）：

```typescript
export class QuotaApproved {
  private constructor(
    public readonly userId: string,
    public readonly approvedSize: number,
  ) {}
  // 只能通过 QuotaService.checkStorageQuota 创建
  static _create(userId, approvedSize): QuotaApproved { ... }
}
```

这是一个防 TOCTOU（Time-of-check to time-of-use）攻击的设计：资产保存函数 `saveAsset` 必须接收 `QuotaApproved` token 才执行写入，防止检查通过后、写入前配额被并发耗尽。

### 6.4 配额检查的应用场景

| 场景 | 检查方式 | 位置 |
|------|----------|------|
| 创建书签 | `canCreateBookmark` | tRPC bookmarks router |
| 上传截图/PDF | `checkStorageQuota` | crawlerWorker storeScreenshot/storePdf |
| 下载图片/视频 | `checkStorageQuota` | crawlerWorker downloadAndStoreFile |
| 归档网页（monolith） | `checkStorageQuota`（先预估1KB，再按实际大小） | crawlerWorker archiveWebpage |
| 存储大 HTML 内容 | `checkStorageQuota` | crawlerWorker storeHtmlContent |
| 资产上传 API | `checkStorageQuota` | api/utils/upload.ts |

---

## 7. 爬虫层域名限流（对外请求防滥用）

这是对**外部网站**的礼貌限流，防止 Karakeep 爬虫被目标网站封禁。

**文件**: [apps/workers/workers/crawlerWorker.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/workers/workers/crawlerWorker.ts#L2149-L2201)

```typescript
async function checkDomainRateLimit(url, jobId) {
  const config = serverConfig.crawler.domainRatelimiting;
  if (!config) return;  // 未配置则不限流

  const rateLimitClient = await getRateLimitClient();
  const hostname = new URL(url).hostname;

  const result = await rateLimitClient.checkRateLimit(
    { name: "domain-ratelimit", ...config },
    hostname,  // Key 仅为域名，不区分用户
  );

  if (!result.allowed) {
    // +40% 随机抖动防止惊群
    const jitterFactor = 1.0 + Math.random() * 0.4;
    const delayMs = result.resetInSeconds * 1000 * jitterFactor;
    throw new QueueRetryAfterError(`Domain rate limited`, delayMs);
  }
}
```

配置来自环境变量（[packages/shared/config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/config.ts#L133-L134)）：

```typescript
CRAWLER_DOMAIN_RATE_LIMIT_WINDOW_MS: z.coerce.number().min(1).optional(),
CRAWLER_DOMAIN_RATE_LIMIT_MAX_REQUESTS: z.coerce.number().min(1).optional(),
```

两者都设置才生效。被限流的任务会延迟后重试（通过 `QueueRetryAfterError`）。

---

## 8. 人机验证（Turnstile CAPTCHA）

### 8.1 配置

**文件**: [packages/shared/config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/config.ts#L58-L59)

```typescript
TURNSTILE_SITE_KEY: z.string().optional(),
TURNSTILE_SECRET_KEY: z.string().optional(),
// 只要配置了 SITE_KEY 即视为启用
auth.turnstile.enabled = (TURNSTILE_SITE_KEY !== undefined)
```

### 8.2 验证实现

**文件**: [packages/trpc/lib/turnstile.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/lib/turnstile.ts#L13-L71)

```typescript
export async function verifyTurnstileToken(token, remoteIp) {
  if (!serverConfig.auth.turnstile.enabled) return { success: true };

  // POST https://challenges.cloudflare.com/turnstile/v0/siteverify
  // body: secret + response + remoteip(可选)
}
```

使用时将客户端 IP 传给 Cloudflare 辅助风控（[packages/trpc/routers/users.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L68-L82)）：

```typescript
const result = await verifyTurnstileToken(
  input.turnstileToken ?? "",
  ctx.req.ip,  // 传递识别出的客户端 IP
);
```

### 8.3 应用场景

- 用户注册 `users.create`（结合 60秒3次 的限流）

---

## 9. 其他防滥用机制

### 9.1 Demo Mode

**文件**: [packages/trpc/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/index.ts#L96-L104)

```typescript
procedure.use(function isDemoMode(opts) {
  if (serverConfig.demoMode && opts.type == "mutation") {
    throw new TRPCError({ message: "Mutations are not allowed in demo mode", code: "FORBIDDEN" });
  }
  return opts.next();
})
```

Demo 模式下禁止所有写操作（mutation）。

### 9.2 密码防暴力破解

**文件**: [packages/trpc/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/auth.ts#L178-L186)

```typescript
if (!user) {
  // 用户不存在也跑一次 bcrypt 比较，掩盖用户是否存在（防时序攻击）
  await bcrypt.compare(password + "<dummy-salt>", "<dummy-hash>");
  throw new Error("User not found");
}
```

即使邮箱不存在也执行 bcrypt，防止通过响应时间枚举注册邮箱。

### 9.3 API Key 节流更新

**文件**: [packages/trpc/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/auth.ts#L139-L150)

```typescript
// lastUsedAt 10分钟内不重复更新数据库
const tenMinutesAgo = new Date(Date.now() - 10 * 60 * 1000);
if (!apiKey.lastUsedAt || apiKey.lastUsedAt < tenMinutesAgo) {
  database.update(apiKeys).set({ lastUsedAt: new Date() })...
  // Fire and forget，不等待
}
```

防止高频 API Key 调用产生过多 DB 写入。

### 9.4 爬虫 SSRF 防护

**文件**: [apps/workers/network.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/workers/network.ts#L13-L88)

爬虫在访问 URL 前通过 `validateUrl()` 进行 SSRF 防护：
- 检查协议只允许 http/https
- 检查 IP 范围，禁止访问内网/回环/私有地址（`DISALLOWED_IP_RANGES`）
- 通过 DNS 解析验证域名（非代理上下文），防止 DNS Rebinding
- 可通过 `CRAWLER_ALLOWED_INTERNAL_HOSTNAMES` 配置白名单
- DNS 查询结果缓存 5 分钟（LRU Cache，最大 1000 条）

---

## 10. 不同入口拦截点汇总

### 10.1 请求入口总览

```
客户端请求
  │
  ├─ Web 浏览器 (Next.js app)
  │    └─ GET/POST /api/*
  │         └─ [route.ts] createContextFromRequest()
  │              ├─ requestIp.getClientIp() 提取 IP
  │              ├─ API Key / Session 认证
  │              └─ Hono app (注入 ctx)
  │                   ├─ 全局中间件 (logger, CORS, metrics)
  │                   ├─ trpcAdapter (错误映射)
  │                   ├─ /api/trpc/* ──► tRPC globalPublic/globalAuthed 限流
  │                   │                    └─ 各 router 自定义限流
  │                   ├─ /api/v1/* (REST)
  │                   ├─ /api/assets/* ──► assets.upload 限流
  │                   └─ /api/public/*
  │
  ├─ 浏览器扩展 / CLI / Mobile / MCP
  │    └─ 通过 tRPC 客户端或 API Key 访问同一入口
  │
  └─ Workers 后台任务
       └─ 队列消费 (crawler/inference/...)
            └─ checkDomainRateLimit() 域名限流
            └─ QuotaService 业务配额检查
```

### 10.2 各入口限流矩阵

| 入口 | 认证方式 | IP 识别 | 全局限流 | 接口限流 | 业务配额 |
|------|----------|---------|----------|----------|----------|
| `publicProcedure` | 无 | ✅ request-ip | ✅ 60s/1000次 | ✅ 如注册接口 | - |
| `authedProcedure` | Session / API Key | ✅ request-ip | ✅ 60s/3000次 | ✅ 如改密码 | ✅ 书签/存储 |
| `sessionProcedure` | Session（禁止 API Key） | ✅ request-ip | ✅ 继承 authed | 视接口而定 | ✅ |
| REST `/api/assets` | authMiddleware + API Key Scope | ✅ request-ip | - | ✅ 60s/30次 | ✅ 存储配额 |
| REST `/api/v1/*` | authMiddleware | ✅ request-ip | - | - | 视业务而定 |
| Crawler Worker | 队列任务（已认证用户触发） | N/A（对外请求） | - | ✅ 域名级限流 | ✅ 存储配额 |

---

## 11. 风险与改进建议

### 已识别的潜在风险

| 风险点 | 位置 | 说明 |
|--------|------|------|
| 限流默认关闭 | [config.ts#L182](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/config.ts#L182) | `RATE_LIMITING_ENABLED` 默认为 `false`，部署时需显式开启 |
| 代理头无条件信任 | [client.ts#L13-L15](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/web/server/api/client.ts#L13-L15) | `request-ip` 未配置可信代理，可能被伪造头绕过 |
| IP 缺失时无限流 | [rateLimit.ts#L20-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/api/middlewares/rateLimit.ts#L20-L22) | IP 为 null 时直接放行 |
| Redis 故障时 Fail-Open | [ratelimit-redis/index.ts#L96-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/plugins/ratelimit-redis/src/index.ts#L96-L103) | Redis 不可用时段无限流保护 |
| 内存限流失效 | [ratelimit-memory/src/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/plugins/ratelimit-memory/src/index.ts) | 多实例部署时状态不共享 |

### 建议改进项

1. **配置可信代理**：在 `requestIp.getClientIp()` 调用时传入 `trustProxy` 参数，仅信任部署环境中的反向代理 IP
2. **IP 缺失时的兜底**：考虑对无法识别 IP 的请求使用更严格的默认限流（如按 User-Agent 或完全拒绝）
3. **Redis Fail-Open 降级**：Redis 故障时可降级到内存限流（而非完全放行），或触发告警
4. **部署文档**：明确 `RATE_LIMITING_ENABLED=true` 为生产必配项
5. **限流指标暴露**：将限流命中次数接入 Prometheus，便于监控滥用攻击
