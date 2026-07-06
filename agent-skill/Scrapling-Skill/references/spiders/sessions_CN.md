# Spiders 会话

一个 spider 可同时使用多个 fetcher 会话。例如，对简单页面使用快速 HTTP 会话，对受保护页面使用隐身浏览器会话。

## 什么是会话？

会话是预配置好的 fetcher 实例，在整次爬取期间保持存活。spider 复用会话，而不是为每个请求新建连接或浏览器，从而更快、更省资源。

默认情况下，每个 spider 会创建一个 [FetcherSession](../fetching/static_CN.md)。你可以通过重写 `configure_sessions()` 添加更多会话或替换默认会话，但只能使用下表中各会话的异步版本：

| Session Type                                    | Use Case                                 |
|-------------------------------------------------|------------------------------------------|
| [FetcherSession](../fetching/static_CN.md)         | Fast HTTP requests, no JavaScript        |
| [AsyncDynamicSession](../fetching/dynamic_CN.md)   | Browser automation, JavaScript rendering |
| [AsyncStealthySession](../fetching/stealthy_CN.md) | Anti-bot bypass, Cloudflare, etc.        |


## 配置会话

在 spider 上重写 `configure_sessions()` 以设置会话。`manager` 参数是 `SessionManager` 实例——使用 `manager.add()` 注册会话：

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

| Argument     | Type      | Default    | Description                                  |
|--------------|-----------|------------|----------------------------------------------|
| `session_id` | `str`     | *required* | A name to reference this session in requests |
| `session`    | `Session` | *required* | The session instance                         |
| `default`    | `bool`    | `False`    | Make this the default session                |
| `lazy`       | `bool`    | `False`    | Start the session only when first used       |

**说明：**

1. 在所有请求中，若未指定使用哪个会话，则使用默认会话。默认会话由以下两种方式之一确定：
    1. 添加到 manager 的第一个会话会自动成为默认会话。
    2. 添加时设置 `default=True` 的会话。
2. 你传入的各会话实例不必事先由你启动；spider 会检查所有会话，对未启动的会话进行启动。
3. 若希望某会话仅在首次使用时才启动，可在将该会话加入 manager 时使用 `lazy` 参数。例如：仅在需要时启动浏览器，而不是在 spider 启动时。

## 多会话 Spider

下面是一个实用示例：对列表页使用快速 HTTP 会话，对带机器人防护的详情页使用隐身浏览器：

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
        manager.add("stealth", AsyncStealthySession(
            headless=True,
            network_idle=True,
        ))

    async def parse(self, response: Response):
        for link in response.css("a.product::attr(href)").getall():
            # Route product pages through the stealth session
            yield response.follow(link, sid="stealth", callback=self.parse_product)

        next_page = response.css("a.next::attr(href)").get()
        if next_page:
            yield response.follow(next_page)

    async def parse_product(self, response: Response):
        yield {
            "name": response.css("h1::text").get(""),
            "price": response.css(".price::text").get(""),
        }
```

关键在于 `sid` 参数——它告诉 spider 每个请求应使用哪个会话。调用 `response.follow()` 时若不传 `sid`，会继承原请求的会话 ID。

会话也可以是同一类的不同实例，配置各不相同：

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

## 会话参数

传给 `Request` 的额外关键字参数（或通过 `response.follow(**kwargs)` 传递）会转发到会话的 fetch 方法。这样可在不修改会话配置的情况下定制单个请求：

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

**警告：** 在 spider 中使用 `FetcherSession` 时，不能直接调用 `.get()` 和 `.post()` 方法。默认请求为 HTTP GET；要使用其他 HTTP 方法，请像上例那样传入 `method` 参数。这样可在所有会话类型间统一 `Request` 接口。

对于浏览器会话（`AsyncDynamicSession`、`AsyncStealthySession`），可传入浏览器专用参数，如 `wait_selector`、`page_action` 或 `extra_headers`：

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

**警告：** 原请求传入的会话参数（**kwargs）会被 `response.follow()` 继承。新传入的 kwargs 优先于继承的 kwargs。

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
**注意：** spider 关闭时，manager 会自动检查是否仍有会话在运行，并在关闭 spider 前关闭它们。
