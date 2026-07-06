# 入门

## 第一个 Spider

Spider 是一个类，定义如何爬取网站并提取数据。下面是最简单的 spider：

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

每个 spider 需要三样东西：

1. **`name`**：spider 的唯一标识。
2. **`start_urls`**：起始 URL 列表。
3. **`parse()`**：处理每个响应并 yield 结果的 async generator 方法。

`parse()` 处理每个响应。选择与 Scrapling [Selector](../parsing/main_classes_CN.md#selector)/[Response](../fetching/choosing_CN.md#response-object) 相同的方式，用 `yield` 输出字典作为抓取项。

## 运行 Spider

创建实例并调用 `start()`：

```python
result = QuotesSpider().start()
```

`start()` 在内部处理全部 async 机制，无需关心事件循环。运行期间，所有事件会记录到终端，爬取结束时会得到详细统计。

统计信息在返回的 `CrawlResult` 对象中：

```python
result = QuotesSpider().start()

# 访问抓取项
for item in result.items:
    print(item["text"], "-", item["author"])

# 查看统计
print(f"Scraped {len(result.items)} quotes")
print(f"Made {result.stats.requests_count} requests")
print(f"Took {result.stats.elapsed_seconds:.1f} seconds")

# 爬取是完成还是暂停？
print(f"Completed: {result.completed}")
```

## 跟随链接

多数爬取需要跨页跟随链接。使用 `response.follow()` 创建后续请求：

```python
from scrapling.spiders import Spider, Response

class QuotesSpider(Spider):
    name = "quotes"
    start_urls = ["https://quotes.toscrape.com"]

    async def parse(self, response: Response):
        # 从当前页提取项
        for quote in response.css("div.quote"):
            yield {
                "text": quote.css("span.text::text").get(""),
                "author": quote.css("small.author::text").get(""),
            }

        # 跟随「下一页」链接
        next_page = response.css("li.next a::attr(href)").get()
        if next_page:
            yield response.follow(next_page, callback=self.parse)
```

`response.follow()` 会自动将相对 URL 与当前页 URL 拼接，并默认将当前页设为 `Referer` 头。

可将后续请求指向不同 callback，处理不同页面类型：

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

**说明：** 所有 callback 方法必须是 async generator（使用 `async def` 和 `yield`）。

## 导出数据

`result.items` 中的 `ItemList` 内置导出方法：

```python
result = QuotesSpider().start()

# 导出为 JSON
result.items.to_json("quotes.json")

# 格式化 JSON 导出
result.items.to_json("quotes.json", indent=True)

# 导出为 JSON Lines（每行一个 JSON 对象）
result.items.to_jsonl("quotes.jsonl")
```

两种方法会在父目录不存在时自动创建。

## 域名过滤

使用 `allowed_domains` 将 spider 限制在指定域名，避免误跟外链：

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    allowed_domains = {"example.com"}

    async def parse(self, response: Response):
        for link in response.css("a::attr(href)").getall():
            # 其他域名的链接会被静默丢弃
            yield response.follow(link, callback=self.parse)
```

子域名会自动匹配，因此 `allowed_domains = {"example.com"}` 也允许 `sub.example.com`、`blog.example.com` 等。

被过滤的请求会计入 `stats.offsite_requests_count`，便于查看丢弃数量。

## robots.txt 合规

设置 `robots_txt_obey = True`，在爬取任何域名前先遵守 robots.txt：

```python
class PoliteSpider(Spider):
    name = "polite"
    start_urls = ["https://example.com"]
    robots_txt_obey = True

    async def parse(self, response: Response):
        for link in response.css("a::attr(href)").getall():
            yield response.follow(link, callback=self.parse)
```

启用后 spider 将：

1. **预抓取 robots.txt**：在爬取开始前并发获取 `start_urls` 中所有域名的 robots.txt。
2. **检查每个请求**：对照该域名 robots.txt 的 `Disallow` 规则；不允许的请求会被静默丢弃并计入 `stats.robots_disallowed_count`。
3. **遵守 `Crawl-delay` 与 `Request-rate`**：取指令与配置的 `download_delay` 中的较大值。即 robots.txt 延迟不会低于你的配置，仅在需要时增加延迟。

robots.txt 使用 spider 默认 session 抓取，并在整个爬取期间按域名缓存。爬取中途发现的域名（不在 `start_urls` 中）会在首次请求该域名时抓取 robots.txt。

**说明：** `robots_txt_obey` 默认关闭。它不影响并发设置——仅调整请求间隔。
