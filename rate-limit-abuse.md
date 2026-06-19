# Karakeep 限流与防滥用入口边界分析

本文档梳理 Karakeep 项目中与限流、防滥用相关的所有代码路径，包括 IP 识别、代理头信任、用户配额、缓存状态与不同入口的拦截逻辑。

---

## 1. 整体架构概览

Karakeep 的防滥用体系分为七层：

| 层级 | 作用 | 核心模块 |
|------|------|----------|
| IP 识别层 | 获取真实客户端 IP，用于限流 Key | `request-ip` 库 |
| 限流插件层 | 提供可插拔的限流存储后端（内存/Redis） | `PluginManager` + RateLimit 插件 |
| 全局限流中间件 | 在 tRPC/Hono 入口对所有请求做基础限流 | `publicProcedure` / `authedProcedure` |
| 接口级限流 | 对敏感接口做更严格的独立限流 | 各 router 中的 `createRateLimitMiddleware` |
| 降级限流 | 高负载时将任务降级到低优先级队列 | `shouldUseLowPriorityQueues` + 双层限流 |
| 业务配额层 | 对用户资源（书签数、存储空间、爬取能力）做配额限制 | `QuotaService` + 数据库字段 |
| 爬虫层限流 | 对外部网站域名的爬取频率做限制 | `checkDomainRateLimit` |
| 人机验证层 | Cloudflare Turnstile CAPTCHA | `verifyTurnstileToken` |
| 日志去重层 | 防止事件日志被洪水般的请求淹没 | `emitRateLimitedEvent` |

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

#### Key 示例汇总

| 场景 | Key 格式示例 |
|------|-------------|
| 未登录用户访问 tRPC | `192.168.1.1:bookmarks.list` |
| 已登录用户（u123）访问 tRPC | `192.168.1.1:user:u123:bookmarks.createBookmark` |
| Hono API 资产上传（已登录） | `192.168.1.1:user:u123:assets.upload` |

### 4.4 IP 变化对已登录用户限流的影响

**核心结论：IP 变化会导致已登录用户的限流 Key 变化，相当于重置限流计数。**

由于 Key 格式为 `${ip}:user:${userId}:${path}`，IP 位于 Key 的最前面，因此：

| 场景 | 限流行为 |
|------|----------|
| 用户从 WiFi 切换到 4G | IP 变化 → Key 变化 → 限流计数重置，用户可继续请求 |
| 同一账户在多个设备同时使用 | 各设备独立计数，不累加（但分别受全局限流约束） |
| 同一 NAT 下多个用户使用同一 IP | 每人独立计数（因 user.id 不同），互不影响 |
| 未登录用户共享同一 IP | 共享配额（Key 中无 user.id） |

**设计意图分析**：
- 这种设计介于"纯 IP 限流"和"纯用户限流"之间
- 好处：同一 IP 下不同用户互不干扰；用户换 IP 后不会被之前的恶意行为牵连
- 坏处：攻击者可通过频繁更换 IP（如代理池）绕过已登录用户的接口级限流
- 注意：全局 `globalAuthed` 限流（3000次/分钟）同样受 IP 变化影响

**特殊例外**：书签高容量检测限流（`shouldUseLowPriorityQueues`）的 Key 仅为 `user.id`，不受 IP 变化影响，详见第 5.2 节。

### 4.5 IP 为 null 时的处理

IP 识别失败时（`ip === null`），所有限流中间件直接跳过（`return opts.next()` / `return next()`），请求不受限制。

---

## 5. 各接口级独立限流

除了全局默认限流，敏感接口还叠加了更严格的独立限流。所有接口级限流均使用相同的 Key 生成策略（IP + userID + 接口名）。

### 5.1 tRPC Router 层面限流完整清单

#### 用户与认证相关

| 接口 | 配置名 | 窗口 | 最大请求 | 过程类型 | 文件 |
|------|--------|------|----------|----------|------|
| 用户注册 `users.create` | `users.create` | 60秒 | 3次 | publicProcedure | [routers/users.ts#L34-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L34-L38) |
| 修改密码 `users.changePassword` | `users.changePassword` | 15分钟 | 5次 | usersProcedure | [routers/users.ts#L120-L124](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L120-L124) |
| 邮箱验证 `users.verifyEmail` | `users.verifyEmail` | 5分钟 | 10次 | publicProcedure | [routers/users.ts#L220-L224](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L220-L224) |
| 重发验证邮件 `users.resendVerificationEmail` | `users.resendVerificationEmail` | 5分钟 | 3次 | publicProcedure | [routers/users.ts#L238-L242](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L238-L242) |
| 忘记密码 `users.forgotPassword` | `users.forgotPassword` | 15分钟 | 3次 | publicProcedure | [routers/users.ts#L261-L265](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L261-L265) |
| 重置密码 `users.resetPassword` | `users.resetPassword` | 5分钟 | 10次 | publicProcedure | [routers/users.ts#L278-L282](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/users.ts#L278-L282) |

#### 书签相关

| 接口 | 配置名 | 窗口 | 最大请求 | 过程类型 | 文件 |
|------|--------|------|----------|----------|------|
| 创建书签 `bookmarks.createBookmark` | `bookmarks.createBookmark` | 60秒 | 30次 | bookmarksProcedure | [routers/bookmarks.ts#L187-L191](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/bookmarks.ts#L187-L191) |
| 重爬书签 `bookmarks.recrawlBookmark` | `bookmarks.recrawlBookmark` | 30分钟 | 200次 | bookmarksProcedure | [routers/bookmarks.ts#L714-L718](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/bookmarks.ts#L714-L718) |
| 摘要书签 `bookmarks.summarizeBookmark` | `bookmarks.summarizeBookmark` | 30分钟 | 100次 | bookmarksProcedure | [routers/bookmarks.ts#L1225-L1229](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/bookmarks.ts#L1225-L1229) |

#### API Key 相关

| 接口 | 配置名 | 窗口 | 最大请求 | 过程类型 | 用途 | 文件 |
|------|--------|------|----------|----------|------|------|
| API Key 交换 `apiKeys.exchange` | `apiKey.exchange` | 15分钟 | 10次 | publicProcedure | 浏览器扩展用用户名密码换 API Key | [routers/apiKeys.ts#L136-L140](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/apiKeys.ts#L136-L140) |
| API Key 验证 `apiKeys.validate` | `apiKey.validate` | 60秒 | 30次 | publicProcedure | 验证 API Key 是否有效 | [routers/apiKeys.ts#L197-L201](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/apiKeys.ts#L197-L201) |

#### 备份相关

| 接口 | 配置名 | 窗口 | 最大请求 | 过程类型 | 文件 |
|------|--------|------|----------|----------|------|
| 触发备份 `backups.triggerBackup` | `backups.triggerBackup` | 1小时 | 5次 | backupsProcedure | [routers/backups.ts#L47-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/backups.ts#L47-L51) |

#### 邀请相关（admin 邀请新用户）

| 接口 | 配置名 | 窗口 | 最大请求 | 过程类型 | 文件 |
|------|--------|------|----------|----------|------|
| 获取邀请信息 `invites.get` | `invites.get` | 60秒 | 10次 | publicProcedure | [routers/invites.ts#L118-L122](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/invites.ts#L118-L122) |
| 接受邀请 `invites.accept` | `invites.accept` | 60秒 | 10次 | publicProcedure | [routers/invites.ts#L155-L159](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/invites.ts#L155-L159) |

#### 协作者相关（列表共享）

| 接口 | 配置名 | 窗口 | 最大请求 | 过程类型 | 文件 |
|------|--------|------|----------|----------|------|
| 添加协作者 `lists.addCollaborator` | `lists.addCollaborator` | 15分钟 | 20次 | listsProcedure | [routers/lists.ts#L264-L268](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/lists.ts#L264-L268) |

### 5.2 书签创建的双层限流与队列降级

书签创建接口使用了**双层限流**设计，第一层硬拒绝，第二层软降级：

**文件**: [packages/trpc/routers/bookmarks.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/routers/bookmarks.ts#L154-L182)

```
第一层：createRateLimitMiddleware
  配置名: bookmarks.createBookmark
  窗口: 60秒
  上限: 30次
  Key: IP + userID + path
  超限行为: 直接返回 429 TOO_MANY_REQUESTS

         ↓ （通过后）

第二层：shouldUseLowPriorityQueues（高容量检测）
  配置名: bookmarks.createBookmark.highVolume
  窗口: 5分钟
  上限: 30次
  Key: 仅 user.id（不受 IP 影响）
  超限行为: 不拒绝，而是将爬虫任务发到低优先级队列
```

第二层限流 Key **仅使用 `user.id`**，不包含 IP，因此：
- 用户换 IP 无法绕过此限制
- 用于检测用户是否在短时间内大量创建书签
- 触发后仅降级到低优先级队列，不阻塞用户使用

### 5.3 Hono REST API 层面限流

| 接口 | 配置名 | 窗口 | 最大请求 | 认证方式 | 文件 |
|------|--------|------|----------|----------|------|
| 资产上传 `POST /api/assets` | `assets.upload` | 60秒 | 30次 | API Key + Scope | [routes/assets.ts#L18-L22](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/api/routes/assets.ts#L18-L22) |

### 5.4 限流执行流程

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

**注意**：全局限流 + 接口级限流是**独立计数**的，两者都通过才能执行。即一个请求会消耗两个限流配额。

---

## 6. 事件日志去重限流

除了请求限流，项目还使用限流机制对事件日志做去重，防止日志洪水。

**文件**: [packages/trpc/lib/rateLimitedEvent.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/lib/rateLimitedEvent.ts#L10-L40)

```typescript
export function emitRateLimitedEvent<F extends EventLogType>(
  eventName: F,
  dedupKey: string,    // 去重 Key
  windowMs: number,    // 时间窗口
  fields: EventFields<F>,
): void {
  // 使用限流客户端检查：窗口内最多 1 次
  // 未超限则记录日志，超限则静默丢弃
  // 异步执行，永不阻塞请求
}
```

特点：
- 使用限流插件的 `maxRequests: 1` 实现"窗口内只记一次"的去重效果
- 异步 fire-and-forget，失败不影响请求
- 用途：防止同一类错误/事件在短时间内重复刷屏

---

## 7. 用户配额系统（业务级防滥用）

### 7.1 配额类型与数据库字段

**文件**: [packages/db/schema.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/db/schema.ts#L45-L50)

```typescript
// users 表字段
bookmarkQuota: integer("bookmarkQuota"),           // 书签数量上限 (null = 无限)
storageQuota: integer("storageQuota"),             // 存储空间上限字节 (null = 无限)
browserCrawlingEnabled: integer("browserCrawlingEnabled", { mode: "boolean" }),  // 浏览器爬取开关
```

### 7.2 环境变量默认配额

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

### 7.3 QuotaService 实现

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

### 7.4 配额检查的应用场景

| 场景 | 检查方式 | 位置 |
|------|----------|------|
| 创建书签 | `canCreateBookmark` | tRPC bookmarks router |
| 上传截图/PDF | `checkStorageQuota` | crawlerWorker storeScreenshot/storePdf |
| 下载图片/视频 | `checkStorageQuota` | crawlerWorker downloadAndStoreFile |
| 归档网页（monolith） | `checkStorageQuota`（先预估1KB，再按实际大小） | crawlerWorker archiveWebpage |
| 存储大 HTML 内容 | `checkStorageQuota` | crawlerWorker storeHtmlContent |
| 资产上传 API | `checkStorageQuota` | api/utils/upload.ts |

---

## 8. 爬虫层域名限流（对外请求防滥用）

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

**Key 特点**：仅使用域名作为 Key，**所有用户共享同一域名的配额**。这是为了避免单个域名被 Karakeep 的所有用户集中访问而被封禁。

---

## 9. 人机验证（Turnstile CAPTCHA）

### 9.1 配置

**文件**: [packages/shared/config.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/config.ts#L58-L59)

```typescript
TURNSTILE_SITE_KEY: z.string().optional(),
TURNSTILE_SECRET_KEY: z.string().optional(),
// 只要配置了 SITE_KEY 即视为启用
auth.turnstile.enabled = (TURNSTILE_SITE_KEY !== undefined)
```

### 9.2 验证实现

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

### 9.3 应用场景

- 用户注册 `users.create`（结合 60秒3次 的限流）

---

## 10. 其他防滥用机制

### 10.1 Demo Mode

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

### 10.2 密码防暴力破解

**文件**: [packages/trpc/auth.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/auth.ts#L178-L186)

```typescript
if (!user) {
  // 用户不存在也跑一次 bcrypt 比较，掩盖用户是否存在（防时序攻击）
  await bcrypt.compare(password + "<dummy-salt>", "<dummy-hash>");
  throw new Error("User not found");
}
```

即使邮箱不存在也执行 bcrypt，防止通过响应时间枚举注册邮箱。

### 10.3 API Key 节流更新

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

### 10.4 爬虫 SSRF 防护

**文件**: [apps/workers/network.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/workers/network.ts#L13-L88)

爬虫在访问 URL 前通过 `validateUrl()` 进行 SSRF 防护：
- 检查协议只允许 http/https
- 检查 IP 范围，禁止访问内网/回环/私有地址（`DISALLOWED_IP_RANGES`）
- 通过 DNS 解析验证域名（非代理上下文），防止 DNS Rebinding
- 可通过 `CRAWLER_ALLOWED_INTERNAL_HOSTNAMES` 配置白名单
- DNS 查询结果缓存 5 分钟（LRU Cache，最大 1000 条）

---

## 11. 不同入口拦截点汇总

### 11.1 请求入口总览

```
客户端请求
  │
  ├─ Web 浏览器 (Next.js app)
  │    ├─ GET/POST /api/auth/*  (NextAuth 独立路由)
  │    │    └─ [route.tsx] authHandler (NextAuth)
  │    │         ├─ ❌ 不经过 Hono 中间件链
  │    │         ├─ ❌ 不经过 tRPC 限流
  │    │         ├─ POST /api/auth/callback/credentials  (密码登录)
  │    │         ├─ POST /api/auth/signout               (登出)
  │    │         └─ GET  /api/auth/session               (会话查询)
  │    │
  │    └─ GET/POST /api/*  (除 /api/auth/* 外)
  │         └─ [route.ts] createContextFromRequest()
  │              ├─ requestIp.getClientIp() 提取 IP
  │              ├─ API Key / Session 认证
  │              └─ Hono app (注入 ctx)
  │                   ├─ 全局中间件 (logger, CORS, metrics)
  │                   ├─ trpcAdapter (错误映射)
  │                   ├─ /api/trpc/* ──► tRPC globalPublic/globalAuthed 限流
  │                   │                    └─ 各 router 自定义限流
  │                   │                    └─ 业务配额检查（书签/存储）
  │                   ├─ /api/v1/* (REST)
  │                   ├─ /api/assets/* ──► assets.upload 限流 + 存储配额
  │                   └─ /api/public/*
  │
  ├─ 浏览器扩展 / CLI / Mobile / MCP
  │    └─ 通过 tRPC 客户端或 API Key 访问同一入口
  │
  └─ Workers 后台任务
       └─ 队列消费 (crawler/inference/...)
            ├─ checkDomainRateLimit() 域名限流（对外）
            ├─ QuotaService 业务配额检查
            └─ SSRF 防护（对外请求）
```

### 11.2 各入口限流矩阵（完整版）

#### tRPC 过程类型层级

| 过程类型 | 继承关系 | 全局默认限流 | 典型接口 |
|----------|----------|-------------|----------|
| `procedure` | 最底层 | - | - |
| `publicProcedure` | procedure → rateLimit | 60s/1000次 (globalPublic) | 注册、忘记密码、邮箱验证、邀请验证、API Key 交换/验证（注：登录走 NextAuth 独立路由，不经过此过程） |
| `authedProcedure` | procedure → rateLimit → isAuthed | 60s/3000次 (globalAuthed) | 所有需要登录的接口 |
| `sessionProcedure` | authedProcedure → isSession | 继承 globalAuthed | 修改密码、管理 API Key |
| `*Procedure` (scoped) | authedProcedure → createScopedAuthedProcedure | 继承 globalAuthed | bookmarksProcedure、usersProcedure 等 |

#### 各业务入口限流详情

| 入口/接口 | 认证方式 | IP 绑定 | 全局限流 | 接口级限流 | 业务配额 | 其他防护 |
|-----------|----------|---------|----------|-----------|----------|----------|
| **用户注册** `users.create` | 无（tRPC publicProcedure） | ✅ IP+userID（创建前仅 IP） | 60s/1000次 (globalPublic) | 60s/3次 | - | Turnstile CAPTCHA |
| **密码登录** `POST /api/auth/callback/credentials` | 无（NextAuth 独立路由） | ❌ 无（未提取 IP） | ❌ 无（不走 tRPC） | ❌ 无 | - | bcrypt 时序防护（仅防枚举） |
| **OAuth 登录** `GET /api/auth/signin/custom` + callback | 无（NextAuth 独立路由） | ❌ 无（未提取 IP） | ❌ 无（不走 tRPC） | ❌ 无 | - | OAuth 协议本身防护 |
| **查询 Session** `GET /api/auth/session` | 无（Cookie） | ❌ 无（未提取 IP） | ❌ 无（不走 tRPC） | ❌ 无 | - | JWT 签名验证 |
| **登出** `POST /api/auth/signout` | Cookie | ❌ 无（未提取 IP） | ❌ 无（不走 tRPC） | ❌ 无 | - | - |
| **修改密码** `users.changePassword` | Session（tRPC sessionProcedure） | ✅ IP+userID | 60s/3000次 (globalAuthed) | 15min/5次 | - | - |
| **邮箱验证** `users.verifyEmail` | 无 | ✅ IP | 60s/1000次 | 5min/10次 | - | - |
| **重发验证邮件** `users.resendVerificationEmail` | 无 | ✅ IP | 60s/1000次 | 5min/3次 | - | - |
| **忘记密码** `users.forgotPassword` | 无 | ✅ IP | 60s/1000次 | 15min/3次 | - | - |
| **重置密码** `users.resetPassword` | 无 | ✅ IP | 60s/1000次 | 5min/10次 | - | - |
| **创建书签** `bookmarks.createBookmark` | Session/API Key | ✅ IP+userID | 60s/3000次 | 60s/30次 + 5min/30次(降级) | ✅ 书签数配额 | 队列降级 |
| **重爬书签** `bookmarks.recrawlBookmark` | Session/API Key | ✅ IP+userID | 60s/3000次 | 30min/200次 | - | - |
| **摘要书签** `bookmarks.summarizeBookmark` | Session/API Key | ✅ IP+userID | 60s/3000次 | 30min/100次 | - | - |
| **API Key 交换** `apiKeys.exchange` | 无（用户名+密码） | ✅ IP | 60s/1000次 | 15min/10次 | - | - |
| **API Key 验证** `apiKeys.validate` | 无 | ✅ IP | 60s/1000次 | 60s/30次 | - | - |
| **触发备份** `backups.triggerBackup` | Session/API Key | ✅ IP+userID | 60s/3000次 | 1h/5次 | - | - |
| **获取邀请** `invites.get` | 无 | ✅ IP | 60s/1000次 | 60s/10次 | - | - |
| **接受邀请** `invites.accept` | 无 | ✅ IP | 60s/1000次 | 60s/10次 | - | - |
| **添加协作者** `lists.addCollaborator` | Session/API Key | ✅ IP+userID | 60s/3000次 | 15min/20次 | - | - |
| **资产上传** `POST /api/assets` | API Key + Scope | ✅ IP+userID | - | 60s/30次 | ✅ 存储配额 | QuotaApproved Token |
| **爬虫（对外）** `crawlerWorker` | 队列任务 | N/A | - | 域名级（按配置） | ✅ 存储配额 | SSRF 防护 + 抖动重试 |

---

## 12. 风险与改进建议

### 12.1 已识别的潜在风险（按严重度降序）

| 风险点 | 位置 | 严重度 | 说明 |
|--------|------|--------|------|
| NextAuth 路径完全无限流（密码登录、OAuth、session、signout） | [route.tsx#L1-L3](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/web/app/api/auth/[...nextauth]/route.tsx#L1-L3) | 高 (Critical) | 所有 `/api/auth/*` 走独立路由，完全绕过 Hono 中间件 + tRPC 限流体系，攻击者可无限制爆破密码、刷 session、枚举 OAuth |
| NextAuth 路径无 IP 识别 | [route.tsx#L1-L3](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/web/app/api/auth/[...nextauth]/route.tsx#L1-L3) | 高 (Critical) | 未调用 `createContextFromRequest()`，不提取 `X-Forwarded-For` 等，无法做 IP 维度限流/封禁/溯源，撞库攻击不留痕 |
| 限流默认关闭 | [config.ts#L182](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/shared/config.ts#L182) | 高 | `RATE_LIMITING_ENABLED` 默认为 `false`，部署时需显式开启；若忘记，全 tRPC 路径也无限流 |
| 代理头无条件信任 | [client.ts#L13-L15](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/apps/web/server/api/client.ts#L13-L15) | 高 | `request-ip` 未配置 `trustProxy`，攻击者伪造 `X-Forwarded-For` 可绕过所有 tRPC 路径的 IP 维度限流 |
| 已登录用户限流可通过换 IP 绕过 | [rateLimit.ts#L38-L39](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/lib/rateLimit.ts#L38-L39) | 中 | tRPC Key 格式为 `${ip}:user:${id}:${path}`，攻击者通过代理池换 IP 即可重置计数（书签高容量检测除外，其 Key 仅 user.id） |
| IP 缺失时 tRPC 限流直接放行 | [rateLimit.ts#L25-L28](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/trpc/lib/rateLimit.ts#L25-L28) | 中 | IP 为 null 时 `return next()` 直接跳过，若 request-ip 解析失败等同于无限流 |
| Redis 故障时 Fail-Open | [ratelimit-redis/index.ts#L96-L103](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/plugins/ratelimit-redis/src/index.ts#L96-L103) | 中 | Redis 不可用全部放行（可用性优先），此时若遭遇攻击无限流保护 |
| 内存限流失效（多实例部署） | [ratelimit-memory/src/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/45-karakeep/packages/plugins/ratelimit-memory/src/index.ts) | 中 | 多实例时 Map 状态不共享，限流仅按实例粒度，N 实例等效于 N 倍配额 |
| 全局 + 接口级限流双重计数 | 全局中间件 + 各 router | 低 | 一个请求消耗两份配额（globalPublic/Authed + 接口级），调参数时需考虑叠加效应，否则实际限制比预期更严 |

### 12.2 建议改进项（按优先级排序）

**P0（上线前必须修）**

1. **为 NextAuth 路径增加 IP 提取 + 限流 + 失败锁定**：
   - **方案 A（推荐）**：在 `apps/web/app/api/auth/[...nextauth]/route.tsx` 中手动调用 `requestIp.getClientIp()` + 限流客户端，在调用 `authHandler` 之前/之后拦截关键操作
   - **方案 B**：新增 `middleware.ts`（Next.js Middleware）对 `/api/auth/:path*` 做统一限流，不侵入业务代码
   - 至少覆盖：密码登录（`/callback/credentials`）、OAuth callback、signin、session、signout、csrf
   - 建议规则：密码登录 5次/分钟/IP + 10次/小时/邮箱 + 连续失败 5 次临时锁定账户 15 分钟
2. **确保 `RATE_LIMITING_ENABLED=true`**：在部署模板（docker-compose / Helm / env 示例）中默认开启，避免漏配

**P1（安全加固）**

3. **配置 `request-ip` 可信代理范围**：
   - 在 `createContextFromRequest()` 中给 `requestIp.getClientIp()` 传 `{ headers, trustProxy: [...] }` 或配置上游 CDN 的 IP 白名单
   - 否则伪造 `X-Forwarded-For` 可绕过所有 tRPC 限流
4. **tRPC 已登录用户接口增加纯 user.id 维度限流**：
   - 与书签高容量检测一致，对敏感写接口（createBookmark、summarize、addCollaborator 等）加一层 Key 仅为 `${config.name}:user:${userId}` 的限流
   - 防止换 IP 绕过；两层都通过才执行，取更严格者

**P2（可用性与可观测性）**

5. **Redis 故障时降级到内存限流**：将 `fail-open` 改为「先查 Redis，失败则回退到进程内 LRU Map」，避免窗口内完全失控
6. **限流命中指标**：暴露 `rate_limit_hits_total{rule,status}` Prometheus 指标 + 命中后的结构化告警（单规则 5 分钟内命中 >100 次触发告警）
7. **登录失败事件接风控**：`user.login_failed` 事件中补 IP 字段，接入 SIEM，按小时/天检测撞库（同一 IP 尝试大量邮箱）模式
8. **登录失败 N 次后强制 Turnstile**：在 `authorize()` 中根据 IP/邮箱在 Redis 中的失败计数，超过阈值后要求前端带 Turnstile Token 才能继续尝试

**P3（架构优化）**

9. **统一入口**：长期考虑将 NextAuth 迁移到 tRPC router（或在 Hono 层包装 NextAuth），使所有请求走同一中间件链，避免路径分裂导致的「忘记加保护」问题
10. **双维度限流 Key**：将 tRPC 限流 Key 拆为两个独立计数器（IP 维度 + user 维度），取较小值通过；既能限制单点爆破，又能限制账号滥用，且对换 IP 不敏感
