# 通用 Spider 模板

大多数爬取属于以下两种模式之一：「跟随匹配某模式的链接」或「爬取站点 sitemap 中列出的每个 URL」。Scrapling 为这两种情况提供了模板，这样你就不必每次都手写相同的 `parse()` 样板代码。

所有模板都基于 `LinkExtractor`，它从 `Response` 中提取 URL（或通过 `matches()` 过滤单个 URL）。`SitemapSpider` 还会在内部解析 sitemap.xml / sitemap_index.xml 正文（无论是否 gzip 压缩）。

你可以在任何普通 `Spider.parse()` 中直接使用 `LinkExtractor`。模板只是帮你省去接线工作。

## CrawlSpider

`CrawlSpider` 根据声明式规则自动跟随链接。

```python
from scrapling.spiders import CrawlSpider, CrawlRule, LinkExtractor

class QuotesSpider(CrawlSpider):
    name = "blog"
    start_urls = ["https://quotes.toscrape.com/"]

    def rules(self):
        return [
            CrawlRule(LinkExtractor(allow=r"/author/"), callback=self.parse_author),
            CrawlRule(LinkExtractor(allow=r"/page/\d+/")),  # follow pagination, no callback
        ]

    async def parse_author(self, response):
        yield {
            '.author-title': response.css('.author-title::text').get(),
            "birthday": response.css('.author-born-date::text').get(),
            "url": response.url,
        }

result = QuotesSpider().start()
```

`CrawlRule` 将 `LinkExtractor` 与可选的 `callback`（spider 上的绑定方法）、分发的 `Request` 的可选 `priority` 覆盖，以及可选的 `process_request`（在产出前修改每个 `Request` 的绑定方法）配对。默认的 `parse()` 对每个响应运行所有规则，并为每个匹配的 URL 产出一个 `Request`。

若规则没有 callback，匹配的 URL 会回退到 spider 的默认 `parse()`。这对分页很方便：提取下一页链接以继续爬取，无需单独的处理程序。

### 结合规则与自定义逻辑

重写 `parse()` 并调用 `super().parse(response)` 以获得规则行为以及你自己的产出：

```python
class MySpider(CrawlSpider):
    def rules(self):
        return [CrawlRule(LinkExtractor(allow=r"/posts/"), callback=self.parse_post)]

    async def parse(self, response):
        yield {"page_url": response.url}
        async for req in super().parse(response):
            yield req
```

### 使用 `process_request` 修改请求

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

`SitemapSpider` 从 sitemap.xml URL 播种爬取。它使用与 `CrawlSpider` 相同的 `rules()` API，因此心智模型一致。

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

对于 sitemap 中的每个 URL，`SitemapSpider` 按顺序检查每个规则的 `LinkExtractor.matches(url)`。第一个匹配的规则获胜，并产出带有该规则 callback 的 `Request`。若没有规则匹配且 `rules()` 非空，则丢弃该 URL。若 `rules()` 返回空列表，每个 URL 都会路由到 spider 的 `parse()` 方法；若未重写，默认会抛出 `NotImplementedError`。

### Sitemap 索引

当 `SitemapSpider` 遇到 `<sitemapindex>`（sitemap 的 sitemap）时，会自动进入每个子 sitemap。要过滤要进入哪些子 sitemap，将 `sitemap_follow` 设为 `LinkExtractor`：

```python
class MySitemap(SitemapSpider):
    name = "sm"
    sitemap_urls = ["https://example.com/sitemap.xml"]
    sitemap_follow = LinkExtractor(allow=r"/posts-sitemap-\d+\.xml")  # only post sitemaps
```

### Robots.txt 支持

将 `robots.txt` URL 直接放入 `sitemap_urls`，`SitemapSpider` 会检测它，提取其中列出的每个 sitemap 并跟随：

```python
class MySitemap(SitemapSpider):
    name = "sm"
    sitemap_urls = ["https://example.com/robots.txt"]
```

### 备用语言 URL

设置 `sitemap_alternate_links = True`，也会通过你的规则分发 `<xhtml:link rel="alternate" hreflang="...">` URL。

## 直接使用 `LinkExtractor`

你不必使用模板。`LinkExtractor` 可在任何普通 `Spider` 中使用：

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

| 参数          | 默认值              | 说明                                                                                       |
|-------------------|----------------------|---------------------------------------------------------------------------------------------------|
| `allow`           | `()`                 | 要保留的 URL 模式。空表示「匹配全部」。字符串、编译后的 `Pattern` 或二者的可迭代对象。 |
| `deny`            | `()`                 | 要丢弃的 URL 模式。始终覆盖 `allow`。                                                   |
| `allow_domains`   | `()`                 | 要保留的主机名。子域名自动匹配（`example.com` 匹配 `api.example.com`）。      |
| `deny_domains`    | `()`                 | 要丢弃的主机名。                                                                                |
| `restrict_css`    | `()`                 | 将 DOM 提取限定在某个区域的 CSS 选择器。                                              |
| `restrict_xpath`  | `()`                 | 将 DOM 提取限定在某个区域的 XPath 选择器。                                            |
| `tags`            | `("a", "area")`      | 查找链接的元素标签。                                                                |
| `attrs`           | `("href",)`          | 从这些标签读取 URL 的属性。                                                       |
| `canonicalize`    | `True`               | 对查询参数排序并规范化路径。                                                         |
| `strip`           | `True`               | 去除提取 URL 的空白字符。                                                             |
| `keep_fragment`   | `False`              | 规范化时保留 `#fragment`。                                                     |
| `deny_extensions` | `IGNORED_EXTENSIONS` | 要丢弃的文件扩展名（pdf、zip、图片、视频等）。                                          |
| `process`         | `None`               | 过滤前应用于每个提取 URL 的可选可调用对象。返回假值则丢弃。   |

`LinkExtractor.extract(response)` 返回绝对、已过滤、去重后的 URL `list[str]`。

`LinkExtractor.matches(url)` 返回 `bool`——仅基于 URL 的过滤（allow/deny/domain/extension），供 `SitemapSpider` 在没有 `Response` 的情况下分发 URL。
