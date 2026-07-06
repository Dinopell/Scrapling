# Fetcher 基础

## 简介
Fetcher 是一类以单行方式发起请求或抓取页面、功能丰富并返回 [Response](#response-object) 对象的类。各 Fetcher 都有独立的 session 类以保持 session 运行（例如浏览器 fetcher 会在你完成所有请求前保持浏览器打开）。

Fetcher 并非其他库的薄封装。它们以这些库为引擎发起请求/抓取，同时增加底层引擎不具备的能力，并仍充分 leverage 与优化底层引擎以用于网页抓取。

## Fetcher 概览

Scrapling 提供三种 Fetcher 类及其 session 类；每种针对特定场景设计。

下表可快速对照选型。


| 特性               | Fetcher                                           | DynamicFetcher                                                                    | StealthyFetcher                                                                            |
|--------------------|---------------------------------------------------|-----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| 相对速度           | 🐇🐇🐇🐇🐇                                        | 🐇🐇🐇                                                                            | 🐇🐇🐇                                                                                     |
| 隐身能力           | ⭐⭐                                                | ⭐⭐⭐                                                                               | ⭐⭐⭐⭐⭐                                                                                      |
| 反机器人选项       | ⭐⭐                                                | ⭐⭐⭐                                                                               | ⭐⭐⭐⭐⭐                                                                                      |
| JavaScript 加载    | ❌                                                 | ✅                                                                                 | ✅                                                                                          |
| 内存占用           | ⭐                                                 | ⭐⭐⭐                                                                               | ⭐⭐⭐                                                                                        |
| 最佳用途           | 纯 HTTP 即可的基础抓取                            | - 动态加载网站 <br/>- 轻量自动化<br/>- 中低强度防护                               | - 动态加载网站 <br/>- 轻量自动化 <br/>- 中高强度防护                                       |
| 浏览器             | ❌                                                 | Chromium 与 Google Chrome                                                         | Chromium 与 Google Chrome                                                                 |
| 使用的浏览器 API   | ❌                                                 | PlayWright                                                                        | PlayWright                                                                                 |
| 配置复杂度         | 简单                                              | 简单                                                                              | 简单                                                                                       |

## 所有 Fetcher 的解析器配置
所有 Fetcher 使用相同导入方式，后续页面会说明：
```python
from scrapling.fetchers import Fetcher, AsyncFetcher, StealthyFetcher, DynamicFetcher
```
无需初始化即可直接使用，将使用默认解析器设置：
```python
page = StealthyFetcher.fetch('https://example.com') 
```
若要在返回响应前配置将使用的解析器（[Selector 类](parsing/main_classes_CN.md#selector)），可先执行：
```python
from scrapling.fetchers import Fetcher
Fetcher.configure(adaptive=True, keep_comments=False, keep_cdata=False)  # 其余参数同理
```
或
```python
from scrapling.fetchers import Fetcher
Fetcher.adaptive=True
Fetcher.keep_comments=False
Fetcher.keep_cdata=False  # 其余参数同理
```
然后照常编写代码。

可用配置参数：`adaptive`、`adaptive_domain`、`huge_tree`、`keep_comments`、`keep_cdata`、`storage`、`storage_args`，与 [Selector](parsing/main_classes_CN.md#selector) 类相同。随时可运行 `<fetcher_class>.display_config()` 查看当前配置。

**信息：** `adaptive` 默认关闭；需显式启用才能使用该功能。

### 按请求设置解析器配置
上述设置解析器配置的逻辑会全局应用于该类的所有请求/抓取，旨在简化使用。

若每个请求/抓取需要不同配置，可向请求方法（`fetch`/`get`/`post`/...）的 `selector_config` 参数传入字典。

## Response 对象
`Response` 对象与 [Selector](parsing/main_classes_CN.md#selector) 类相同，但额外包含响应详情，如响应头、状态码、cookies 等：
```python
from scrapling.fetchers import Fetcher
page = Fetcher.get('https://example.com')

page.status          # HTTP 状态码
page.reason          # 状态消息
page.cookies         # 响应 cookies（字典）
page.headers         # 响应头
page.request_headers # 请求头
page.history         # 重定向历史（如有）
page.body            # 原始响应体（bytes）
page.encoding        # 响应编码
page.meta            # 响应元数据字典（如使用的 proxy），主要用于 spider 系统
page.captured_xhr    # 捕获的 XHR/fetch 响应列表（浏览器 session 启用 capture_xhr 时）
```
所有 Fetcher 均返回 `Response` 对象。

**说明：** 与 [Selector](parsing/main_classes_CN.md#selector) 类不同，自 v0.4 起 `Response` 类的 body 始终为 bytes。
