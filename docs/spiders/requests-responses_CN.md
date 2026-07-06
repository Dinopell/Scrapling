# 请求与响应

!!! success "前置条件"

    1. 你已阅读 [入门指南](getting-started_CN.md) 页面，了解如何创建和运行基础 spider。

本页详细介绍 `Request` 对象：如何构造请求、在回调间传递数据、控制优先级和去重，以及如何使用 `response.follow()` 跟随链接。

## Request 对象

`Request` 表示待获取的 URL。你可以直接创建请求，或通过 `response.follow()` 创建：

```python
from scrapling.spiders import Request

# Direct construction
request = Request(
    "https://example.com/page",
    callback=self.parse_page,
    priority=5,
)

# Via response.follow (preferred in callbacks)
request = response.follow("/page", callback=self.parse_page)
```

以下是可传递给 `Request` 的所有参数：

| 参数      | 类型       | 默认值    | 说明                                                                                           |
|---------------|------------|------------|-------------------------------------------------------------------------------------------------------|
| `url`         | `str`      | *必填* | 要获取的 URL                                                                                      |
| `sid`         | `str`      | `""`       | Session ID - 将请求路由到指定 session（参见 [Session](sessions_CN.md)）                   |
| `callback`    | `callable` | `None`     | 处理响应的异步生成器方法。默认为 `parse()`                                 |
| `priority`    | `int`      | `0`        | 数值越大越先处理                                                                     |
| `dont_filter` | `bool`     | `False`    | 若为 `True`，跳过去重（允许重复请求）                                              |
| `meta`        | `dict`     | `{}`       | 任意元数据，会传递到响应                                                     |
| `**kwargs`    |            |            | 传递给 session fetch 方法的额外关键字参数（如 `headers`、`method`、`data`） |

任何额外的关键字参数都会直接转发给底层 session。例如，发起 POST 请求：

```python
yield Request(
    "https://example.com/api",
    method="POST",
    data={"key": "value"},
    callback=self.parse_result,
)
```

## Response.follow()

`response.follow()` 是在回调中创建后续请求的推荐方式。相比直接构造 `Request` 对象，它有以下优势：

- **相对 URL** 会自动根据当前页面 URL 解析
- **Referer 请求头** 默认设为当前页面 URL
- **Session 关键字参数** 从原始请求继承（headers、代理设置等）
- **回调、session ID 和优先级** 若未指定，则从原始请求继承

```python
async def parse(self, response: Response):
    # Minimal - inherits callback, sid, priority from current request
    yield response.follow("/next-page")

    # Override specific fields
    yield response.follow(
        "/product/123",
        callback=self.parse_product,
        priority=10,
    )

    # Pass additional metadata to
    yield response.follow(
        "/details",
        callback=self.parse_details,
        meta={"category": "electronics"},
    )
```

| 参数           | 类型       | 默认值    | 说明                                                |
|--------------------|------------|------------|------------------------------------------------------------|
| `url`              | `str`      | *必填* | 要跟随的 URL（绝对或相对）                       |
| `sid`              | `str`      | `""`       | Session ID（为空时从原始请求继承）       |
| `callback`         | `callable` | `None`     | 回调方法（为 `None` 时从原始请求继承） |
| `priority`         | `int`      | `None`     | 优先级（为 `None` 时从原始请求继承）        |
| `dont_filter`      | `bool`     | `False`    | 跳过去重                                         |
| `meta`             | `dict`     | `None`     | 元数据（与现有响应 meta 合并）              |
| **`referer_flow`** | `bool`     | `True`     | 将当前 URL 设为 Referer 请求头                          |
| `**kwargs`         |            |            | 与原始请求的 session 关键字参数合并              |

### 禁用 Referer 传递

默认情况下，`response.follow()` 会将 `Referer` 请求头设为当前页面 URL。要禁用此行为：

```python
yield response.follow("/page", referer_flow=False)
```

## 回调

回调是 spider 上处理响应的异步生成器方法。它们必须 `yield` 以下三种类型之一：

- **`dict`** - 爬取条目，加入结果
- **`Request`** - 后续请求，加入队列
- **`None`** - 静默忽略

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]

    async def parse(self, response: Response):
        # Yield items (dicts)
        yield {"url": response.url, "title": response.css("title::text").get("")}

        # Yield follow-up requests
        for link in response.css("a::attr(href)").getall():
            yield response.follow(link, callback=self.parse_page)

    async def parse_page(self, response: Response):
        yield {"content": response.css("article::text").get("")}
```

!!! tip "注意："

    所有回调方法必须是 `async def` 并使用 `yield`（而非 `return`）。即使回调只产出条目、没有后续请求，也必须是异步生成器。

## 请求优先级

优先级数值较大的请求会先处理。这在某些页面需要比其他页面优先处理时很有用：

```python
async def parse(self, response: Response):
    # High priority - process product pages first
    for link in response.css("a.product::attr(href)").getall():
        yield response.follow(link, callback=self.parse_product, priority=10)

    # Low priority - pagination links processed after products
    next_page = response.css("a.next::attr(href)").get()
    if next_page:
        yield response.follow(next_page, callback=self.parse, priority=0)
```

使用 `response.follow()` 时，除非指定新值，否则优先级从原始请求继承。

## 去重

spider 会根据 URL、HTTP 方法、请求体和 session ID 计算的指纹自动去重。如果两个请求产生相同指纹，第二个会被静默丢弃。

要允许重复请求（例如登录后重新访问页面），设置 `dont_filter=True`：

```python
yield Request("https://example.com/dashboard", dont_filter=True, callback=self.parse_dashboard)

# Or with response.follow
yield response.follow("/dashboard", dont_filter=True, callback=self.parse_dashboard)
```

你可以使用 spider 上的类属性微调指纹的组成：

| 属性            | 默认值 | 效果                                                                                                          |
|----------------------|---------|-----------------------------------------------------------------------------------------------------------------|
| `fp_include_kwargs`  | `False` | 将额外请求关键字参数（传给 session fetch 的参数，如 headers 等）纳入指纹 |
| `fp_keep_fragments`  | `False` | 计算指纹时保留 URL 片段（`#section`）                                                     |
| `fp_include_headers` | `False` | 将请求头纳入指纹                                                                      |

例如，若需要将 `https://example.com/page#section1` 和 `https://example.com/page#section2` 视为不同 URL：

```python
class MySpider(Spider):
    name = "my_spider"
    fp_keep_fragments = True
    # ...
```

## 请求 Meta

`meta` 字典允许你在回调之间传递任意数据。当你需要将一个页面的上下文用于处理另一个页面时很有用：

```python
async def parse(self, response: Response):
    for product in response.css("div.product"):
        category = product.css("span.category::text").get("")
        link = product.css("a::attr(href)").get()
        if link:
            yield response.follow(
                link,
                callback=self.parse_product,
                meta={"category": category},
            )

async def parse_product(self, response: Response):
    yield {
        "name": response.css("h1::text").get(""),
        "price": response.css(".price::text").get(""),
        # Access meta from the request
        "category": response.meta.get("category", ""),
    }
```

使用 `response.follow()` 时，当前响应的 meta 会与你提供的新 meta 合并（新值优先）。

spider 系统还会自动存储一些元数据。例如，启用代理轮换后，请求使用的代理可通过 `response.meta["proxy"]` 获取。
