# Crawler 浏览器实例池、代理切换与重试退避协同脉络

## 一、整体架构概览

Crawler 的并发、代理与重试并非在同一个类里实现，而是分散在三个层次中：

```
┌──────────────────────────────────────────────────────────────────┐
│  Queue 层（liteque / restate）                                    │
│    ── 任务入队、出队、重试计数、退避延迟                            │
├──────────────────────────────────────────────────────────────────┤
│  CrawlerWorker 层（crawlerWorker.ts）                             │
│    ── 浏览器实例管理、代理选择、页面爬取、状态码判断、触发重试         │
├──────────────────────────────────────────────────────────────────┤
│  Network 层（network.ts）                                        │
│    ── 代理 URL 选择、代理 Agent 构建、URL 安全校验、重定向跟踪       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 二、浏览器实例管理（非"池"，而是单实例 + 按需创建）

Karakeep **没有**实现一个浏览器实例池（browser pool），而是采用"全局单实例 + 按需连接"模式。

### 2.1 全局单实例模式（默认）

- 入口：[launchBrowser()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L299-L330)
- 启动时调用 `startBrowserInstance()` 连接到远程浏览器（通过 WebSocket 或 CDP）
- 连接失败 → 5 秒后递归重试 `setTimeout(() => launchBrowser(), 5000)`
- 断线重连：`globalBrowser.on("disconnected", () => launchBrowser())`
- 用 `browserMutex`（async-mutex）保护并发访问
- **shutdown 早退**：
  - 连接失败重试前检查 `exitAbortController.signal.aborted` → 为 true 则 log "won't retry" 后 return，不再重试
  - 断线重连回调前检查 `exitAbortController.signal.aborted` → 为 true 则 log "won't restart it" 后 return，不再重连

```ts
let globalBrowser: Browser | undefined;   // 唯一全局实例
const browserMutex = new Mutex();
```

### 2.2 按需连接模式（browserConnectOnDemand）

- 配置 `CRAWLER_BROWSER_CONNECT_ON_DEMAND=true` 时，启动时不连接浏览器
- 每次 `crawlPage()` 时调用 `startBrowserInstance()` 创建新连接
- 任务完成后关闭浏览器实例：`browser.close()`

### 2.3 浏览器不可用时降级

在 [crawlPage()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L532-L1133) 中：

```
获取 browser 实例
  ↓
browser === undefined ?
  ├─ YES → browserlessCrawlPage()（纯 HTTP 请求，无截图/PDF）
  └─ NO  → 创建 BrowserContext → 创建 Page → 导航 → 抓取
```

用户级别也可禁用浏览器爬取：`userData.browserCrawlingEnabled === false` → 走 browserless 路径。

### 2.4 BrowserContext 生命周期

每个爬取任务创建一个独立的 `BrowserContext`：

1. **创建**：`browser.newContext({ proxy: proxyConfig, ... })` → 注册到 `activeContexts` Map
2. **使用**：创建 Page → 导航 → 抓取 HTML/截图/PDF
3. **清理**（finally 块）：
   - `page.close()` 带 5 秒超时
   - `context.close()` 带 10 秒超时
   - 按需模式下额外 `browser.close()`

### 2.5 泄漏回收：Context Reaper

[startContextReaper()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L218-L268)

- 每 5 分钟扫描 `activeContexts`
- 超过 `jobTimeoutSec + 30s + 5min` 的上下文被标记为过期
- 尝试 `context.close()`，10 秒超时
- 关闭成功 → 从 Map 删除；关闭失败 → 保留让下次再试
- shutdown 早退：`exitAbortController.signal` abort 时 `clearInterval(intervalId)`，停止调度

### 2.6 与浏览器共生的全局状态：globalBlocker 与 globalCookies

两者均在 [CrawlerWorker.ensureInitialized()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L336-L365) 中初始化，与浏览器实例同生命周期。

**globalBlocker（广告拦截器）**
- 初始化条件：`serverConfig.crawler.enableAdblocker == true`
- 加载方式：`PlaywrightBlocker.fromPrebuiltFull(fetchWithProxy, { cachePath: os.tmpdir()/karakeep_adblocker.bin })`
- **引导期依赖**：`fromPrebuiltFull` 的第一个参数传入 `fetchWithProxy` 作为 HTTP 客户端，用于下载 EasylistFull 等远端广告规则列表。这意味着：
  - 广告规则下载本身也经过完整安全链路（代理选择 + SSRF 校验 + manual redirect 逐跳校验）
  - fetchWithProxy 不依赖 globalBlocker，globalBlocker 依赖 fetchWithProxy 完成初始化
  - 初始化期间下载请求不会被 adblocker 自身拦截（此时 globalBlocker 尚未就绪）
- 失败降级：加载失败不抛出异常，仅打 error 日志，不启用拦截功能
- 注入路径：[crawlPage()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L738-L741) 创建 Page 后调用 `globalBlocker.enableBlockingInPage(nextPage)`

**globalCookies（共享 Cookie）**
- 初始化：`loadCookiesFromFile()` 从 `CRAWLER_BROWSER_COOKIE_PATH` 读取 JSON 文件，经 `cookiesSchema`（zod）验证后赋值给 `globalCookies`
- 失败行为：读取/解析失败 → throw Error，初始化流程终止
- 注入路径：[crawlPage()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L619-L624) 创建 BrowserContext 后、创建 Page 前调用 `context.addCookies(globalCookies)`
- 空值跳过：`globalCookies.length == 0` 时不注入

> **生命周期对齐**：两者均为模块级全局变量，初始化一次后复用所有爬取任务。按需模式下浏览器每次销毁重建，但 globalBlocker/globalCookies 不重建。

---

## 三、可用性探测

### 3.1 globalBrowser 可用性判定

[crawlPage()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L574-L592) 中获取浏览器实例后立即判断：

```ts
browser = serverConfig.crawler.browserConnectOnDemand
  ? startBrowserInstance()  // 按需连接：每次都新建
  : globalBrowser;          // 单实例模式：取全局

if (!browser) {
  return browserlessCrawlPage(...);  // 浏览器不可用 → 降级为纯 HTTP
}
```

降级发生的触发条件：
1. `globalBrowser` 未初始化（启动时连接失败或按需模式下启动失败）
2. 用户级 `browserCrawlingEnabled === false`（先于浏览器可用性判断）

降级后行为：调用 [browserlessCrawlPage()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L491-L530)，使用 `fetchWithProxy` 发起纯 HTTP GET 请求，不经过 Playwright，无法生成截图和 PDF。

### 3.2 getContentType MIME 嗅探

[runCrawler()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L2254-L2285) 入口处先探测内容类型再分支处理：

```ts
const contentType = precrawledArchiveAssetId
  ? ASSET_TYPES.TEXT_HTML                          // 已预爬取，直接假定 HTML
  : await getContentType(url, jobId, abortSignal, runProxy);

// 根据 Content-Type 决定处理路径
if (contentType === ASSET_TYPES.APPLICATION_PDF) {
  handleAsAssetBookmark(url, "pdf", ...);          // 直接下载，不经过浏览器
} else if (IMAGE_ASSET_TYPES.has(contentType) && ...) {
  handleAsAssetBookmark(url, "image", ...);        // 直接下载，不经过浏览器
} else {
  crawlAndParseUrl(...);                           // HTML 页面：走浏览器爬取
}
```

[getContentType()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L1621-L1670) 实现：

- **发起完整 GET 请求**（不是 HEAD）：`fetchWithProxy(url, { method: "GET", signal: timeout(5s)+abortSignal }, runProxy)`
- 读取 `Content-Type` 响应头，剥离 charset 等参数后标准化
- 失败时返回 `null`，后续按 HTML 路径处理
- **注意**：由于使用完整 GET（响应体也会被传输），5 秒超时内无法完成则整体中断

探测失败的 fallback：`contentType === null` 时跳过 PDF/图片分支，进入 `crawlAndParseUrl()` 走浏览器爬取路径。

---

## 四、代理切换机制

### 4.1 代理配置结构

[RunProxyConfig](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L244-L248)：

```ts
interface RunProxyConfig {
  httpProxy: string | undefined;
  httpsProxy: string | undefined;
  noProxy: string[] | undefined;   // 绕过代理的域名模式列表
}
```

### 4.2 "Run 级"代理选择：一次爬取任务只选一次

[selectRunProxies()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L254-L261) 在 `runCrawler()` 入口处调用一次：

```ts
const runProxy = selectRunProxies();  // 随机选一个代理，整个 run 复用
```

- 从 `serverConfig.proxy.httpProxy`（数组）中随机选一个
- 从 `serverConfig.proxy.httpsProxy`（数组）中随机选一个
- **关键设计**：代理在整个爬取 run 期间固定不变，不会中途切换

### 4.3 代理注入的两个路径

**路径 A：Playwright 浏览器爬取**

[getPlaywrightProxyConfig()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/crawlerWorker.ts#L172-L188) 将 `RunProxyConfig` 转换为 Playwright 的 `BrowserContextOptions.proxy`：

```ts
browser.newContext({ proxy: proxyConfig, ... })
```

浏览器内所有请求（包括子资源）都走这个代理。代理认证通过 CDP 的 `Fetch.authRequired` 事件处理。

**CDP Fetch.authRequired 代理认证链路**
1. `context.newCDPSession(page)` 为 Page 创建 CDP 会话
2. `cdpSession.send("Fetch.enable", { handleAuthRequests: true, patterns: [{ urlPattern: "*" }] })` 启用请求拦截，开启认证处理
3. 监听 `cdpSession.on("Fetch.authRequired")` 事件：
   - `event.authChallenge.source === "Proxy"` 且 `proxyConfig` 含 username/password → 响应 `ProvideCredentials`，携带 `username` 和 `password`
   - 其他情况（非代理认证或无认证信息）→ 响应 `Default`，让浏览器自行处理
4. `cdpSession.send("Fetch.continueWithAuth", { requestId, authChallengeResponse })` 提交认证决策
5. 错误吞掉：`send()` 的 `catch()` 为空，避免请求已取消导致的异常冒泡

**路径 B：HTTP 请求（browserless 模式 / content-type 探测 / 文件下载）**

[fetchWithProxy()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L418-L486) 通过 `getProxyAgent()` 创建 `HttpProxyAgent` / `HttpsProxyAgent`：

- 每次请求根据 URL 协议选 http 或 https 代理
- 检查 `noProxy` 列表决定是否绕过

### 4.4 noProxy 绕过逻辑

[matchesNoProxy()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L229-L238) 调用 `hostnameMatchesAnyPattern`，支持三种模式匹配：

| 匹配规则 | pattern 示例 | 匹配 hostname |
|---------|-------------|---------------|
| 精确匹配 | `example.com` | `example.com` |
| 显式后缀匹配（pattern 以 `.` 开头） | `.example.com` | `www.example.com`、`sub.example.com` 等所有子域 |
| **隐式后缀匹配**（pattern 不以 `.` 开头） | `example.com` | `www.example.com`、`sub.example.com` 等所有子域（等价于 `.example.com`） |
| 通配 | `.` | 所有域名 |

隐式后缀匹配的实现：[hostnameMatchesPattern L115](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L115) `hostname.endsWith("." + pattern)`，即 pattern 不加点前缀时自动隐式地也匹配子域。注意精确匹配和隐式后缀匹配会同时生效。

### 4.5 重要结论：代理不会"切换"

代理在 run 开始时随机选定，后续不再变更。如果某次爬取因代理问题失败，**重试时**会重新调用 `selectRunProxies()` 随机选择新的代理（因为重试是新的 run）。

---

## 五、重试与退避机制

### 5.1 两条重试路径

```
              ┌───────────────────────────────┐
              │        任务执行失败              │
              └──────────┬────────────────────┘
                         │
          ┌──────────────┴───────────────┐
          ▼                              ▼
  普通错误 throw Error          限速错误 throw QueueRetryAfterError
          │                              │
          ▼                              ▼
  计入重试次数                    不计入重试次数
  (runNumber++)                  (runNumber 不变)
  退避由队列实现决定               延迟 delayMs 后重新入队
```

### 5.2 普通错误重试

**触发条件** — [shouldRetryCrawlStatusCode()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/crawlerWorker.ts#L137-L142)：

- HTTP 403 / 429 / 5xx → 如果 `numRetriesLeft > 0`，抛出 Error 触发重试
- 其他异常（网络错误、超时、解析失败等）→ 自然 throw → 队列层捕获后重试

**队列配置** — 两个爬取队列均设置 `numRetries: 5`：

- [LinkCrawlerQueue](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/shared-server/src/queues.ts#L89-L97)：普通优先级
- [LowPriorityCrawlerQueue](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/shared-server/src/queues.ts#L101-L109)：低优先级（导入等场景）

**退避策略**因队列后端而异：

| 后端 | 场景 | 退避策略 | 代码位置 |
|------|------|---------|---------|
| **liteque** | 所有错误 | 由 liteque 库内部决定（指数退避） | [queue-liteque/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-liteque/src/index.ts) |
| **restate** | RPC 级错误（runner 服务不可达） | 指数退避 + full jitter：`delayMs = rand(0, min(5000×2^runNumber, 60000))` | [dispatcher.ts#L169-L173](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L169-L173) |
| **restate** | 业务级错误（runner 返回 `{type:"error"}`） | 恒定 1 秒：`ctx.sleep(1000)` | [dispatcher.ts#L203-L206](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L203-L206) |

### 5.3 403/429/5xx 重试耗尽的 Fallback

[crawlAndParseUrl()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L1932-L1941) 中：

```ts
if (shouldRetryCrawlStatusCode(statusCode)) {
  if (numRetriesLeft > 0) {
    throw new Error(`Received status code ${statusCode}. Will retry ...`);
  }
  // numRetriesLeft == 0：不抛异常，继续往下走
  logger.info(`Received status code ${statusCode} on latest retry. Proceeding ...`);
}

// 无论状态码如何，继续解析和落库
const { metadata, readableContent } = await runParseSubprocess(htmlContent, ...);
await db.update(bookmarkLinks).set({
  title: meta.title,
  description: meta.description,
  crawlStatusCode: statusCode,  // 状态码被记录
  // ...
});
```

**关键行为**：HTTP 403/429/5xx 耗尽重试后，不会让任务整体失败，而是把已获取到的（可能是错误页）HTML 照常解析并写入数据库，同时把错误状态码记入 `crawlStatusCode` 字段。这样用户至少能看到爬取尝试过。

### 5.4 限速重试（QueueRetryAfterError）

[QueueRetryAfterError](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/shared/queueing.ts#L10-L18) 是一种特殊的重试信号：

- **不消耗重试次数**：`runNumber` 不递增
- **指定延迟时间**：由 `delayMs` 参数控制
- **唯一触发点**：[checkDomainRateLimit()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L2153-L2201)

**领域限速逻辑**：

```
runCrawler() 入口
  ↓
checkDomainRateLimit(url, jobId)
  ↓
从 URL 提取 hostname
  ↓
rateLimitClient.checkRateLimit({ maxRequests, windowMs }, hostname)
  ↓
allowed ?
  ├─ YES → 继续执行
  └─ NO  → 计算延迟（resetInSeconds × 1000 × [1.0~1.4] jitter）
           → throw new QueueRetryAfterError(message, delayMs)
```

**队列后端如何处理 QueueRetryAfterError**：

| 后端 | 转换方式 | 效果 |
|------|---------|------|
| **liteque** | 包装层 catch → 转为 liteque 的 `RetryAfterError(delayMs)` | 延迟后重试，不计数 |
| **restate** | runner 层 catch → 返回 `{ type: "rate_limit", delayMs }` → dispatcher 层 `ctx.sleep(delayMs)` 后重试，不递增 runNumber | 同上 |

### 5.5 重试耗尽时的处理

[CrawlerWorker.build() 的 onError 回调](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L400-L453)：

- `job.numRetriesLeft == 0` → 标记为永久失败
- 更新 `bookmarkLinks.crawlStatus = "failure"`
- 清理 pending 状态的 tagging/summarization/embedding 任务

> 注意：这是**最后兜底**的失败处理。403/429/5xx 重试耗尽不会走到这里，而是在 crawlAndParseUrl 内部 fallback 解析。只有发生未捕获异常（如网络断开、浏览器崩溃等）且重试耗尽才会触发 onError。

---

## 六、Restate 两层重试机制

使用 restate 作为队列后端时，存在两层重试叠加：

### 6.1 第一层：Restate-SDK 级 retryPolicy

[dispatcher.ts#L46-L54](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L46-L54) 为 dispatcher service 配置的 SDK 级重试：

```ts
retryPolicy: {
  maxAttempts: NUM_RETRIES,     // 5
  initialInterval: { seconds: 5 },
  maxInterval: { minutes: 1 },
},
```

- **触发时机**：dispatcher handler 本身抛出未捕获异常时（如 `restate.CancelledError` 被 re-throw）
- **退避策略**：指数退避，初始 5 秒，最大 1 分钟
- **实际很少触发**：绝大多数错误都在 while 循环内被 tryCatch 捕获

### 6.2 第二层：Dispatcher Handler 内 while 自循环

[dispatcher.ts#L93](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L93)：

```ts
let runNumber = 0;
while (runNumber <= NUM_RETRIES) {  // runNumber 从 0 到 5，共 6 次
  // 1. semaphore.acquire() 获取信号量租约 → 返回 leaseId
  //    ├─ idempotencyKey 已存在 → 跳过（leaseId === false）
  //    └─ 取消/失败 → 内部 release 后抛错
  // 2. 调用 runner.run(jobData)
  // 3. 根据返回结果分支处理（每条分支都 release）：
  //    ├─ RPC error → release → 指数退避 + runNumber++ → continue
  //    ├─ rate_limit → release → sleep(delayMs) + 不递增 → continue
  //    ├─ error → onError → release → sleep(1000) + runNumber++ → continue
  //    └─ success → onCompleted → release → break
}
```

| 结果类型 | 处理方式 | runNumber 变化 | 退避 |
|---------|---------|--------------|------|
| RPC 级错误 | `runner.run()` 抛异常 | `++` | `rand(0, min(5000×2^n, 60000))` |
| rate_limit | 返回 `{type:"rate_limit"}` | 不变 | `delayMs`（由 crawler 指定） |
| 业务级错误 | 返回 `{type:"error"}` | `++` | `1000 ms`（恒定） |
| 成功 | 返回 `{type:"success"}` | - | - |
| 幂等跳过 | `acquire()` 返回 `false` | - | 直接 return |

### 6.3 Runner 层：禁用重试

[runner.ts#L52-L55](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/runner.ts#L52-L55) 显式配置：

```ts
retryPolicy: { maxAttempts: 1 },  // No retries at runner level
```

所有重试逻辑统一由 dispatcher 控制，避免分散。

### 6.4 两层重试的触发边界

```
dispatcher handler 执行
  ↓
tryCatch(runner.run(jobData))
  ↓
  ├─ 未捕获异常 → throw → 触发第一层（SDK retryPolicy 指数退避）
  └─ 已捕获 → 返回结果 → 进入第二层（while 自循环按类型分支）
```

> 只有 `restate.CancelledError` 会被 re-throw 到 SDK 层触发第一层重试，其余所有错误都在第二层 while 循环内消化。

---

## 七、队列调度与并发控制

### 7.1 Worker 创建

[CrawlerWorker.build()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L370-L464)：

```ts
createRunner(queue, funcs, {
  pollIntervalMs: 1000,
  timeoutSecs: serverConfig.crawler.jobTimeoutSec,
  concurrency: serverConfig.crawler.numWorkers,
})
```

- `concurrency`：同时运行的最大任务数（对应 `CRAWLER_NUM_WORKERS`）
- `timeoutSecs`：单任务超时（对应 `CRAWLER_JOB_TIMEOUT_SEC`）

### 7.2 两个优先级队列

| 队列 | 用途 | numRetries |
|------|------|-----------|
| `link_crawler_queue` | 用户直接创建的书签 | 5 |
| `low_priority_crawler_queue` | 批量导入等低优先级场景 | 5 |

分离队列防止低优先级任务阻塞正常爬取的并发度。

### 7.3 Restate 后端的并发控制

[RestateSemaphore](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/semaphore.ts#L537-L582) 在 dispatcher 层实现分布式信号量：

**LeaseId 生成与超时机制**
- `leaseId` = `ctx.awakeable().id`，是每次 `acquire()` 调用时 restate 生成的唯一标识
- 租约超时：`leaseDurationMs = Math.ceil(opts.timeoutSecs * 1.5 * 1000)`，即任务超时的 1.5 倍
- 超时回收：`pruneExpiredLeases()` 在每次 `acquire/release/tick` 时扫描 `metadata.leases`，删除 `expiry <= now` 的过期租约，释放被异常中断任务占用的槽位
- 服务端状态：`metadata.leases[leaseId] = now + leaseDurationMs` 记录每个租约的过期时间

**四条释放分支**（在 dispatcher while 循环内）：

| 分支 | 触发条件 | 代码位置 | 释放时机 | 后续行为 |
|------|---------|---------|---------|---------|
| **1. RPC 级错误** | `runner.run()` 抛出异常（服务不可达等） | [dispatcher.ts#L146](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L146) | **release 先于 onError** | 指数退避 + `runNumber++` → continue |
| **2. rate_limit** | runner 返回 `{type:"rate_limit"}`（QueueRetryAfterError） | [dispatcher.ts#L184](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L184) | release → ctx.sleep(delayMs) | 不递增 runNumber → continue |
| **3. 业务级错误** | runner 返回 `{type:"error"}`（普通异常） | [dispatcher.ts#L201](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L201) | **onError 先于 release**（保证 inFlight 一致） | sleep(1000) + `runNumber++` → continue |
| **4. 成功** | runner 返回 `{type:"success"}` | [dispatcher.ts#L220](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L220) | **onCompleted 先于 release**（保证 inFlight 一致） | break → 退出循环 |

> **一致性约定**：分支 3/4 先通知 runner 回调（onError/onCompleted 会更新 inFlight 计数），再释放信号量；分支 1 因为 runner 本身不可达，只能先释放再尝试通知。

**其他特性**：
- 信号量容量 = `opts.concurrency`
- 支持 priority 排队（数字越小优先级越高）和 groupId 隔离
- 支持 idempotencyKey 防重复入队
- `queueSize()` 可查询 pending/running 任务数

---

## 八、安全防护

### 8.1 URL 安全校验（SSRF 防护核心）

[validateUrl()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L137-L223) 在四个位置被调用（fetch、浏览器导航、CDP 重定向、子资源请求），逐层拦截 SSRF 攻击。

**校验流程**：

```
1. URL 语法解析 → 失败则拒绝
2. 协议白名单 → 仅允许 http: / https:
3. hostname 非空检查
4. 白名单旁路 → CRAWLER_ALLOWED_INTERNAL_HOSTNAMES 匹配则直接放行
5. IP 字面量检查 → isAddressForbidden()
6. 代理上下文 → runningInProxyContext == true 时跳过 DNS
7. DNS 解析 + 缓存 → 逐条检查 resolved 地址 isAddressForbidden()
```

**IP 黑名单（14 项）** — [DISALLOWED_IP_RANGES](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L13-L30)：

| 类别 | range() 值 | 说明 |
|------|-----------|------|
| **IPv4（8 项）** | | |
| | `unspecified` | 0.0.0.0/8 |
| | `broadcast` | 255.255.255.255/32 |
| | `multicast` | 224.0.0.0/4 |
| | `linkLocal` | 169.254.0.0/16 |
| | `loopback` | 127.0.0.0/8 |
| | `private` | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 |
| | `reserved` | 其他 IETF 保留地址 |
| | `carrierGradeNat` | 100.64.0.0/10 |
| **IPv6（6 项）** | | |
| | `uniqueLocal` | fc00::/7 |
| | `6to4` | 2002::/16（RFC 3056 过渡机制） |
| | `teredo` | 2001::/32（RFC 4380 隧道） |
| | `benchmarking` | 2001:2::/48（RFC 5180） |
| | `deprecated` | 已弃用地址（RFC 3879） |
| | `discard` | 100::/64（RFC 6666 丢弃前缀） |

**IPv4-mapped IPv6 处理** — [isAddressForbidden()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L75-L88)：

```ts
const parsed = ipaddr.parse(address);
if (parsed.kind() === "ipv6" && (parsed as IPv6).isIPv4MappedAddress()) {
  const mapped = (parsed as IPv6).toIPv4Address();  // ::ffff:127.0.0.1 → 127.0.0.1
  return DISALLOWED_IP_RANGES.has(mapped.range());  // 用 IPv4 range 判定
}
return DISALLOWED_IP_RANGES.has(parsed.range());
```

防止攻击者用 `::ffff:127.0.0.1` 等 IPv4-mapped 地址绕过 IPv4 黑名单。

**白名单旁路** — [isHostnameAllowedForInternalAccess()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L127-L135)：

- 配置 `CRAWLER_ALLOWED_INTERNAL_HOSTNAMES`（字符串数组）
- 匹配规则与 `noProxy` 一致：精确匹配 / 显式后缀匹配（`.example.com`）/ 隐式后缀匹配（`example.com` 也匹配子域）/ `.` 通配
- **优先级最高**：在 IP 检查和 DNS 解析之前执行，命中后直接返回 `{ ok: true }`
- 用途：允许爬取内部服务（如 wiki.company.internal）而不触发 SSRF 拦截

**DNS 解析 + 并行容错** — [resolveHostAddresses()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L38-L73)：

```ts
const results = await Promise.allSettled([
  resolver.resolve4(hostname),  // A 记录
  resolver.resolve6(hostname),  // AAAA 记录
]);
```

- `Promise.allSettled` 并行查询 A + AAAA 记录，任一成功即收集地址
- 全部失败才抛出聚合错误信息
- DNS 缓存：LRU 1000 条，5 分钟 TTL
- 超时：`CRAWLER_IP_VALIDATION_DNS_RESOLVER_TIMEOUT_SEC`

**代理上下文跳过 DNS**：`runningInProxyContext == true` 时，DNS 由代理端解析，本地不做 DNS 查询，避免解析结果与代理端不一致。

### 8.2 HTTP 重定向处理（fetchWithProxy manual redirect）

[fetchWithProxy()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L418-L486) 使用 `redirect: "manual"` 手动处理重定向，每跳都重新执行安全校验：

```
while (true) {
  1. getProxyAgent(currentUrl) ← 每跳重新选择代理（按 URL 协议 + noProxy）
  2. validateUrl(currentUrl, !!agent) ← 每跳重新校验（含 DNS 解析）
  3. fetch(currentUrl, { redirect: "manual" })
  4. 非 3xx → return response
  5. 3xx → 解析 Location → 方法切换规则 → continue
}
```

**每跳重新校验**：重定向目标可能指向内网地址（如 `http://169.254.169.254/latest/meta-data/`），因此每次跳转都必须重新执行 `validateUrl()`，包括 DNS 解析和 IP 黑名单检查。

**每跳重新选择代理**：`getProxyAgent(currentUrl)` 根据当前 URL 的协议（http/https）和 `noProxy` 列表决定是否使用代理及使用哪个代理，跨协议重定向时代理可能切换。

**方法切换规则** — [L472-L481](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L472-L481)：

| 状态码 | 方法变化 | Body 变化 |
|--------|---------|----------|
| **303** | 任何方法 → **GET** | 清除 body + 删除 content-length |
| **301/302** | GET/HEAD → 不变 | 不变 |
| **301/302** | POST/PUT 等 → **GET** | 清除 body + 删除 content-length |
| **307/308** | 不变 | 不变（保留原始方法和 body） |

- 默认最大重定向次数：5 次（`maxRedirects`）
- 超出限制 → throw `Too many redirects`

### 8.3 CDP 重定向守卫

在 [crawlPage()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L677-L724) 中：

- 通过 CDP `Fetch.requestPaused` 拦截所有 3xx 重定向
- 对重定向目标 URL 执行 `validateUrl()`
- 不合法 → `Fetch.failRequest`（阻止重定向）
- 合法 → `Fetch.continueRequest`（放行）

> 注意：CDP 层的重定向拦截是浏览器内部的，与 fetchWithProxy 的 HTTP 层重定向处理互补。浏览器导航走 CDP，纯 HTTP 请求走 fetchWithProxy manual redirect。

### 8.4 子资源请求拦截

[route("**/*")](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L752-L796)：

- 阻止 audio/video 资源
- 对每个 http(s) 子请求执行 `validateUrl()`
- 不合法 → `route.abort("blockedbyclient")`

### 8.5 超时与中断保护

所有关键操作都使用 `raceWith()` + `timeoutRace()` + `abortRace()`：

| 操作 | 超时 | 代码位置 |
|------|------|---------|
| 页面导航 | `CRAWLER_NAVIGATE_TIMEOUT_SEC` | crawlPage L852-L858 |
| 等待加载 | 5 秒 | crawlPage L880-L888 |
| 截图 | `CRAWLER_SCREENSHOT_TIMEOUT_SEC` | crawlPage L937-L953 |
| PDF 生成 | `CRAWLER_SCREENSHOT_TIMEOUT_SEC` | crawlPage L986-L1000 |
| Page 关闭 | 5 秒 | crawlPage L1057-L1067 |
| Context 关闭 | 10 秒 | crawlPage L1084-L1094 |
| content-type 探测 | 5 秒 | getContentType L1646 |
| browserless 请求 | 5 秒 | browserlessCrawlPage L514 |
| HTML 解析子进程 | `CRAWLER_PARSE_TIMEOUT_SEC` | runParseSubprocess L1188-L1196 |

---

## 九、协同流程总结

一个完整的爬取任务生命周期：

```
1. 队列出队任务
2. Dispatcher while 循环（restate 后端）
   ├─ semaphore.acquire() → 返回 leaseId（idempotent 已存在则跳过）
   ├─ runner.run() 调用
   └─ 按结果类型 release 租约，分支重试或继续
3. runCrawler() 入口
   ├─ checkDomainRateLimit()
   │    └─ 命中限速 → throw QueueRetryAfterError → delayMs 后重试（不计次数）
   ├─ selectRunProxies()        ← 随机选代理，整个 run 复用
   ├─ getContentType()          ← MIME 嗅探 + 代理 + 5s 超时
   ├─ 根据 content-type 分支：
   │    ├─ PDF/图片 → handleAsAssetBookmark()（直接下载，不经过浏览器）
   │    └─ HTML → crawlAndParseUrl()
   │         ├─ crawlPage()
   │         │    ├─ 获取 browser 实例（全局 or 按需）
   │         │    ├─ browser 不可用 → browserlessCrawlPage()  ← 可用性探测
   │         │    ├─ 创建 BrowserContext（注入代理配置）
   │         │    ├─ context.addCookies(globalCookies)       ← 注入共享 Cookie
   │         │    ├─ 创建 CDP 会话 → 开启 Fetch 拦截（含认证处理）
   │         │    ├─ globalBlocker.enableBlockingInPage(page) ← 注入广告拦截
   │         │    ├─ CDP Fetch.authRequired → ProvideCredentials 代理认证
   │         │    ├─ CDP 安装重定向守卫 + 子资源安全校验
   │         │    ├─ 导航 + 等待加载（带超时/中断）
   │         │    ├─ 抓取 HTML + 截图 + PDF（并行）
   │         │    └─ finally: 关闭 page/context/browser（带超时保护）
   │         ├─ shouldRetryCrawlStatusCode()?
   │         │    ├─ 403/429/5xx + 有重试余量 → throw Error → 队列重试
   │         │    └─ 403/429/5xx + 重试耗尽 → 继续 parse + 落库（Fallback）
   │         └─ 解析 + 存储 + 入队下游任务
   └─ 完成 → onComplete 回调
4. 未捕获异常 → onError 回调
   └─ numRetriesLeft == 0 → 标记永久失败
5. Shutdown 时 exitAbortController.abort()
   ├─ launchBrowser 连接失败/断线 → 不再重试
   └─ Context Reaper 定时器 → 被 clearInterval 停止
```

---

## 十、关键设计要点

1. **浏览器不是池**：单实例全局共享，每个任务创建独立的 BrowserContext 做隔离；按需模式下每次任务创建/销毁完整浏览器实例
2. **代理不中途切换**：一次 run 内代理固定；重试时（新 run）才重新随机选择
3. **两类重试**：普通错误消耗重试次数 + 退避递增；限速错误不消耗次数 + 按指定延迟重试
4. **403/429/5xx Fallback**：重试耗尽后不抛弃已抓取内容，继续解析并写入数据库，仅记录错误状态码
5. **Restate 两层重试**：SDK 级 retryPolicy（未捕获异常）与 handler 内 while 自循环（绝大多数场景）并存，runner 层禁用重试
6. **可用性探测前置**：globalBrowser 判定（browserless 降级）和 getContentType 完整 GET 请求 Content-Type 嗅探（不是 HEAD）均在实际爬取前完成
7. **多层 SSRF 防护**：14 项 IP 黑名单（IPv4 8 项 + IPv6 6 项）+ IPv4-mapped IPv6 反绕过 + 白名单旁路（`CRAWLER_ALLOWED_INTERNAL_HOSTNAMES`）+ DNS `allSettled` 并行容错，在 fetch/manual redirect、CDP 重定向、子资源请求四层均被执行
8. **重定向逐跳安全校验**：fetchWithProxy 每次重定向都重新 `validateUrl()` + 重新 `getProxyAgent()` 选择代理，防止开放重定向 SSRF；方法切换遵循 HTTP 语义（303→GET，301/302 非 GET/HEAD→GET，307/308 不变）
9. **noProxy 四层匹配**：精确匹配 + 显式后缀（`.example.com`）+ 隐式后缀（`example.com` 自动匹配子域）+ 通配（`.`），白名单旁路复用同一匹配函数
10. **超时无处不在**：所有异步操作都有 `raceWith` + 超时 + 中断信号保护，防止任何操作无限挂起
11. **泄漏兜底**：Context Reaper 定期扫描回收超龄上下文，page/context 关闭本身也有超时保护
12. **队列后端可插拔**：liteque（SQLite）和 restate 两种后端，QueueRetryAfterError 在适配层统一转换为后端原生的延迟重试语义
13. **与浏览器共生的全局状态**：globalBlocker（广告拦截）和 globalCookies（共享 Cookie）初始化一次后复用所有爬取任务，与浏览器实例同生命周期；globalBlocker 初始化时将 `fetchWithProxy` 传入作为 HTTP 客户端下载远端广告规则，引导期下载请求本身也经过完整安全链路（代理+SSRF+逐跳重定向校验）但不被自身拦截
14. **CDP 代理认证链路完整闭环**：`Fetch.enable` 开启认证拦截 → `Fetch.authRequired` 事件判定来源 → 提供 `ProvideCredentials` 携带代理用户名密码 → `Fetch.continueWithAuth` 提交决策
15. **Semaphore 四条释放路径**：RPC 错误先 release 再 onError；业务错误/成功先 onError/onCompleted 再 release；rate_limit release 后 sleep 不计数；各路径严格遵循 inFlight 计数一致性约定
16. **Lease 超时自动回收**：semaphore 租约超时设为任务超时的 1.5 倍，每次 `acquire/release/tick` 时自动扫描并释放过期租约，防止任务异常中断导致的永久占用
17. **Shutdown 早退分支**：launchBrowser 连接失败/断线重连、Context Reaper 定时器均监听 `exitAbortController.signal`，在优雅退出时立即停止重试/调度，避免 shutdown 挂起
