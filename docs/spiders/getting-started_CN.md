# 入门指南

## 简介

!!! success "前置条件"

    1. 你已完成或阅读过 [Fetcher 基础](../fetching/choosing_CN.md) 页面，了解不同 fetcher 类型及其适用场景。
    2. 你已完成或阅读过 [主要类](../parsing/main_classes_CN.md) 页面，了解 [Selector](../parsing/main_classes_CN.md#selector) 和 [Response](../fetching/choosing_CN.md#response-object) 类。
    3. 你已阅读 [架构](architecture_CN.md) 页面，对 spider 系统的工作原理有宏观了解。

spider 系统让你用寥寥几行代码就能构建并发、多页面的爬虫。如果你用过 Scrapy，其中的模式会感觉很熟悉。如果没有，本指南会带你掌握入门所需的一切。

## 你的第一个 Spider

spider 是一个类，用于定义如何爬取网站并提取数据。下面是最简单的 spider 示例：

```python
from scrapling.spiders import Spider, Response

class QuotesSpider(Spider):
    name = "quotes"
    start_urls = ["https://quotes.toscrape.com"]

    async def parse(self, response: Response):
        for quote in response.css("div.quote"):
            yield {
                "text": quote.css("span.text::text").get(""),
                "author": quote.css("small.author::text").get(""),
            }
```

每个 spider 都需要三样东西：

1. **`name`** - spider 的唯一标识符。
2. **`start_urls`** - 爬取起始 URL 列表。
3. **`parse()`** - 处理每个响应并产出结果的异步生成器方法。

`parse()` 方法是核心所在。你可以使用与 Scrapling 的 [Selector](../parsing/main_classes_CN.md#selector)/[Response](../fetching/choosing_CN.md#response-object) 相同的选择方法，并通过 `yield` 字典输出爬取到的条目。

## 运行 Spider

要运行 spider，创建实例并调用 `start()`：

```python
result = QuotesSpider().start()
```

`start()` 方法在内部处理所有异步机制，因此无需关心事件循环。spider 运行期间，所有事件都会记录到终端，爬取结束后你会得到非常详细的统计信息。

这些统计信息在返回的 `CrawlResult` 对象中，它提供你所需的一切：

```python
result = QuotesSpider().start()

# Access scraped items
for item in result.items:
    print(item["text"], "-", item["author"])

# Check statistics
print(f"Scraped {result.stats.items_scraped} items")
print(f"Made {result.stats.requests_count} requests")
print(f"Took {result.stats.elapsed_seconds:.1f} seconds")

# Did the crawl finish or was it paused?
print(f"Completed: {result.completed}")
```

## 跟随链接

大多数爬取需要跨多个页面跟随链接。使用 `response.follow()` 创建后续请求：

```python
from scrapling.spiders import Spider, Response

class QuotesSpider(Spider):
    name = "quotes"
    start_urls = ["https://quotes.toscrape.com"]

    async def parse(self, response: Response):
        # Extract items from the current page
        for quote in response.css("div.quote"):
            yield {
                "text": quote.css("span.text::text").get(""),
                "author": quote.css("small.author::text").get(""),
            }

        # Follow the "next page" link
        next_page = response.css("li.next a::attr(href)").get()
        if next_page:
            yield response.follow(next_page, callback=self.parse)
```

`response.follow()` 会自动将相对 URL 与当前页面 URL 拼接。默认情况下，它还会将当前页面设为 `Referer` 请求头。

你可以将后续请求指向不同的回调方法，以处理不同类型的页面：

```python
async def parse(self, response: Response):
    for link in response.css("a.product-link::attr(href)").getall():
        yield response.follow(link, callback=self.parse_product)

async def parse_product(self, response: Response):
    yield {
        "name": response.css("h1::text").get(""),
        "price": response.css(".price::text").get(""),
    }
```

!!! note

    所有回调方法都必须是异步生成器（使用 `async def` 和 `yield`）。

## 导出数据

`result.items` 中返回的 `ItemList` 内置了导出方法：

```python
result = QuotesSpider().start()

# Export as JSON
result.items.to_json("quotes.json")

# Export as JSON with pretty-printing
result.items.to_json("quotes.json", indent=True)

# Export as JSON Lines (one JSON object per line)
result.items.to_jsonl("quotes.jsonl")
```

两种方法都会在父目录不存在时自动创建。

## 过滤域名

使用 `allowed_domains` 将 spider 限制在特定域名内。这可以防止它意外跟随指向外部网站的链接：

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    allowed_domains = {"example.com"}

    async def parse(self, response: Response):
        for link in response.css("a::attr(href)").getall():
            # Links to other domains are silently dropped
            yield response.follow(link, callback=self.parse)
```

子域名会自动匹配，因此设置 `allowed_domains = {"example.com"}` 也会允许 `sub.example.com`、`blog.example.com` 等。

当请求被过滤掉时，会计入 `stats.offsite_requests_count`，以便你了解有多少请求被丢弃。

## 遵守 robots.txt

设置 `robots_txt_obey = True` 可使 spider 在爬取任何域名之前遵守 robots.txt 规则：

```python
class PoliteSpider(Spider):
    name = "polite"
    start_urls = ["https://example.com"]
    robots_txt_obey = True

    async def parse(self, response: Response):
        for link in response.css("a::attr(href)").getall():
            yield response.follow(link, callback=self.parse)
```

启用后，spider 将：

1. **在爬取开始前**（并发地）为 `start_urls` 中的所有域名**预取 robots.txt**。
2. **检查每个请求**是否符合该域名 robots.txt 的 `Disallow` 规则。不允许的请求会被静默丢弃，并计入 `stats.robots_disallowed_count`。
3. **遵守 `Crawl-delay` 和 `Request-rate` 指令**，取该指令与你配置的 `download_delay` 中的较大值。这意味着 robots.txt 的延迟永远不会降低你已配置的延迟，仅在需要时增加延迟。

robots.txt 文件使用 spider 的默认 session 获取，并在整个爬取过程中按域名缓存。爬取中途发现的域名（不在 `start_urls` 中）会在首次请求该域名时获取其 robots.txt。

**注意：** `robots_txt_obey` 默认关闭，以避免意外行为。如果启用，它不会影响你的并发设置（`concurrent_requests`、`concurrent_requests_per_domain`）——仅会调整请求之间的延迟。

## 下一步

掌握基础后，你可以继续探索：

- [请求与响应](requests-responses_CN.md) - 了解请求优先级、去重、元数据等。
- [Session](sessions_CN.md) - 在单个 spider 中使用多种 fetcher 类型（HTTP、浏览器、stealth）。
- [代理管理与拦截处理](proxy-blocking_CN.md) - 在请求间轮换代理，以及如何在 spider 中处理拦截。
- [高级功能](advanced_CN.md) - 并发控制、暂停/恢复、流式处理、生命周期钩子和日志。
