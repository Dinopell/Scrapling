# Spider 架构

!!! success "前置条件"

    1. 你已完成或阅读过 [Fetcher 基础](../fetching/choosing_CN.md) 页面，了解不同 fetcher 类型及其适用场景。
    2. 你已完成或阅读过 [主要类](../parsing/main_classes_CN.md) 页面，了解 [Selector](../parsing/main_classes_CN.md#selector) 和 [Response](../fetching/choosing_CN.md#response-object) 类。

Scrapling 的 spider 系统是一个受 Scrapy 启发的异步爬取框架，专为并发、多 session 爬取而设计，并内置暂停/恢复支持。它将 Scrapling 的解析引擎与 fetcher 整合为统一的爬取 API，同时增加了调度、并发控制和检查点功能。

如果你熟悉 Scrapy，会很快上手。如果不熟悉，也不用担心——系统设计得直观易懂。

## 数据流

下图展示了爬取运行时数据在 spider 系统中的流动方式：

<img src="../assets/spider_architecture.png" title="Spider architecture diagram by @TrueSkills" alt="Spider architecture diagram by @TrueSkills" style="width: 70%;"/>

运行 spider 时，大致按以下步骤进行（省略部分细节）：

1. **Spider** 生成第一批 `Request` 对象。默认情况下，它为 `start_urls` 中的每个 URL 创建一个请求，但你可以重写 `start_requests()` 以实现自定义逻辑。
2. **Scheduler** 接收请求并将其放入优先级队列，并为它们创建指纹。优先级较高的请求会先出队。
3. **Crawler Engine** 向 **Scheduler** 请求出队下一个请求，同时遵守并发限制（全局和按域名）及下载延迟。如果启用了 `robots_txt_obey`，引擎会在继续之前检查该域名的 robots.txt 规则——不允许的请求会被静默丢弃。引擎收到请求后，将其传递给 **Session Manager**，后者根据请求的 `sid`（session ID）将其路由到正确的 session。
4. **session** 获取页面并向 **Crawler Engine** 返回 [Response](../fetching/choosing_CN.md#response-object) 对象。引擎记录统计信息并检查响应是否被拦截。如果响应被拦截，引擎会将请求重试最多 `max_blocked_retries` 次。当然，拦截检测和被拦截请求的重试逻辑可以自定义。
5. **Crawler Engine** 将 [Response](../fetching/choosing_CN.md#response-object) 传递给请求的回调。回调要么产出字典（作为爬取条目处理），要么产出后续请求（发送到调度器排队）。
6. 从步骤 2 开始循环，直到调度器为空且没有活跃任务，或 spider 被暂停。
7. 如果在启动 spider 时设置了 `crawldir`，**Crawler Engine** 会定期将检查点（待处理请求 + 已见 URL 集合）保存到磁盘。优雅关闭（Ctrl+C）时会保存最终检查点。下次使用相同 `crawldir` 运行 spider 时，会从上次中断处恢复，跳过 `start_requests()` 并恢复调度器状态。


## 组件

### Spider

你与之交互的核心类。你继承 `Spider`，定义 `start_urls` 和 `parse()` 方法，并可选择配置 session 和重写生命周期钩子。

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

引擎编排整个爬取过程。它管理主循环、强制执行并发限制、通过 Session Manager 分发请求，并处理回调返回的结果。你无需直接与它交互——`Spider.start()` 和 `Spider.stream()` 方法会为你处理。

### Scheduler

带内置 URL 去重的优先级队列。请求根据 URL、HTTP 方法、请求体和 session ID 生成指纹。调度器支持 `snapshot()` 和 `restore()` 用于检查点系统，使爬取状态可以保存和恢复。

### Session Manager

管理一个或多个命名的 session 实例。每个 session 是以下之一：

- [FetcherSession](../fetching/static_CN.md)
- [AsyncDynamicSession](../fetching/dynamic_CN.md)
- [AsyncStealthySession](../fetching/stealthy_CN.md)

当请求到达时，Session Manager 根据请求的 `sid` 字段将其路由到正确的 session。session 可以在 spider 启动时启动（默认），也可以懒加载（首次使用时启动）。

### Checkpoint System

可选系统，启用后将爬虫状态（待处理请求 + 已见 URL 指纹）保存到磁盘的 pickle 文件。写入是原子的（临时文件 + 重命名），以防止损坏。检查点按可配置间隔定期保存，并在优雅关闭时保存。成功完成（非暂停）后，检查点文件会自动清理。

### Response Cache

可选缓存，启用开发模式后，将每次获取的响应存储到磁盘，并在后续运行中重放。每个响应以请求指纹为键，序列化为 JSON（body 经 base64 编码，以便二进制内容得以保留）。它用于在不重新请求目标服务器的情况下迭代 `parse()` 逻辑，不适用于生产环境。

### Output

爬取的条目收集在 `ItemList` 中（带有 `to_json()` 和 `to_jsonl()` 导出方法的列表子类）。爬取统计信息记录在 `CrawlStats` 数据类中，包含大量有用信息。


## 与 Scrapy 的对比

如果你来自 Scrapy，以下是 Scrapling spider 系统的对应关系：

| 概念            | Scrapy                        | Scrapling                                                       |
|--------------------|-------------------------------|-----------------------------------------------------------------|
| Spider 定义  | `scrapy.Spider` 子类      | `scrapling.spiders.Spider` 子类                             |
| 初始请求   | `start_requests()`            | `async start_requests()`                                        |
| 回调          | `def parse(self, response)`   | `async def parse(self, response)`                               |
| 跟随链接    | `response.follow(url)`        | `response.follow(url)`                                          |
| 条目输出        | `yield dict` 或 `yield Item`  | `yield dict`                                                    |
| 请求调度 | Scheduler + Dupefilter        | 带内置去重的 Scheduler                           |
| 下载        | Downloader + Middlewares      | 支持多 session 的 Session Manager                      |
| 条目处理    | Item Pipelines                | `on_scraped_item()` 钩子                                        |
| 拦截检测  | 通过自定义中间件    | 内置 `is_blocked()` + `retry_blocked_request()` 钩子       |
| 并发        | `CONCURRENT_REQUESTS` 设置 | `concurrent_requests` 类属性                           |
| 域名过滤   | `allowed_domains`             | `allowed_domains`                                               |
| Robots.txt         | `ROBOTSTXT_OBEY` 设置      | `robots_txt_obey` 类属性                               |
| 暂停/恢复       | `JOBDIR` 设置              | `crawldir` 构造函数参数                                 |
| 导出             | Feed exports                  | `result.items.to_json()` / `to_jsonl()` 或通过钩子自定义 |
| 运行            | `scrapy crawl spider_name`    | `MySpider().start()`                                            |
| 流式处理          | 无                           | `async for item in spider.stream()`                             |
| 多 session      | 无                           | 单个 spider 可使用多种类型的多个 session               |
