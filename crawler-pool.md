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

- 发起 GET 请求（`fetchWithProxy` + 代理 + 5 秒超时）
- 读取 `Content-Type` 响应头，剥离 charset 等参数后标准化
- 失败时返回 `null`，后续按 HTML 路径处理

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

浏览器内所有请求（包括子资源）都走这个代理。CDP 的 `Fetch.authRequired` 事件处理代理认证。

**路径 B：HTTP 请求（browserless 模式 / content-type 探测 / 文件下载）**

[fetchWithProxy()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L418-L486) 通过 `getProxyAgent()` 创建 `HttpProxyAgent` / `HttpsProxyAgent`：

- 每次请求根据 URL 协议选 http 或 https 代理
- 检查 `noProxy` 列表决定是否绕过

### 4.4 noProxy 绕过逻辑

[matchesNoProxy()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L229-L238) 支持三种模式匹配：
- 精确匹配：`example.com`
- 域后缀匹配：`.example.com` 匹配所有子域
- 通配：`.` 匹配所有域名

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
  // 1. 获取信号量租约
  // 2. 调用 runner.run()
  // 3. 根据返回结果分支处理：
  //    ├─ RPC error → 指数退避 + runNumber++ → continue
  //    ├─ rate_limit → sleep(delayMs) + 不递增 → continue
  //    ├─ error → sleep(1000) + runNumber++ → continue
  //    └─ success → break
}
```

| 结果类型 | 处理方式 | runNumber 变化 | 退避 |
|---------|---------|--------------|------|
| RPC 级错误 | `runner.run()` 抛异常 | `++` | `rand(0, min(5000×2^n, 60000))` |
| rate_limit | 返回 `{type:"rate_limit"}` | 不变 | `delayMs`（由 crawler 指定） |
| 业务级错误 | 返回 `{type:"error"}` | `++` | `1000 ms`（恒定） |
| 成功 | 返回 `{type:"success"}` | - | - |

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

[RestateSemaphore](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L83-L88) 在 dispatcher 层实现：

- 信号量容量 = `opts.concurrency`
- 每次 run 先 `acquire`，完成后 `release`
- 支持 priority 排队和 groupId 隔离

---

## 八、安全防护

### 8.1 URL 安全校验

[validateUrl()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L137-L223) 在多个位置被调用：

- 协议白名单：只允许 http/https
- IP 黑名单：拒绝回环/私有/保留地址（防止 SSRF）
- DNS 解析 + 缓存（LRU 1000 条，5 分钟 TTL）
- **代理上下文跳过 DNS**：如果请求走代理，DNS 由代理端解析，本地不做

### 8.2 CDP 重定向守卫

在 [crawlPage()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L677-L724) 中：

- 通过 CDP `Fetch.requestPaused` 拦截所有 3xx 重定向
- 对重定向目标 URL 执行 `validateUrl()`
- 不合法 → `Fetch.failRequest`（阻止重定向）
- 合法 → `Fetch.continueRequest`（放行）

### 8.3 子资源请求拦截

[route("**/*")](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L752-L796)：

- 阻止 audio/video 资源
- 对每个 http(s) 子请求执行 `validateUrl()`
- 不合法 → `route.abort("blockedbyclient")`

### 8.4 超时与中断保护

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
2. runCrawler() 入口
   ├─ checkDomainRateLimit()
   │    └─ 命中限速 → throw QueueRetryAfterError → 延迟重试（不计次数）
   ├─ selectRunProxies()        ← 随机选代理，整个 run 复用
   ├─ getContentType()          ← MIME 嗅探 + 代理 + 5s 超时
   ├─ 根据 content-type 分支：
   │    ├─ PDF/图片 → handleAsAssetBookmark()（直接下载，不经过浏览器）
   │    └─ HTML → crawlAndParseUrl()
   │         ├─ crawlPage()
   │         │    ├─ 获取 browser 实例（全局 or 按需）
   │         │    ├─ browser 不可用 → browserlessCrawlPage()  ← 可用性探测
   │         │    ├─ 创建 BrowserContext（注入代理配置）
   │         │    ├─ CDP 安装重定向守卫 + 子资源安全校验
   │         │    ├─ 导航 + 等待加载（带超时/中断）
   │         │    ├─ 抓取 HTML + 截图 + PDF（并行）
   │         │    └─ finally: 关闭 page/context/browser（带超时保护）
   │         ├─ shouldRetryCrawlStatusCode()?
   │         │    ├─ 403/429/5xx + 有重试余量 → throw Error → 队列重试
   │         │    └─ 403/429/5xx + 重试耗尽 → 继续 parse + 落库（Fallback）
   │         └─ 解析 + 存储 + 入队下游任务
   └─ 完成 → onComplete 回调
3. 未捕获异常 → onError 回调
   └─ numRetriesLeft == 0 → 标记永久失败
```

---

## 十、关键设计要点

1. **浏览器不是池**：单实例全局共享，每个任务创建独立的 BrowserContext 做隔离；按需模式下每次任务创建/销毁完整浏览器实例
2. **代理不中途切换**：一次 run 内代理固定；重试时（新 run）才重新随机选择
3. **两类重试**：普通错误消耗重试次数 + 退避递增；限速错误不消耗次数 + 按指定延迟重试
4. **403/429/5xx Fallback**：重试耗尽后不抛弃已抓取内容，继续解析并写入数据库，仅记录错误状态码
5. **Restate 两层重试**：SDK 级 retryPolicy（未捕获异常）与 handler 内 while 自循环（绝大多数场景）并存，runner 层禁用重试
6. **可用性探测前置**：globalBrowser 判定（browserless 降级）和 getContentType MIME 嗅探（处理路径分支）均在实际爬取前完成
7. **多层安全防护**：URL 校验（DNS + IP 黑名单）在 fetch、浏览器导航、CDP 重定向、子资源请求四层均被执行
8. **超时无处不在**：所有异步操作都有 `raceWith` + 超时 + 中断信号保护，防止任何操作无限挂起
9. **泄漏兜底**：Context Reaper 定期扫描回收超龄上下文，page/context 关闭本身也有超时保护
10. **队列后端可插拔**：liteque（SQLite）和 restate 两种后端，QueueRetryAfterError 在适配层统一转换为后端原生的延迟重试语义
