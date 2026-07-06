# 请求与响应

本页详述 `Request` 对象：如何构造请求、在 callback 间传递数据、控制优先级与去重，以及使用 `response.follow()` 跟随链接。

## Request 对象

`Request` 表示待抓取的 URL。可直接构造，或通过 `response.follow()` 创建：

```python
from scrapling.spiders import Request

# 直接构造
request = Request(
    "https://example.com/page",
    callback=self.parse_page,
    priority=5,
)

# 通过 response.follow（callback 中推荐）
request = response.follow("/page", callback=self.parse_page)
```

`Request` 可接受的全部参数：

| 参数          | 类型       | 默认值     | 说明                                                                                           |
|---------------|------------|------------|------------------------------------------------------------------------------------------------|
| `url`         | `str`      | *必填*     | 要抓取的 URL                                                                                   |
| `sid`         | `str`      | `""`       | Session ID——将请求路由到指定 session（见 [Sessions](sessions_CN.md)）                           |
| `callback`    | `callable` | `None`     | 处理响应的 async generator 方法。默认 `parse()`                                                 |
| `priority`    | `int`      | `0`        | 数值越大越先处理                                                                               |
| `dont_filter` | `bool`     | `False`    | 为 `True` 时跳过去重（允许重复请求）                                                           |
| `meta`        | `dict`     | `{}`       | 任意元数据，会传递到 response                                                                  |
| `**kwargs`    |            |            | 额外关键字参数转发给 session 的 fetch 方法（如 `headers`、`method`、`data`）                    |

额外关键字参数会直接转发到底层 session。例如 POST 请求：

```python
yield Request(
    "https://example.com/api",
    method="POST",
    data={"key": "value"},
    callback=self.parse_result,
)
```

## Response.follow()

`response.follow()` 是 callback 内创建后续请求的推荐方式，相比直接构造 `Request` 有以下优势：

- **相对 URL** 会自动基于当前页 URL 解析
- **Referer 头** 默认设为当前页 URL
- **Session kwargs** 继承自原请求（headers、proxy 设置等）
- **Callback、session ID、priority** 未指定时继承自原请求

```python
async def parse(self, response: Response):
    # 最简——继承 callback、sid、priority
    yield response.follow("/next-page")

    # 覆盖特定字段
    yield response.follow(
        "/product/123",
        callback=self.parse_product,
        priority=10,
    )

    # 传递额外元数据
    yield response.follow(
        "/details",
        callback=self.parse_details,
        meta={"category": "electronics"},
    )
```

| 参数               | 类型       | 默认值     | 说明                                                |
|--------------------|------------|------------|-----------------------------------------------------|
| `url`              | `str`      | *必填*     | 要跟随的 URL（绝对或相对）                          |
| `sid`              | `str`      | `""`       | Session ID（为空则继承原请求）                      |
| `callback`         | `callable` | `None`     | Callback 方法（`None` 则继承原请求）                |
| `priority`         | `int`      | `None`     | 优先级（`None` 则继承原请求）                       |
| `dont_filter`      | `bool`     | `False`    | 跳过去重                                            |
| `meta`             | `dict`     | `None`     | 元数据（与现有 response meta 合并）                 |
| **`referer_flow`** | `bool`     | `True`     | 将当前 URL 设为 Referer 头                          |
| `**kwargs`         |            |            | 与原请求的 session kwargs 合并                      |

### 禁用 Referer 传递

默认 `response.follow()` 会将 `Referer` 设为当前页 URL。要禁用：

```python
yield response.follow("/page", referer_flow=False)
```

## Callback

Callback 是 spider 上处理响应的 async generator 方法。必须 `yield` 以下三种类型之一：

- **`dict`**：抓取项，加入结果
- **`Request`**：后续请求，加入队列
- **`None`**：静默忽略

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]

    async def parse(self, response: Response):
        # yield 项（dict）
        yield {"url": response.url, "title": response.css("title::text").get("")}

        # yield 后续请求
        for link in response.css("a::attr(href)").getall():
            yield response.follow(link, callback=self.parse_page)

    async def parse_page(self, response: Response):
        yield {"content": response.css("article::text").get("")}
```

**说明：** 所有 callback 必须是 `async def` 并使用 `yield`（不能用 `return`）。即使 callback 只 yield 项、无后续请求，也必须是 async generator。

## 请求优先级

priority 值越大越先处理。适用于某些页面需优先于其他页面：

```python
async def parse(self, response: Response):
    # 高优先级——先处理商品页
    for link in response.css("a.product::attr(href)").getall():
        yield response.follow(link, callback=self.parse_product, priority=10)

    # 低优先级——分页链接在商品之后
    next_page = response.css("a.next::attr(href)").get()
    if next_page:
        yield response.follow(next_page, callback=self.parse, priority=0)
```

使用 `response.follow()` 时，除非指定新值，否则继承原请求的 priority。

## 去重

Spider 根据 URL、HTTP 方法、请求体、session ID 计算的指纹自动去重。两个请求指纹相同则后者被静默丢弃。

要允许重复请求（如登录后重新访问页面），设置 `dont_filter=True`：

```python
yield Request("https://example.com/dashboard", dont_filter=True, callback=self.parse_dashboard)

# 或使用 response.follow
yield response.follow("/dashboard", dont_filter=True, callback=self.parse_dashboard)
```

可通过 spider 类属性微调指纹内容：

| 属性                 | 默认值  | 效果                                                                                                          |
|----------------------|---------|---------------------------------------------------------------------------------------------------------------|
| `fp_include_kwargs`  | `False` | 将额外请求 kwargs（传给 session fetch 的参数，如 headers 等）纳入指纹                                         |
| `fp_keep_fragments`  | `False` | 计算指纹时保留 URL fragment（`#section`）                                                                     |
| `fp_include_headers` | `False` | 将请求头纳入指纹                                                                                              |

例如需将 `https://example.com/page#section1` 与 `https://example.com/page#section2` 视为不同 URL：

```python
class MySpider(Spider):
    name = "my_spider"
    fp_keep_fragments = True
    # ...
```

## Request Meta

`meta` 字典用于在 callback 间传递任意数据，适用于需要上一页上下文处理下一页：

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
        # 从请求访问 meta
        "category": response.meta.get("category", ""),
    }
```

使用 `response.follow()` 时，当前 response 的 meta 会与新 meta 合并（新值优先）。

Spider 系统也会自动写入部分元数据。例如启用 proxy 轮换时，使用的 proxy 可通过 `response.meta["proxy"]` 访问。
