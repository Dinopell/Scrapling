# 抓取器基础

## 简介
抓取器（Fetcher）是一类能以单行代码轻松发起请求或抓取页面、具备多种功能并返回 [Response](#response-object) 对象的类。自 v0.3 起，所有抓取器都有独立的会话类以保持会话运行；例如使用浏览器的抓取器会在你通过它完成所有请求之前保持浏览器打开，而不是每次打开多个浏览器。因此取决于你的使用场景。

引入此特性是因为 v0.2 之前 Scrapling 仅是解析引擎。目标是逐步成为网络爬虫的一站式解决方案。

> 抓取器并非基于其他库的简单封装。它们仅以这些库为引擎来请求/抓取页面。进一步说明：所有抓取器都具有底层引擎没有的特性，同时充分利用并针对网络爬虫优化这些引擎。

## 抓取器概览

Scrapling 提供三种不同的抓取器类及其会话类；每种针对特定使用场景设计。

下表对比它们，可快速参考。


| 特性            | Fetcher                                           | DynamicFetcher                                                                    | StealthyFetcher                                                                            |
|--------------------|---------------------------------------------------|-----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| 相对速度     | 🐇🐇🐇🐇🐇                                        | 🐇🐇🐇                                                                            | 🐇🐇🐇                                                                                     |
| 隐匿性            | ⭐⭐                                                | ⭐⭐⭐                                                                               | ⭐⭐⭐⭐⭐                                                                                      |
| 反机器人选项   | ⭐⭐                                                | ⭐⭐⭐                                                                               | ⭐⭐⭐⭐⭐                                                                                      |
| JavaScript 加载 | ❌                                                 | ✅                                                                                 | ✅                                                                                          |
| 内存占用       | ⭐                                                 | ⭐⭐⭐                                                                               | ⭐⭐⭐                                                                                        |
| 最佳用途      | 纯 HTTP 请求即可完成的简单爬取 | - 动态加载网站 <br/>- 轻度自动化<br/>- 中低强度防护 | - 动态加载网站 <br/>- 轻度自动化 <br/>- 中高强度复杂防护 |
| 浏览器         | ❌                                                 | Chromium 和 Google Chrome                                                        | Chromium 和 Google Chrome                                                                 |
| 使用的浏览器 API   | ❌                                                 | PlayWright                                                                        | PlayWright                                                                                 |
| 配置复杂度   | 简单                                            | 简单                                                                            | 简单                                                                                     |

后续页面将逐一详细说明。

## 所有抓取器中的解析器配置
所有抓取器共享相同的导入方式，后续页面会展示
```python
from scrapling.fetchers import Fetcher, AsyncFetcher, StealthyFetcher, DynamicFetcher
```
然后可直接使用而无需初始化，将使用默认解析器设置：
```python
page = StealthyFetcher.fetch('https://example.com') 
```
若要在返回响应前配置将用于响应的解析器（[Selector 类](../parsing/main_classes_CN.md#selector)），先执行：
```python
from scrapling.fetchers import Fetcher
Fetcher.configure(adaptive=True, keep_comments=False, keep_cdata=False)  # and the rest
```
或
```python
from scrapling.fetchers import Fetcher
Fetcher.adaptive=True
Fetcher.keep_comments=False
Fetcher.keep_cdata=False  # and the rest
```
然后照常编写代码。

可用配置参数：`adaptive`、`adaptive_domain`、`huge_tree`、`keep_comments`、`keep_cdata`、`storage` 和 `storage_args`，与传给 [Selector](../parsing/main_classes_CN.md#selector) 类的相同。随时可运行 `<fetcher_class>.display_config()` 查看当前配置。

!!! info

    `adaptive` 参数默认关闭；必须启用才能使用该特性。

### 按请求设置解析器配置
如上所述，解析器配置逻辑会全局应用于通过该类的所有请求/抓取，旨在简化使用。

若每个请求/抓取需要不同配置，可向请求方法（`fetch`/`get`/`post`/...）的 `selector_config` 参数传入字典。

## Response 对象
`Response` 对象与 [Selector](../parsing/main_classes_CN.md#selector) 类相同，但包含响应的额外信息，如响应头、状态码、Cookie 等，如下所示：
```python
from scrapling.fetchers import Fetcher
page = Fetcher.get('https://example.com')

page.status          # HTTP status code
page.reason          # Status message
page.cookies         # Response cookies as a dictionary
page.headers         # Response headers
page.request_headers # Request headers
page.history         # Response history of redirections, if any
page.body            # Raw response body as bytes
page.encoding        # Response encoding
page.meta            # Response metadata dictionary (e.g., proxy used). Mainly helpful with the spiders system.
page.captured_xhr    # List of captured XHR/fetch responses (when capture_xhr is enabled on a browser session)
```
所有抓取器都返回 `Response` 对象。

!!! note

    与 [Selector](../parsing/main_classes_CN.md#selector) 类不同，自 v0.4 起 `Response` 类的 body 始终为 bytes。
