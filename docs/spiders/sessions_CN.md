# Spider Session

!!! success "前置条件"

    1. 你已阅读 [入门指南](getting-started_CN.md) 页面，了解如何创建和运行基础 spider。
    2. 你熟悉 [Fetcher 基础](../fetching/choosing_CN.md) 以及 HTTP、Dynamic 和 Stealthy session 之间的区别。

一个 spider 可以同时使用多个 fetcher session。例如，对简单页面使用快速的 HTTP session，对受保护页面使用 stealth 浏览器 session。本页介绍如何配置和使用 session。

## 什么是 Session？

如你所知，session 是预配置的 fetcher 实例，在爬取期间保持活跃。spider 会复用 session，而不是为每个请求创建新连接或浏览器，这样更快且更节省资源。

默认情况下，每个 spider 会创建一个 [FetcherSession](../fetching/static_CN.md)。你可以通过重写 `configure_sessions()` 方法添加更多 session 或替换默认 session，但必须只使用各 session 的异步版本，如下表所示：


| Session 类型                                    | 适用场景                                 |
|-------------------------------------------------|------------------------------------------|
| [FetcherSession](../fetching/static_CN.md)         | 快速 HTTP 请求，无 JavaScript        |
| [AsyncDynamicSession](../fetching/dynamic_CN.md)   | 浏览器自动化，JavaScript 渲染 |
| [AsyncStealthySession](../fetching/stealthy_CN.md) | 反机器人绕过，Cloudflare 等        |


## 配置 Session

在 spider 上重写 `configure_sessions()` 来设置 session。`manager` 参数是 `SessionManager` 实例。使用 `manager.add()` 注册 session：

```python
from scrapling.spiders import Spider, Response
from scrapling.fetchers import FetcherSession

class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]

    def configure_sessions(self, manager):
        manager.add("default", FetcherSession())

    async def parse(self, response: Response):
        yield {"title": response.css("title::text").get("")}
```

`manager.add()` 方法接受以下参数：

| 参数     | 类型      | 默认值    | 说明                                  |
|--------------|-----------|------------|----------------------------------------------|
| `session_id` | `str`     | *必填* | 在请求中引用此 session 的名称 |
| `session`    | `Session` | *必填* | session 实例                         |
| `default`    | `bool`    | `False`    | 将此 session 设为默认                |
| `lazy`       | `bool`    | `False`    | 仅在首次使用时启动 session       |

!!! note "注意："

    1. 在所有请求中，若未指定使用哪个 session，则使用默认 session。默认 session 通过以下两种方式之一确定：
        1. 添加到 manager 的第一个 session 会自动成为默认 session。
        2. 添加到 manager 时设置 `default=True` 的 session。
    2. 你传入的各 session 实例无需事先由你启动；spider 会检查所有 session，若尚未启动则启动它们。
    3. 若希望某个 session 仅在首次使用时启动，可在将该 session 添加到 manager 时使用 `lazy` 参数。例如：仅在需要时启动浏览器，而非在 spider 启动时启动。

## 多 Session Spider

下面是一个实用示例：对列表页使用快速 HTTP session，对有机器人保护的详情页使用 stealth 浏览器：

```python
from scrapling.spiders import Spider, Response
from scrapling.fetchers import FetcherSession, AsyncStealthySession

class ProductSpider(Spider):
    name = "products"
    start_urls = ["https://shop.example.com/products"]

    def configure_sessions(self, manager):
        # Fast HTTP for listing pages (default)
        manager.add("http", FetcherSession())

        # Stealth browser for protected product pages
        # capture_xhr captures background API calls matching the regex
        manager.add("stealth", AsyncStealthySession(
            headless=True,
            network_idle=True,
            capture_xhr=r"https://api\.shop\.example\.com/.*",
        ))

    async def parse(self, response: Response):
        for link in response.css("a.product::attr(href)").getall():
            # Route product pages through the stealth session
            yield response.follow(link, sid="stealth", callback=self.parse_product)

        next_page = response.css("a.next::attr(href)").get()
        if next_page:
            yield response.follow(next_page)

    async def parse_product(self, response: Response):
        # Access captured XHR/fetch API calls (if capture_xhr was set on the session)
        for xhr in response.captured_xhr:
            self.logger.info(f"Captured API call: {xhr.url} ({xhr.status})")

        yield {
            "name": response.css("h1::text").get(""),
            "price": response.css(".price::text").get(""),
        }
```

关键是 `sid` 参数——它告诉 spider 每个请求使用哪个 session。调用 `response.follow()` 时若不指定 `sid`，会从原始请求继承 session ID。

注意，session 不必来自不同类，也可以是同一类的不同实例、不同配置，例如：

```python
from scrapling.spiders import Spider, Response
from scrapling.fetchers import FetcherSession

class ProductSpider(Spider):
    name = "products"
    start_urls = ["https://shop.example.com/products"]

    def configure_sessions(self, manager):
        chrome_requests = FetcherSession(impersonate="chrome")
        firefox_requests = FetcherSession(impersonate="firefox")

        manager.add("chrome", chrome_requests)
        manager.add("firefox", firefox_requests)

    async def parse(self, response: Response):
        for link in response.css("a.product::attr(href)").getall():
            yield response.follow(link, callback=self.parse_product)

        next_page = response.css("a.next::attr(href)").get()
        if next_page:
            yield response.follow(next_page, sid="firefox")

    async def parse_product(self, response: Response):
        yield {
            "name": response.css("h1::text").get(""),
            "price": response.css(".price::text").get(""),
        }
```

你也可以分离关注点，为特定请求保留带有 cookie/状态的 session 等……

## Session 参数

传递给 `Request` 的额外关键字参数（或通过 `response.follow(**kwargs)`）会转发给 session 的 fetch 方法。这样你可以在不修改 session 配置的情况下自定义单个请求：

```python
async def parse(self, response: Response):
    # Pass extra headers for this specific request
    yield Request(
        "https://api.example.com/data",
        headers={"Authorization": "Bearer token123"},
        callback=self.parse_api,
    )

    # Use a different HTTP method
    yield Request(
        "https://example.com/submit",
        method="POST",
        data={"field": "value"},
        sid="firefox",
        callback=self.parse_result,
    )
```

!!! warning

    通常，使用 `FetcherSession`、`Fetcher` 或 `AsyncFetcher` 时，你会通过 `.get()`、`.post()` 等方法指定 HTTP 方法。但在 spider 中使用 `FetcherSession` 时不能这样做。默认情况下，请求是 _HTTP GET_ 请求；若要使用其他 HTTP 方法，必须传入 `method` 参数，如上例所示。这样做的原因是为了在所有 session 类型之间统一 `Request` 接口。

对于浏览器 session（`AsyncDynamicSession`、`AsyncStealthySession`），你可以传递浏览器专用参数，如 `wait_selector`、`page_action` 或 `extra_headers`：

```python
async def parse(self, response: Response):
    # Use Cloudflare solver with the `AsyncStealthySession` we configured above
    yield Request(
        "https://nopecha.com/demo/cloudflare",
        sid="stealth",
        callback=self.parse_result,
        solve_cloudflare=True,
        block_webrtc=True,
        hide_canvas=True,
        google_search=True,
    )

    yield response.follow(
        "/dynamic-page",
        sid="browser",
        callback=self.parse_dynamic,
        wait_selector="div.loaded",
        network_idle=True,
    )
```

!!! warning

    原始请求传递的 session 参数（**kwargs）会被 `response.follow()` 继承。新的 kwargs 优先于继承的 kwargs。

```python
from scrapling.spiders import Spider, Response
from scrapling.fetchers import FetcherSession

class ProductSpider(Spider):
    name = "products"
    start_urls = ["https://shop.example.com/products"]

    def configure_sessions(self, manager):
        manager.add("http", FetcherSession(impersonate='chrome'))

    async def parse(self, response: Response):
        # I don't want the follow request to impersonate a desktop Chrome like the previous request, but a mobile one
        # so I override it like this
        for link in response.css("a.product::attr(href)").getall():
            yield response.follow(link, impersonate="chrome131_android", callback=self.parse_product)

        next_page = response.css("a.next::attr(href)").get()
        if next_page:
            yield Request(next_page)

    async def parse_product(self, response: Response):
        yield {
            "name": response.css("h1::text").get(""),
            "price": response.css(".price::text").get(""),
        }
```
!!! info

    无需多说，spider 关闭时，manager 会自动检查是否有 session 仍在运行，并在关闭 spider 之前关闭它们。
