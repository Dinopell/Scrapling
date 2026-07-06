# 通用 Spider 模板

多数爬取属于两种模式之一：「跟随匹配某正则的链接」或「爬取站点 sitemap 中列出的全部 URL」。Scrapling 为两者提供模板，避免重复编写相同的 `parse()` 样板代码。

两种模板都基于 `LinkExtractor`，从 `Response` 中提取 URL（或通过 `matches()` 过滤单个 URL）。`SitemapSpider` 还会内部解析 sitemap.xml / sitemap_index.xml 正文（无论是否 gzip 压缩）。

可在任意普通 `Spider.parse()` 中直接使用 `LinkExtractor`；模板只是省去接线工作。

## CrawlSpider

`CrawlSpider` 根据声明式规则自动跟随链接。

```python
from scrapling.spiders import CrawlSpider, CrawlRule, LinkExtractor

class BlogCrawler(CrawlSpider):
    name = "blog"
    start_urls = ["https://example.com"]

    def rules(self):
        return [
            CrawlRule(LinkExtractor(allow=r"/posts/"), callback=self.parse_post),
            CrawlRule(LinkExtractor(allow=r"/page/\d+/")),  # 跟随分页，无 callback
        ]

    async def parse_post(self, response):
        yield {
            "title": response.css("h1::text").get(),
            "url": response.url,
        }

result = BlogCrawler().start()
```

`CrawlRule` 将 `LinkExtractor` 与可选的 `callback`（spider 上的绑定方法）、可选的 `priority` 覆盖，以及可选的 `process_request`（在 yield 前修改每个 `Request` 的绑定方法）配对。默认 `parse()` 对每个 response 运行全部规则，并为每个匹配 URL yield 一个 `Request`。

若规则无 callback，匹配 URL 会落到 spider 默认 `parse()`（若未覆盖则可能无 callback）。这对分页很方便：提取下一页链接以继续爬取，但无需单独 handler。

### 规则与自定义逻辑结合

覆盖 `parse()` 并调用 `super().parse(response)` 可同时获得规则行为与自定义 yield：

```python
class MySpider(CrawlSpider):
    def rules(self):
        return [CrawlRule(LinkExtractor(allow=r"/posts/"), callback=self.parse_post)]

    async def parse(self, response):
        yield {"page_url": response.url}
        async for req in super().parse(response):
            yield req
```

### 用 `process_request` 修改 Request

```python
def add_priority(self, request, response):
    request.priority = 10
    return request

def rules(self):
    return [CrawlRule(
        LinkExtractor(allow=r"/posts/"),
        callback=self.parse_post,
        process_request=self.add_priority,
    )]
```

## SitemapSpider

`SitemapSpider` 从 sitemap.xml URL 播种爬取。与 `CrawlSpider` 使用相同的 `rules()` API，心智模型一致。

```python
from scrapling.spiders import SitemapSpider, CrawlRule, LinkExtractor

class MySitemap(SitemapSpider):
    name = "sm"
    sitemap_urls = ["https://example.com/sitemap.xml"]

    def rules(self):
        return [
            CrawlRule(LinkExtractor(allow=r"/posts/"), callback=self.parse_post),
            CrawlRule(LinkExtractor(allow=r"/products/"), callback=self.parse_product),
        ]

    async def parse_post(self, response):
        yield {"title": response.css("h1::text").get()}

    async def parse_product(self, response):
        yield {"sku": response.css(".sku::text").get()}

result = MySitemap().start()
```

### URL 如何分发

对 sitemap 中每个 URL，`SitemapSpider` 按顺序检查每条规则的 `LinkExtractor.matches(url)`。第一条匹配规则胜出，并 yield 带该规则 callback 的 `Request`。若无规则匹配且 `rules()` 非空，URL 被丢弃（与 Scrapy 行为一致）。若 `rules()` 返回空列表，所有 URL 路由到 spider 的 `parse()`，默认会抛出 `NotImplementedError`——需覆盖以处理它们。

### Sitemap 索引

当 `SitemapSpider` 遇到 `<sitemapindex>`（sitemap 的 sitemap）时，会自动进入每个子 sitemap。要过滤进入哪些子 sitemap，将 `sitemap_follow` 设为 `LinkExtractor`：

```python
class MySitemap(SitemapSpider):
    name = "sm"
    sitemap_urls = ["https://example.com/sitemap.xml"]
    sitemap_follow = LinkExtractor(allow=r"/posts-sitemap-\d+\.xml")  # 仅文章 sitemap
```

### robots.txt 支持

将 `robots.txt` URL 直接放入 `sitemap_urls`，`SitemapSpider` 会检测并通过 `protego` 提取每个 `Sitemap:` 指令并跟随：

```python
class MySitemap(SitemapSpider):
    name = "sm"
    sitemap_urls = ["https://example.com/robots.txt"]  # 自动发现 Sitemap: 指令
```

###  alternate 语言 URL

设置 `sitemap_alternate_links = True`，也会通过规则分发 `<xhtml:link rel="alternate" hreflang="...">` URL。

## 直接使用 `LinkExtractor`

不必使用模板。`LinkExtractor` 可在任意普通 `Spider` 中使用：

```python
from scrapling.spiders import Spider, LinkExtractor

class CustomSpider(Spider):
    name = "custom"
    start_urls = ["https://example.com"]

    def __init__(self):
        super().__init__()
        self._links = LinkExtractor(allow=r"/posts/", deny_domains="ads.example.com")

    async def parse(self, response):
        for url in self._links.extract(response):
            yield response.follow(url, callback=self.parse_post)

    async def parse_post(self, response):
        yield {"title": response.css("h1::text").get()}
```

## LinkExtractor 参考

| 参数              | 默认值              | 说明                                                                 |
|-------------------|---------------------|----------------------------------------------------------------------|
| `allow`           | `()`                | 保留的 URL 模式。空表示「匹配全部」。字符串、编译后的 `Pattern` 或可迭代对象。 |
| `deny`            | `()`                | 丢弃的 URL 模式。始终覆盖 `allow`。                                  |
| `allow_domains`   | `()`                | 保留的主机名。子域名自动匹配（`example.com` 匹配 `api.example.com`）。 |
| `deny_domains`    | `()`                | 丢弃的主机名。                                                       |
| `restrict_css`    | `()`                | 将 DOM 提取限定在区域的 CSS 选择器。                                 |
| `restrict_xpath`  | `()`                | 将 DOM 提取限定在区域的 XPath 选择器。                               |
| `tags`            | `("a", "area")`     | 查找链接的元素标签。                                                 |
| `attrs`           | `("href",)`         | 从这些标签读取 URL 的属性。                                          |
| `canonicalize`    | `True`              | 排序查询参数并规范化路径。                                           |
| `strip`           | `True`              | 去除提取 URL 的空白。                                                |
| `keep_fragment`   | `False`             | 规范化时保留 `#fragment`。                                           |
| `deny_extensions` | `IGNORED_EXTENSIONS`| 丢弃的文件扩展名（pdf、zip、图片、视频等）。                         |
| `process`         | `None`              | 过滤前应用于每个提取 URL 的可选 callable。返回 falsy 则丢弃。        |

`LinkExtractor.extract(response)` 返回绝对、已过滤、去重后的 `list[str]` URL。

`LinkExtractor.matches(url)` 返回 `bool`——仅 URL 的过滤（allow/deny/domain/extension），供 `SitemapSpider` 在无 `Response` 时分发 URL。
