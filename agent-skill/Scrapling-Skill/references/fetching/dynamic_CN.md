# 抓取动态网站

`DynamicFetcher`（原 `PlayWrightFetcher`）提供灵活的浏览器自动化，支持多种配置选项，并内置隐匿增强。

如后文所述，要自动化页面，你需要了解 [Playwright 的 Page API](https://playwright.dev/python/docs/api/class-page)。

## 基本用法
导入此 Fetcher 的主要方式只有一种，与其他 fetcher 相同。

```python
from scrapling.fetchers import DynamicFetcher
```
了解如何配置解析选项，请参见[此处](choosing_CN.md#parser-configuration-in-all-fetchers)

**说明：** `fetch` 方法的异步版本为 `async_fetch`。

此 fetcher 提供三种主要运行模式，可按需组合。

分别为：

### 1. 原生 Playwright
```python
DynamicFetcher.fetch('https://example.com')
```
这样使用会打开 Chromium 浏览器并加载页面。底层有速度优化和部分自动隐匿，但除非启用额外选项，否则就是 plain PlayWright API，没有额外技巧或功能。

### 2. 真实 Chrome
```python
DynamicFetcher.fetch('https://example.com', real_chrome=True)
```
若已安装 Google Chrome，可使用此选项。与第一种相同，但会使用设备上已安装的 Google Chrome，而非 Chromium。请求看起来更真实，更难被检测，效果更好。

若未安装 Google Chrome 却想使用此选项，可在终端运行以下命令由库安装，而无需手动安装：
```commandline
playwright install chrome
```

### 3. CDP 连接
```python
DynamicFetcher.fetch('https://example.com', cdp_url='ws://localhost:9222')
```
不在本地启动浏览器（Chromium/Google Chrome）时，可通过 [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) 连接远程浏览器。


**说明：**
* 此处曾有 `stealth` 选项，自 0.3.13 起已移至 `StealthyFetcher` 类，并增加更多功能，见下一页说明。
* 这样对新用户更清晰、更易维护，并带来其他好处，详见 [StealthyFetcher 页面](stealthy_CN.md)。

## 完整参数列表
`DynamicFetcher` 及其会话类的全部参数：

|      Argument       | Description                                                                                                                                                                                                                         | Optional |
|:-------------------:|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:--------:|
|         url         | 目标 URL                                                                                                                                                                                                                          |    ❌     |
|      headless       | 传入 `True` 以无头/隐藏模式运行浏览器（**默认**），或 `False` 为有界面/可见模式。                                                                                                                                |    ✔️    |
|  disable_resources  | 丢弃不必要的资源请求以提速。丢弃类型包括 `font`、`image`、`media`、`beacon`、`object`、`imageset`、`texttrack`、`websocket`、`csp_report` 和 `stylesheet`。                         |    ✔️    |
|       cookies       | 为下一次请求设置 Cookie。                                                                                                                                                                                                   |    ✔️    |
|      useragent      | 传入要使用的 useragent 字符串。**否则 fetcher 会生成并使用同浏览器同版本的真实 Useragent。**                                                                                              |    ✔️    |
|    network_idle     | 等待页面至少 500 ms 内无网络连接。                                                                                                                                                       |    ✔️    |
|      load_dom       | 默认启用，等待页面上所有 JavaScript 完全加载并执行（等待 `domcontentloaded` 状态）。                                                                                                           |    ✔️    |
|       timeout       | 页面中所有操作与等待使用的超时（毫秒）。默认 30,000 ms（30 秒）。                                                                                                                |    ✔️    |
|        wait         | 一切完成后、关闭页面并返回 `Response` 对象前 fetcher 等待的时间（毫秒）。                                                                                                |    ✔️    |
|     page_action     | 用于自动化。传入接收 `page` 对象的函数，在导航后运行并完成所需自动化。                                                                                                       |    ✔️    |
|     page_setup      | 接收 `page` 对象的函数，在导航前运行。用于注册必须在页面加载前设置的事件监听器或路由。                                                                            |    ✔️    |
|    wait_selector    | 等待指定 CSS 选择器达到特定状态。                                                                                                                                                                         |    ✔️    |
|     init_script     | 在本会话所有页面创建时执行的 JavaScript 文件的绝对路径。                                                                                                                                |    ✔️    |
| wait_selector_state | Scrapling 会等待 `wait_selector` 所给选择器达到指定状态。_默认状态为 `attached`。_                                                                                                 |    ✔️    |
|    google_search    | 默认启用，Scrapling 会设置 Google referer 头。                                                                                               |    ✔️    |
|    extra_headers    | 要添加到请求的额外请求头字典。_若与 `google_search` 同时使用，此处设置的 referer 优先级低于 `google_search` 设置的 referer。_                                                                   |    ✔️    |
|        proxy        | 请求使用的代理。可为字符串，或仅含 'server'、'username'、'password' 键的字典。                                                                                                     |    ✔️    |
|     real_chrome     | 若设备已安装 Chrome，启用后 Fetcher 会启动并使用你的浏览器实例。                                                                                                |    ✔️    |
|       locale        | 指定用户区域，例如 `en-GB`、`de-DE` 等。会影响 `navigator.language`、`Accept-Language` 请求头以及数字、日期格式规则。默认为系统默认区域。 |    ✔️    |
|     timezone_id     | 更改浏览器时区。默认为系统时区。                                                                                                                                                               |    ✔️    |
|       cdp_url       | 不启动新浏览器实例，而是连接此 CDP URL，通过 CDP 控制真实浏览器。                                                                                                                          |    ✔️    |
|    user_data_dir    | User Data Directory 路径，存储 Cookie、local storage 等浏览器会话数据。默认创建临时目录。**仅会话可用**                                                       |    ✔️    |
|     extra_flags     | 浏览器启动时传入的额外浏览器标志列表。                                                                                                                                                                |    ✔️    |
|   additional_args   | 作为额外设置传给 Playwright context 的附加参数，优先级高于 Scrapling 设置。                                                                                          |    ✔️    |
|   selector_config   | 创建最终 `Selector`/`Response` 类时使用的自定义解析参数字典。                                                                                                                            |    ✔️    |
|   blocked_domains   | 要拦截请求的域名集合。子域名也会匹配（例如 `"example.com"` 也会拦截 `"sub.example.com"`）。                                                                                                     |    ✔️    |
|     block_ads       | 拦截约 3,500 个已知广告/追踪域名。可与 `blocked_domains` 组合使用。                                                                                                                                         |    ✔️    |
|   dns_over_https    | 通过 Cloudflare 的 DNS-over-HTTPS 路由 DNS 查询，在使用代理时防止 DNS 泄漏。                                                                                                                                      |    ✔️    |
|    proxy_rotator    | 用于自动轮换代理的 `ProxyRotator` 实例。不能与 `proxy` 同时使用。                                                                                                                                            |    ✔️    |
|       retries       | 失败请求的重试次数。默认为 3。                                                                                                                                                                        |    ✔️    |
|     retry_delay     | 重试之间的等待秒数。默认为 1。                                                                                                                                                                              |    ✔️    |
|     capture_xhr     | 传入正则 URL 模式字符串，在页面加载期间捕获匹配的 XHR/fetch 请求。捕获的响应可通过 `response.captured_xhr` 访问。默认为 `None`（禁用）。                                             |    ✔️    |
|   executable_path   | 自定义浏览器可执行文件的绝对路径，替代捆绑的 Chromium。适用于非标准安装或自定义浏览器构建。                                                                                |    ✔️    |

在会话类中，这些参数可全局设置。仍可通过传入部分可在浏览器标签页级别配置的参数，为单次请求单独配置，例如：`google_search`、`timeout`、`wait`、`page_action`、`page_setup`、`extra_headers`、`disable_resources`、`wait_selector`、`wait_selector_state`、`network_idle`、`load_dom`、`blocked_domains`、`proxy` 和 `selector_config`。

**说明：**
1. 测试中 `disable_resources` 对部分网站约提速 25%，并有助于节省代理流量，但需谨慎，可能导致部分网站永远无法完成加载。
2. `google_search` 默认对所有请求启用，将 referer 设为 `https://www.google.com/`。与 `extra_headers` 同时使用时，其 referer 优先级高于后者。
3. 自 0.3.13 起，`stealth` 选项已移除，改用 `StealthyFetcher` 类；`hide_canvas` 也已移至该类。`disable_webgl` 已移至 `StealthyFetcher` 并重命名为 `allow_webgl`。
4. 若未设置 user agent 且启用了无头模式，fetcher 会为同浏览器版本生成真实 user agent 并使用。若未设置 user agent 且未启用无头模式，fetcher 会使用浏览器默认 user agent，与最新版标准浏览器相同。


## 示例

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

**警告：** 默认情况下，所有基于浏览器的 fetcher 与会话使用带标签页池的持久浏览器 context。但由于浏览器无法按标签页设置代理，使用 `ProxyRotator` 时 fetcher 会为每个代理自动打开独立 context，每个 context 一个标签页。标签页任务完成后，标签页及其 context 都会关闭。

### 下载文件

```python
page = DynamicFetcher.fetch('https://raw.githubusercontent.com/D4Vinci/Scrapling/main/docs/assets/main_cover.png')

with open(file='main_cover.png', mode='wb') as f:
    f.write(page.body)
```

`Response` 对象的 `body` 属性始终返回 `bytes`。

### 导航前设置
若需在页面导航前注册事件监听器、路由或脚本，请使用 `page_setup`。该函数接收 `page` 对象，在调用 `page.goto()` 之前运行。

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

可与 `page_action` 组合使用——`page_setup` 在导航前运行，`page_action` 在导航后运行。

### 浏览器自动化
此处会用到你对 [Playwright 的 Page API](https://playwright.dev/python/docs/api/class-page) 的了解。传入的函数接收 Playwright API 的 page 对象，执行所需操作后 fetcher 继续。

该函数在等待 `network_idle`（若启用）之后、等待 `wait_selector` 参数之前执行，因此不仅可用于自动化。可按需修改页面。

下例使用页面的 [mouse 事件](https://playwright.dev/python/docs/api/class-mouse) 用滚轮滚动页面，然后移动鼠标。
```python
from playwright.sync_api import Page

def scroll_page(page: Page):
    page.mouse.wheel(10, 0)
    page.mouse.move(100, 400)
    page.mouse.up()

page = DynamicFetcher.fetch('https://example.com', page_action=scroll_page)
```
若使用异步 fetch 版本，函数也必须是 async。
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
    'https://example.com',
    wait_selector='h1',
    wait_selector_state='visible'
)
```
这是 fetcher 在返回响应前进行的最后一次等待（若启用）。向 `wait_selector` 传入 CSS 选择器，fetcher 会等待 `wait_selector_state` 中指定的状态满足。若未传入状态，默认为 `attached`，即等待元素出现在 DOM 中。

之后，若启用了 `load_dom`（默认），fetcher 会再次检查所有 JavaScript 是否已加载并执行（`domcontentloaded` 状态），否则继续等待。若启用了 `network_idle`，fetcher 会再次等待 `network_idle` 满足，如上所述。

fetcher 可等待的状态包括（[来源](https://playwright.dev/python/docs/api/class-page#page-wait-for-selector)）：

- `attached`：等待元素出现在 DOM 中。
- `detached`：等待元素不在 DOM 中。
- `visible`：等待元素具有非空 bounding box 且无 `visibility:hidden`。注意无内容或 `display:none` 的元素 bounding box 为空，不算可见。
- `hidden`：等待元素从 DOM 分离，或 bounding box 为空，或 `visibility:hidden`。与 `'visible'` 相反。

### 捕获 XHR/Fetch 请求

许多 SPA 通过后台 API（XHR/fetch）加载数据。可在会话级别向 `capture_xhr` 传入正则 URL 模式来捕获这些请求：

```python
from scrapling.fetchers import DynamicSession

with DynamicSession(capture_xhr=r"https://api\.example\.com/.*", headless=True) as session:
    page = session.fetch('https://example.com')

    # Access captured XHR responses
    for xhr in page.captured_xhr:
        print(xhr.url, xhr.status)
        print(xhr.body)  # Raw response body as bytes
```

`captured_xhr` 中每一项都是完整的 `Response` 对象，具有相同属性（`.url`、`.status`、`.headers`、`.body` 等）。当 `capture_xhr` 未设置或为 `None` 时，`captured_xhr` 为空列表。

### 部分隐匿功能

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

## 会话管理

若要在相同配置下保持浏览器打开并发起多次请求，请使用 `DynamicSession`/`AsyncDynamicSession` 类。这些类可接受 `fetch` 函数的全部参数，便于为整个会话指定配置。

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

你可能注意到 `max_pages` 参数。这是新参数，使 fetcher 创建**浏览器标签页轮换池**。不再用单个标签页处理所有请求，而是限制同时可打开的最大页面数。每次请求时，库会关闭已完成任务的标签页，并检查当前标签数是否低于允许的最大页面/标签数，然后：

1. 若在允许范围内，fetcher 会为你创建新标签页，之后流程照常。
2. 否则，会以亚秒级间隔检查是否允许创建新标签页，持续 60 秒后抛出 `TimeoutError`。这可能发生在所抓取网站无响应时。

该逻辑允许在同一浏览器中同时抓取多个 URL，节省大量资源，更重要的是速度极快 :)

在 0.3 与 0.3.1 版本中，池会复用已完成标签页以节省资源/时间。该逻辑被证明有缺陷，因为几乎无法防止页面/标签被上一次请求的配置污染。

### 会话优势

- **浏览器复用**：复用同一浏览器实例，后续请求快得多。
- **Cookie 持久化**：像普通浏览器一样自动处理 Cookie 与会话状态。
- **一致指纹**：所有请求使用相同浏览器指纹。
- **内存效率**：相比每次 fetch 都启动新浏览器，资源占用更优。

## 何时使用

在以下情况使用 DynamicFetcher：

- 需要浏览器自动化
- 需要多种浏览器选项
- 使用真实 Chrome 浏览器
- 需要自定义浏览器配置
- 需要少量隐匿选项

若需要更多隐匿与控制且配置不多，请查看 [StealthyFetcher](stealthy_CN.md)。
