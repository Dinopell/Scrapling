# 抓取动态网站

此处讨论 `DynamicFetcher` 类（原 `PlayWrightFetcher`）。该类提供灵活的浏览器自动化、多种配置选项及少量底层隐匿优化。

如下文所述，要进行页面自动化，需要了解 [Playwright 的 Page API](https://playwright.dev/python/docs/api/class-page)。

!!! success "前置条件"

    1. 你已完成或阅读过[抓取器基础](../fetching/choosing_CN.md)页面，了解 [Response 对象](../fetching/choosing_CN.md#response-object) 是什么以及该用哪种抓取器。
    2. 你已完成或阅读过[查询元素](../parsing/selection_CN.md)页面，了解如何从 [Selector](../parsing/main_classes_CN.md#selector)/[Response](../fetching/choosing_CN.md#response-object) 对象中查找/提取元素。
    3. 你已完成或阅读过[核心类](../parsing/main_classes_CN.md)页面，了解 [Response](../fetching/choosing_CN.md#response-object) 类从 [Selector](../parsing/main_classes_CN.md#selector) 类继承了哪些属性/方法。

## 基本用法
导入此 Fetcher 的主要方式与其他抓取器相同。

```python
from scrapling.fetchers import DynamicFetcher
```
解析选项配置见[此处](choosing_CN.md#parser-configuration-in-all-fetchers)

下面将用示例逐一说明大部分参数。若要跳转到所有参数的速查表，[点击此处](#full-list-of-arguments)

!!! abstract

    `fetch` 方法的异步版本当然是 `async_fetch`。


此抓取器目前提供三种主要运行模式，可按需组合。

即：

### 1. 原生 Playwright
```python
DynamicFetcher.fetch('https://example.com')
```
这样使用会打开 Chromium 浏览器并加载页面。有速度优化，底层会自动做一些隐匿处理，但除非启用额外特性，否则就是普通 Playwright API，无额外技巧。

### 2. 真实 Chrome
```python
DynamicFetcher.fetch('https://example.com', real_chrome=True)
```
若设备上安装了 Google Chrome，使用此选项。与第一种相同，但使用你安装的 Google Chrome 而非 Chromium，使请求更真实、更难被检测，效果更好。

若未安装 Google Chrome 却想用此选项，可在终端运行以下命令由库安装，无需手动安装：
```commandline
playwright install chrome
```

### 3. CDP 连接
```python
DynamicFetcher.fetch('https://example.com', cdp_url='ws://localhost:9222')
```
不在本地启动浏览器（Chromium/Google Chrome），而是通过 [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) 连接远程浏览器。


!!! note "注意："

    * 此处曾有 `stealth` 选项，但已移至 `StealthyFetcher` 类，如下一页所述，自 0.3.13 起有更多特性。<br/>
    * 这样对新用户更清晰、更易维护，并有其他好处，见 [StealthyFetcher 页面](../fetching/stealthy_CN.md)。

## 完整参数列表
Scrapling 为此抓取器及其会话类提供许多选项。为尽量简化，此处列出选项，并对大部分参数给出使用示例。

|      参数       | 说明                                                                                                                                                                                                                         | 可选 |
|:-------------------:|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:--------:|
|         url         | 目标 URL                                                                                                                                                                                                                          |    ❌     |
|      headless       | 传 `True` 以无头/隐藏模式运行浏览器（**默认**），或 `False` 为有界面/可见模式。                                                                                                                                |    ✔️    |
|  disable_resources  | 丢弃不必要资源请求以提速。丢弃类型：`font`、`image`、`media`、`beacon`、`object`、`imageset`、`texttrack`、`websocket`、`csp_report` 和 `stylesheet`。                         |    ✔️    |
|       cookies       | 为下次请求设置 Cookie。                                                                                                                                                                                                   |    ✔️    |
|      useragent      | 传入要使用的 user agent 字符串。**否则抓取器会生成并使用同浏览器同版本的真实 User-Agent。**                                                                                              |    ✔️    |
|    network_idle     | 等待页面直到至少 500 ms 内无网络连接。                                                                                                                                                       |    ✔️    |
|      load_dom       | 默认启用，等待页面上所有 JavaScript 完全加载并执行（等待 `domcontentloaded` 状态）。                                                                                                           |    ✔️    |
|       timeout       | 页面所有操作和等待使用的超时（毫秒）。默认 30,000 ms（30 秒）。                                                                                                                |    ✔️    |
|        wait         | 一切完成后、关闭页面并返回 `Response` 对象前的等待时间（毫秒）。                                                                                                |    ✔️    |
|     page_action     | 用于自动化。传入接受 `page` 对象的函数，在导航后运行并完成必要自动化。                                                                                                       |    ✔️    |
|     page_setup      | 接受 `page` 对象的函数，在导航前运行。用于注册必须在页面加载前设置的事件监听器或路由。                                                                            |    ✔️    |
|    wait_selector    | 等待特定 CSS 选择器达到特定状态。                                                                                                                                                                         |    ✔️    |
|     init_script     | 要在本会话所有页面创建时执行的 JavaScript 文件的绝对路径。                                                                                                                                |    ✔️    |
| wait_selector_state | Scrapling 将等待 `wait_selector` 所给选择器达到指定状态。_默认状态为 `attached`。_                                                                                                 |    ✔️    |
|    google_search    | 默认启用，Scrapling 会设置 Google referer 头。                                                                                                                                                                     |    ✔️    |
|    extra_headers    | 要添加到请求的额外头字典。_若与 `google_search` 同用，此处设置的 referer 优先级低于 `google_search` 设置的 referer。_                                                                                |    ✔️    |
|        proxy        | 请求使用的代理。可为字符串或仅含 'server'、'username'、'password' 键的字典。                                                                                                     |    ✔️    |
|     real_chrome     | 若设备上安装了 Chrome，启用后 Fetcher 会启动并使用你的浏览器实例。                                                                                                |    ✔️    |
|       locale        | 指定用户区域，如 `en-GB`、`de-DE` 等。影响 `navigator.language`、`Accept-Language` 请求头以及数字和日期格式。默认为系统默认区域。 |    ✔️    |
|     timezone_id     | 更改浏览器时区。默认为系统时区。                                                                                                                                                               |    ✔️    |
|       cdp_url       | 不启动新浏览器实例，而是连接此 CDP URL 通过 CDP 控制真实浏览器。                                                                                                                          |    ✔️    |
|    user_data_dir    | 用户数据目录路径，存储 Cookie、local storage 等浏览器会话数据。默认创建临时目录。**仅适用于会话**                                                       |    ✔️    |
|     extra_flags     | 启动时传给浏览器的额外浏览器标志列表。                                                                                                                                                                |    ✔️    |
|   additional_args   | 作为额外设置传给 Playwright context 的额外参数，优先级高于 Scrapling 设置。                                                                                          |    ✔️    |
|   selector_config   | 创建最终 `Selector`/`Response` 类时使用的自定义解析参数字典。                                                                                                                            |    ✔️    |
|   blocked_domains   | 要阻止请求的域名集合。子域名也会匹配（例如 `"example.com"` 也会阻止 `"sub.example.com"`）。                                                                                                     |    ✔️    |
|      block_ads      | 阻止对约 3,500 个已知广告/追踪域名的请求。可与 `blocked_domains` 组合。                                                                                                                                         |    ✔️    |
|   dns_over_https    | 通过 Cloudflare 的 DNS-over-HTTPS 路由 DNS 查询，使用代理时防止 DNS 泄漏。                                                                                                                                      |    ✔️    |
|    proxy_rotator    | 用于自动代理轮换的 `ProxyRotator` 实例。不能与 `proxy` 组合。                                                                                                                                            |    ✔️    |
|       retries       | 失败请求的重试次数。默认 3。                                                                                                                                                                        |    ✔️    |
|     retry_delay     | 重试间隔秒数。默认 1。                                                                                                                                                                              |    ✔️    |
|     capture_xhr     | 传入正则 URL 模式字符串，在页面加载期间捕获匹配的 XHR/fetch 请求。捕获的响应通过 `response.captured_xhr` 访问。默认为 `None`（禁用）。                                            |    ✔️    |
|   executable_path   | 自定义浏览器可执行文件的绝对路径，替代捆绑的 Chromium。适用于非标准安装或自定义浏览器构建。                                                                                |    ✔️    |

在会话类中，这些参数可全局设置。仍可通过传入此处可在浏览器标签页级别配置的参数，为每个请求单独配置：`google_search`、`timeout`、`wait`、`page_action`、`page_setup`、`extra_headers`、`disable_resources`、`wait_selector`、`wait_selector_state`、`network_idle`、`load_dom`、`blocked_domains`、`proxy` 和 `selector_config`。

!!! note "注意："

    1. 我的测试中，`disable_resources` 对某些网站使请求快约 25%，也有助于节省代理用量，但需谨慎，可能导致某些网站永远无法加载完成。
    2. `google_search` 参数默认对所有请求启用，将 referer 设为 `https://www.google.com/`。若与 `extra_headers` 同用，其 referer 优先级高于 `extra_headers` 中设置的 referer。
    3. 自 0.3.13 起，`stealth` 选项已移除，改用 `StealthyFetcher` 类；`hide_canvas` 也已移至该类。`disable_webgl` 参数移至 `StealthyFetcher` 并重命名为 `allow_webgl`。
    4. 若未设置 user agent 且启用了无头模式，抓取器会生成同浏览器版本的真实 user agent 并使用。若未设置 user agent 且未启用无头模式，抓取器使用浏览器默认 user agent，与最新版标准浏览器相同。


## 会话管理

若要在相同配置下保持浏览器打开以发起多个请求，使用 `DynamicSession`/`AsyncDynamicSession` 类。这些类可接受 `fetch` 函数的所有参数，便于为整个会话指定配置。

```python
from scrapling.fetchers import DynamicSession

# Create a session with default configuration
with DynamicSession(
    headless=True,
    disable_resources=True,
    real_chrome=True
) as session:
    # Make multiple requests with the same browser instance
    page1 = session.fetch('https://example1.com')
    page2 = session.fetch('https://example2.com')
    page3 = session.fetch('https://dynamic-site.com')
    
    # All requests reuse the same tab on the same browser instance
```

### 异步会话用法

```python
import asyncio
from scrapling.fetchers import AsyncDynamicSession

async def scrape_multiple_sites():
    async with AsyncDynamicSession(
        network_idle=True,
        timeout=30000,
        max_pages=3
    ) as session:
        # Make async requests with shared browser configuration
        pages = await asyncio.gather(
            session.fetch('https://spa-app1.com'),
            session.fetch('https://spa-app2.com'),
            session.fetch('https://dynamic-content.com')
        )
        return pages
```

你可能注意到 `max_pages` 参数。这是新参数，使抓取器创建**浏览器标签页轮换池**。不是对所有请求使用单一标签页，而是设置可同时显示的最大页面数。每次请求时，库会关闭所有已完成任务的标签页，并检查当前标签数是否低于允许的最大页面/标签数，然后：

1. 若在允许范围内，抓取器会创建新标签页，之后一切照常。
2. 否则，每秒检查是否允许创建新标签页，持续 60 秒后抛出 `TimeoutError`。这可能发生在抓取的网站无响应时。

此逻辑允许在同一浏览器中同时抓取多个 URL，节省大量资源，更重要的是非常快 :)

在 0.3 和 0.3.1 版本中，池会复用已完成的标签页以进一步节省资源/时间。该逻辑有缺陷，几乎无法防止标签页被上一次请求的配置污染。

### 会话优势

- **浏览器复用**：复用同一浏览器实例，后续请求快得多。
- **Cookie 持久化**：像普通浏览器一样自动处理 Cookie 和会话状态。
- **一致指纹**：所有请求使用相同浏览器指纹。
- **内存效率**：相比每次抓取启动新浏览器，资源占用更优。


## 示例
示例更易理解，下面来看。

### 资源控制

```python
# Disable unnecessary resources
page = DynamicFetcher.fetch('https://example.com', disable_resources=True)  # Blocks fonts, images, media, etc.
```

### 域名拦截

```python
# Block requests to specific domains (and their subdomains)
page = DynamicFetcher.fetch('https://example.com', blocked_domains={"ads.example.com", "tracker.net"})
```

### 网络控制

```python
# Wait for network idle (Consider fetch to be finished when there are no network connections for at least 500 ms)
page = DynamicFetcher.fetch('https://example.com', network_idle=True)

# Custom timeout (in milliseconds)
page = DynamicFetcher.fetch('https://example.com', timeout=30000)  # 30 seconds

# Proxy support (It can also be a dictionary with only the keys 'server', 'username', and 'password'.)
page = DynamicFetcher.fetch('https://example.com', proxy='http://username:password@host:port')
```

### 下载文件

```python
page = DynamicFetcher.fetch('https://raw.githubusercontent.com/D4Vinci/Scrapling/main/docs/assets/main_cover.png')

with open(file='main_cover.png', mode='wb') as f:
    f.write(page.body)
```

`Response` 对象的 `body` 属性始终返回 `bytes`。

### 导航前设置
若需在页面导航前设置事件监听器、路由或脚本，使用 `page_setup`。该函数接收 `page` 对象，在调用 `page.goto()` 之前运行。

```python
from playwright.sync_api import Page

def capture_websockets(page: Page):
    page.on("websocket", lambda ws: print(f"WebSocket opened: {ws.url}"))

page = DynamicFetcher.fetch('https://example.com', page_setup=capture_websockets)
```
异步版本：
```python
from playwright.async_api import Page

async def capture_websockets(page: Page):
    page.on("websocket", lambda ws: print(f"WebSocket opened: {ws.url}"))

page = await DynamicFetcher.async_fetch('https://example.com', page_setup=capture_websockets)
```

可与 `page_action` 组合——`page_setup` 在导航前运行，`page_action` 在导航后运行。

### 浏览器自动化
此处需要 [Playwright 的 Page API](https://playwright.dev/python/docs/api/class-page) 知识。传入的函数接收 Playwright API 的 page 对象，执行所需操作后抓取器继续。

该函数在等待 `network_idle`（若启用）之后、等待 `wait_selector` 参数之前立即执行，因此不仅可用于自动化。可按需修改页面。

下例使用页面的[鼠标事件](https://playwright.dev/python/docs/api/class-mouse)滚轮滚动页面，然后移动鼠标。
```python
from playwright.sync_api import Page

def scroll_page(page: Page):
    page.mouse.wheel(10, 0)
    page.mouse.move(100, 400)
    page.mouse.up()

page = DynamicFetcher.fetch('https://example.com', page_action=scroll_page)
```
若使用异步 fetch，函数也必须是 async。
```python
from playwright.async_api import Page

async def scroll_page(page: Page):
   await page.mouse.wheel(10, 0)
   await page.mouse.move(100, 400)
   await page.mouse.up()

page = await DynamicFetcher.async_fetch('https://example.com', page_action=scroll_page)
```

### 等待条件

```python
# Wait for the selector
page = DynamicFetcher.fetch(
    'https://quotes.toscrape.com/js-delayed/',
    wait_selector='.quote',
    wait_selector_state='visible'
)
```
这是抓取器在返回响应前进行的最后一次等待（若启用）。向 `wait_selector` 传入 CSS 选择器，抓取器会等待 `wait_selector_state` 中传入的状态满足。若未传状态，默认为 `attached`，即等待元素出现在 DOM 中。

之后，若启用 `load_dom`（默认），抓取器会再次检查所有 JavaScript 文件是否已加载并执行（`domcontentloaded` 状态）或继续等待。若启用了 `network_idle`，抓取器会再次等待 `network_idle` 满足，如上所述。

抓取器可等待的状态包括（[来源](https://playwright.dev/python/docs/api/class-page#page-wait-for-selector)）：

- `attached`：等待元素出现在 DOM 中。
- `detached`：等待元素从 DOM 中移除。
- `visible`：等待元素具有非空边界框且无 `visibility:hidden`。注意无内容或 `display:none` 的元素边界框为空，不算可见。
- `hidden`：等待元素从 DOM 分离，或边界框为空，或 `visibility:hidden`。与 `'visible'` 相反。

### 捕获 XHR/Fetch 请求

许多 SPA 通过后台 API 调用（XHR/fetch）加载数据。可在会话级别向 `capture_xhr` 传入正则 URL 模式来捕获这些请求：

```python
from scrapling.fetchers import DynamicSession

with DynamicSession(capture_xhr=r"https://api\.example\.com/.*", headless=True) as session:
    page = session.fetch('https://example.com')

    # Access captured XHR responses
    for xhr in page.captured_xhr:
        print(xhr.url, xhr.status)
        print(xhr.body)  # Raw response body as bytes
```

`captured_xhr` 中每项都是完整的 `Response` 对象，具有相同属性（`.url`、`.status`、`.headers`、`.body` 等）。未设置 `capture_xhr` 或为 `None` 时，`captured_xhr` 为空列表。

### 部分隐匿特性

```python
page = DynamicFetcher.fetch(
    'https://example.com',
    google_search=True,
    useragent='Mozilla/5.0...',  # Custom user agent
    locale='en-US',  # Set browser locale
)
```

### 综合示例
```python
from scrapling.fetchers import DynamicFetcher

def scrape_dynamic_content():
    # Use Playwright for JavaScript content
    page = DynamicFetcher.fetch(
        'https://example.com/dynamic',
        network_idle=True,
        wait_selector='.content'
    )
    
    # Extract dynamic content
    content = page.css('.content')
    
    return {
        'title': content.css('h1::text').get(),
        'items': [
            item.text for item in content.css('.item')
        ]
    }
```

### 代理轮换

```python
from scrapling.fetchers import DynamicSession, ProxyRotator

# Set up proxy rotation
rotator = ProxyRotator([
    "http://proxy1:8080",
    "http://proxy2:8080",
    "http://proxy3:8080",
])

# Use with session - rotates proxy automatically with each request
with DynamicSession(proxy_rotator=rotator, headless=True) as session:
    page1 = session.fetch('https://example1.com')
    page2 = session.fetch('https://example2.com')

    # Override rotator for a specific request
    page3 = session.fetch('https://example3.com', proxy='http://specific-proxy:8080')
```

!!! warning

    请记住，默认情况下所有基于浏览器的抓取器和会话使用带标签页池的持久浏览器上下文。但浏览器无法为每个标签页单独设置代理，使用 `ProxyRotator` 时，抓取器会为每个代理自动打开独立上下文，每个上下文一个标签页。标签页任务完成后，标签页及其上下文都会关闭。

## 何时使用

在以下情况使用 DynamicFetcher：

- 需要浏览器自动化
- 需要多种浏览器选项
- 使用真实 Chrome 浏览器
- 需要自定义浏览器配置
- 需要少量隐匿选项

若需要更多隐匿和控制且配置不多，请查看 [StealthyFetcher](stealthy_CN.md)。
