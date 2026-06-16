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

## 三、代理切换机制

### 3.1 代理配置结构

[RunProxyConfig](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L244-L248)：

```ts
interface RunProxyConfig {
  httpProxy: string | undefined;
  httpsProxy: string | undefined;
  noProxy: string[] | undefined;   // 绕过代理的域名模式列表
}
```

### 3.2 "Run 级"代理选择：一次爬取任务只选一次

[selectRunProxies()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L254-L261) 在 `runCrawler()` 入口处调用一次：

```ts
const runProxy = selectRunProxies();  // 随机选一个代理，整个 run 复用
```

- 从 `serverConfig.proxy.httpProxy`（数组）中随机选一个
- 从 `serverConfig.proxy.httpsProxy`（数组）中随机选一个
- **关键设计**：代理在整个爬取 run 期间固定不变，不会中途切换

### 3.3 代理注入的两个路径

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

### 3.4 noProxy 绕过逻辑

[matchesNoProxy()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L229-L238) 支持三种模式匹配：
- 精确匹配：`example.com`
- 域后缀匹配：`.example.com` 匹配所有子域
- 通配：`.` 匹配所有域名

### 3.5 重要结论：代理不会"切换"

代理在 run 开始时随机选定，后续不再变更。如果某次爬取因代理问题失败，**重试时**会重新调用 `selectRunProxies()` 随机选择新的代理（因为重试是新的 run）。

---

## 四、重试与退避机制

### 4.1 两条重试路径

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

### 4.2 普通错误重试

**触发条件** — [shouldRetryCrawlStatusCode()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/crawlerWorker.ts#L137-L142)：

- HTTP 403 / 429 / 5xx → 如果 `numRetriesLeft > 0`，抛出 Error 触发重试
- 其他异常（网络错误、超时、解析失败等）→ 自然 throw → 队列层捕获后重试

**队列配置** — 两个爬取队列均设置 `numRetries: 5`：

- [LinkCrawlerQueue](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/shared-server/src/queues.ts#L89-L97)：普通优先级
- [LowPriorityCrawlerQueue](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/shared-server/src/queues.ts#L101-L109)：低优先级（导入等场景）

**退避策略**因队列后端而异：

| 后端 | 退避策略 | 代码位置 |
|------|---------|---------|
| **liteque** | 由 liteque 库内部决定（指数退避） | [queue-liteque/index.ts](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-liteque/src/index.ts) |
| **restate** | 指数退避 + full jitter：`delayMs = rand(0, min(5000×2^runNumber, 60000))` | [dispatcher.ts#L169-L173](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L169-L173) |

### 4.3 限速重试（QueueRetryAfterError）

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

### 4.4 重试耗尽时的处理

[CrawlerWorker.build() 的 onError 回调](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L400-L453)：

- `job.numRetriesLeft == 0` → 标记为永久失败
- 更新 `bookmarkLinks.crawlStatus = "failure"`
- 清理 pending 状态的 tagging/summarization/embedding 任务

---

## 五、队列调度与并发控制

### 5.1 Worker 创建

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

### 5.2 两个优先级队列

| 队列 | 用途 | numRetries |
|------|------|-----------|
| `link_crawler_queue` | 用户直接创建的书签 | 5 |
| `low_priority_crawler_queue` | 批量导入等低优先级场景 | 5 |

分离队列防止低优先级任务阻塞正常爬取的并发度。

### 5.3 Restate 后端的并发控制

[RestateSemaphore](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/packages/plugins/queue-restate/src/dispatcher.ts#L83-L88) 在 dispatcher 层实现：

- 信号量容量 = `opts.concurrency`
- 每次 run 先 `acquire`，完成后 `release`
- 支持 priority 排队和 groupId 隔离

---

## 六、可用性探测与安全防护

### 6.1 URL 安全校验

[validateUrl()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/network.ts#L137-L223) 在多个位置被调用：

- 协议白名单：只允许 http/https
- IP 黑名单：拒绝回环/私有/保留地址（防止 SSRF）
- DNS 解析 + 缓存（LRU 1000 条，5 分钟 TTL）
- **代理上下文跳过 DNS**：如果请求走代理，DNS 由代理端解析，本地不做

### 6.2 CDP 重定向守卫

在 [crawlPage()](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L677-L724) 中：

- 通过 CDP `Fetch.requestPaused` 拦截所有 3xx 重定向
- 对重定向目标 URL 执行 `validateUrl()`
- 不合法 → `Fetch.failRequest`（阻止重定向）
- 合法 → `Fetch.continueRequest`（放行）

### 6.3 子资源请求拦截

[route("**/*")](file:///d:/fz/0601-2/solo-dogfeeding/code/2-karakeep/apps/workers/workers/crawlerWorker.ts#L752-L796)：

- 阻止 audio/video 资源
- 对每个 http(s) 子请求执行 `validateUrl()`
- 不合法 → `route.abort("blockedbyclient")`

### 6.4 超时与中断保护

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

## 七、协同流程总结

一个完整的爬取任务生命周期：

```
1. 队列出队任务
2. runCrawler() 入口
   ├─ checkDomainRateLimit()
   │    └─ 命中限速 → throw QueueRetryAfterError → 延迟重试（不计次数）
   ├─ selectRunProxies()        ← 随机选代理，整个 run 复用
   ├─ getContentType()          ← fetchWithProxy + 代理 + 5s 超时
   ├─ 根据 content-type 分支：
   │    ├─ PDF/图片 → handleAsAssetBookmark()（直接下载，不经过浏览器）
   │    └─ HTML → crawlAndParseUrl()
   │         ├─ crawlPage()
   │         │    ├─ 获取 browser 实例（全局 or 按需）
   │         │    ├─ browser 不可用 → browserlessCrawlPage()
   │         │    ├─ 创建 BrowserContext（注入代理配置）
   │         │    ├─ CDP 安装重定向守卫 + 子资源安全校验
   │         │    ├─ 导航 + 等待加载（带超时/中断）
   │         │    ├─ 抓取 HTML + 截图 + PDF（并行）
   │         │    └─ finally: 关闭 page/context/browser（带超时保护）
   │         ├─ shouldRetryCrawlStatusCode()?
   │         │    └─ 403/429/5xx + 有重试余量 → throw Error → 队列重试
   │         └─ 解析 + 存储 + 入队下游任务
   └─ 完成 → onComplete 回调
3. 异常时 → onError 回调
   └─ numRetriesLeft == 0 → 标记永久失败
```

---

## 八、关键设计要点

1. **浏览器不是池**：单实例全局共享，每个任务创建独立的 BrowserContext 做隔离；按需模式下每次任务创建/销毁完整浏览器实例
2. **代理不中途切换**：一次 run 内代理固定；重试时（新 run）才重新随机选择
3. **两类重试**：普通错误消耗重试次数 + 退避递增；限速错误不消耗次数 + 按指定延迟重试
4. **多层安全防护**：URL 校验（DNS + IP 黑名单）在 fetch、浏览器导航、CDP 重定向、子资源请求四层均被执行
5. **超时无处不在**：所有异步操作都有 `raceWith` + 超时 + 中断信号保护，防止任何操作无限挂起
6. **泄漏兜底**：Context Reaper 定期扫描回收超龄上下文，page/context 关闭本身也有超时保护
7. **队列后端可插拔**：liteque（SQLite）和 restate 两种后端，QueueRetryAfterError 在适配层统一转换为后端原生的延迟重试语义
