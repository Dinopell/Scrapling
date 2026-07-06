# Spiders 架构

Scrapling 的 spider 系统是一个异步爬取框架，专为并发、多会话爬取而设计，并内置暂停/恢复支持。它将 Scrapling 的解析引擎与 fetcher 整合为统一的爬取 API，同时提供调度、并发控制和检查点功能。

## 数据流

下图展示了爬取运行期间数据在 spider 系统中的流动方式：

运行 spider 时，各步骤依次如下：

1. **Spider** 生成第一批 `Request` 对象。默认情况下，它会为 `start_urls` 中的每个 URL 创建一个请求，但你可以重写 `start_requests()` 以实现自定义逻辑。
2. **Scheduler** 接收请求并将其放入优先级队列，并为它们创建指纹。优先级更高的请求会先出队。
3. **Crawler Engine** 向 **Scheduler** 请求出队下一个请求，同时遵守并发限制（全局与按域名）和下载延迟。若启用了 `robots_txt_obey`，引擎会在继续之前检查该域名的 robots.txt 规则——不允许的请求会被静默丢弃。引擎收到请求后，将其交给 **Session Manager**，后者根据请求的 `sid`（会话 ID）将其路由到正确的会话。
4. **会话** 抓取页面并向 **Crawler Engine** 返回 [Response](../fetching/choosing_CN.md#response-object) 对象。引擎记录统计信息并检查是否被拦截。若响应被判定为拦截，引擎会将该请求重试最多 `max_blocked_retries` 次。当然，拦截检测以及对被拦截请求的重试逻辑均可自定义。
5. **Crawler Engine** 将 [Response](../fetching/choosing_CN.md#response-object) 传给该请求的回调。回调要么 `yield` 一个字典（视为已抓取条目），要么 `yield` 一个后续请求（送入调度器排队）。
6. 从第 2 步开始循环，直到调度器为空且没有活跃任务，或 spider 被暂停。
7. 若在启动 spider 时设置了 `crawldir`，**Crawler Engine** 会定期将检查点（待处理请求 + 已见 URL 集合）保存到磁盘。优雅关闭（Ctrl+C）时会保存最终检查点。下次使用相同 `crawldir` 运行 spider 时，会从上次中断处恢复，跳过 `start_requests()` 并还原调度器状态。


## 组件

### Spider

你与之交互的核心类。继承 `Spider`，定义 `start_urls` 和 `parse()` 方法，并可选择配置会话、重写生命周期钩子。

```python
from scrapling.spiders import Spider, Response, Request

class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]

    async def parse(self, response: Response):
        for link in response.css("a::attr(href)").getall():
            yield response.follow(link, callback=self.parse_page)

    async def parse_page(self, response: Response):
        yield {"title": response.css("h1::text").get("")}
```

### Crawler Engine

引擎编排整个爬取过程。它管理主循环、强制执行并发限制、通过 Session Manager 分发请求，并处理回调返回的结果。你无需直接与之交互——`Spider.start()` 和 `Spider.stream()` 会替你处理。

### Scheduler

带内置 URL 去重的优先级队列。请求根据其 URL、HTTP 方法、请求体和会话 ID 生成指纹。调度器支持 `snapshot()` 和 `restore()`，供检查点系统保存与恢复爬取状态。

### Session Manager

管理一个或多个命名会话实例。每个会话为以下之一：

- [FetcherSession](../fetching/static_CN.md)
- [AsyncDynamicSession](../fetching/dynamic_CN.md)
- [AsyncStealthySession](../fetching/stealthy_CN.md)

请求到达时，Session Manager 根据请求的 `sid` 字段将其路由到对应会话。会话可在 spider 启动时一并启动（默认），也可懒加载（首次使用时才启动）。

### Checkpoint System

可选系统；启用后会将爬虫状态（待处理请求 + 已见 URL 指纹）以 pickle 文件写入磁盘。写入为原子操作（临时文件 + 重命名），避免损坏。检查点按可配置间隔定期保存，并在优雅关闭时保存。正常完成（非暂停）后会自动清理检查点文件。

### Response Cache

可选缓存；在启用开发模式时，将每次抓取的响应存盘，并在后续运行中回放。每个响应以请求指纹为键，序列化为 JSON（正文 base64 编码以保留二进制内容）。用于在不重复请求目标服务器的情况下迭代 `parse()` 逻辑，而非用于生产环境。

### Output

抓取的条目收集在 `ItemList` 中（列表子类，提供 `to_json()` 和 `to_jsonl()` 导出方法）。爬取统计记录在 `CrawlStats` 数据类中，包含大量有用信息。


## 与 Scrapy 对比

若你来自 Scrapy，Scrapling spider 系统的对应关系如下：

| Concept            | Scrapy                        | Scrapling                                                       |
|--------------------|-------------------------------|-----------------------------------------------------------------|
| Spider definition  | `scrapy.Spider` subclass      | `scrapling.spiders.Spider` subclass                             |
| Initial requests   | `start_requests()`            | `async start_requests()`                                        |
| Callbacks          | `def parse(self, response)`   | `async def parse(self, response)`                               |
| Following links    | `response.follow(url)`        | `response.follow(url)`                                          |
| Item output        | `yield dict` or `yield Item`  | `yield dict`                                                    |
| Request scheduling | Scheduler + Dupefilter        | Scheduler with built-in deduplication                           |
| Downloading        | Downloader + Middlewares      | Session Manager with multi-session support                      |
| Item processing    | Item Pipelines                | `on_scraped_item()` hook                                        |
| Blocked detection  | Through custom middlewares    | Built-in `is_blocked()` + `retry_blocked_request()` hooks       |
| Concurrency        | `CONCURRENT_REQUESTS` setting | `concurrent_requests` class attribute                           |
| Domain filtering   | `allowed_domains`             | `allowed_domains`                                               |
| Robots.txt         | `ROBOTSTXT_OBEY` setting      | `robots_txt_obey` class attribute                               |
| Pause/Resume       | `JOBDIR` setting              | `crawldir` constructor argument                                 |
| Export             | Feed exports                  | `result.items.to_json()` / `to_jsonl()` or custom through hooks |
| Running            | `scrapy crawl spider_name`    | `MySpider().start()`                                            |
| Streaming          | N/A                           | `async for item in spider.stream()`                             |
| Multi-session      | N/A                           | Multiple sessions with different types per spider               |
