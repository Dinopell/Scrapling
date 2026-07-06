# 抓取带强防护的动态网站

此处讨论 `StealthyFetcher` 类。该类与 [DynamicFetcher](dynamic_CN.md#introduction) 类非常相似，包括浏览器、自动化以及对 [Playwright API](https://playwright.dev/python/docs/intro) 的使用。主要区别在于此类提供高级反机器人绕过能力；大部分在底层自动处理，其余由你按需启用。

与 [DynamicFetcher](dynamic_CN.md#introduction) 一样，进行页面自动化需要了解 [Playwright 的 Page API](https://playwright.dev/python/docs/api/class-page)，如下文所述。

!!! success "前置条件"

    1. 你已完成或阅读过 [DynamicFetcher](dynamic_CN.md#introduction) 页面，因为此类在其基础上构建，为此不重复相同信息。
    2. 你已完成或阅读过[抓取器基础](../fetching/choosing_CN.md)页面，了解 [Response 对象](../fetching/choosing_CN.md#response-object) 是什么以及该用哪种抓取器。
    3. 你已完成或阅读过[查询元素](../parsing/selection_CN.md)页面，了解如何从 [Selector](../parsing/main_classes_CN.md#selector)/[Response](../fetching/choosing_CN.md#response-object) 对象中查找/提取元素。
    4. 你已完成或阅读过[核心类](../parsing/main_classes_CN.md)页面，了解 [Response](../fetching/choosing_CN.md#response-object) 类从 [Selector](../parsing/main_classes_CN.md#selector) 类继承了哪些属性/方法。

## 基本用法
导入此 Fetcher 的主要方式与其他抓取器相同。

```python
 from scrapling.fetchers import StealthyFetcher
```
解析选项配置见[此处](choosing_CN.md#parser-configuration-in-all-fetchers)

!!! abstract

    `fetch` 方法的异步版本当然是 `async_fetch`。

## 它能做什么？

`StealthyFetcher` 类是 [DynamicFetcher](dynamic_CN.md#introduction) 类的隐匿版本，主要功能包括：

1. 可轻松自动绕过各类 Cloudflare Turnstile/Interstitial。
2. 绕过 CDP 运行时泄漏和 WebRTC 泄漏。
3. 隔离 JS 执行、移除许多 Playwright 指纹，并阻止通过部分已知机器人行为进行的检测。
4. 生成 canvas 噪声以防止通过 canvas 指纹识别。
5. 自动修补检测无头运行的已知方法，并提供选项抵御时区不匹配攻击。
6. 以及其他反防护选项...

## 完整参数列表
Scrapling 为此抓取器及其会话类提供许多选项。跳转到[示例](#examples)前，以下是完整参数列表


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
|  solve_cloudflare   | 启用后，抓取器在返回响应前解决所有类型的 Cloudflare Turnstile/Interstitial 挑战。                                                                                                      |    ✔️    |
|    block_webrtc     | 强制 WebRTC 遵守代理设置，防止本地 IP 地址泄漏。                                                                                                                                                           |    ✔️    |
|     hide_canvas     | 为 canvas 操作添加随机噪声以防止指纹识别。                                                                                                                                                                    |    ✔️    |
|     allow_webgl     | 默认启用。禁用时完全禁用 WebGL 和 WebGL 2.0 支持。禁用 WebGL 不推荐，许多 WAF 会检查 WebGL 是否启用。                                                                     |    ✔️    |
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

在会话类中，这些参数可全局设置。仍可通过传入此处可在浏览器标签页级别配置的参数，为每个请求单独配置：`google_search`、`timeout`、`wait`、`page_action`、`page_setup`、`extra_headers`、`disable_resources`、`wait_selector`、`wait_selector_state`、`network_idle`、`load_dom`、`solve_cloudflare`、`blocked_domains`、`proxy` 和 `selector_config`。

!!! note "注意："

    1. 基本上与 [DynamicFetcher](dynamic_CN.md#introduction) 类参数相同，但额外有：`solve_cloudflare`、`block_webrtc`、`hide_canvas` 和 `allow_webgl`。`capture_xhr` 与 `DynamicFetcher` 共用。
    2. 我的测试中，`disable_resources` 对某些网站使请求快约 25%，也有助于节省代理用量，但需谨慎，可能导致某些网站永远无法加载完成。
    3. `google_search` 参数默认对所有请求启用，将 referer 设为 `https://www.google.com/`。若与 `extra_headers` 同用，其 referer 优先级高于 `extra_headers` 中设置的 referer。
    4. 若未设置 user agent 且启用了无头模式，抓取器会生成同浏览器版本的真实 user agent 并使用。若未设置 user agent 且未启用无头模式，抓取器使用浏览器默认 user agent，与最新版标准浏览器相同。

## 示例
示例更易理解，下面将逐一说明大部分参数。因与 [DynamicFetcher](dynamic_CN.md#introduction) 同类，更多示例可参考该页，此处不重复。

### Cloudflare 与隐匿选项

```python
# Automatic Cloudflare solver
page = StealthyFetcher.fetch('https://nopecha.com/demo/cloudflare', solve_cloudflare=True)

# Works with other stealth options
page = StealthyFetcher.fetch(
    'https://protected-site.com',
    solve_cloudflare=True,
    block_webrtc=True,
    real_chrome=True,
    hide_canvas=True,
    google_search=True,
    proxy='http://username:password@host:port',  # It can also be a dictionary with only the keys 'server', 'username', and 'password'.
)
```

`solve_cloudflare` 参数启用自动检测并解决所有类型的 Cloudflare Turnstile/Interstitial 挑战：

- JavaScript 挑战（托管型）
- 交互式挑战（点击验证框）
- 隐形挑战（后台自动验证）

甚至可解决嵌入验证码的自定义页面。

!!! notes "**重要说明：**"

    1. 对使用自定义实现的网站，有时需在解决验证码后使用 `wait_selector` 确保 Scrapling 等待真实网站内容加载。部分网站可能是边缘案例，我们正努力使求解器尽可能通用。
    2. 使用 Cloudflare 求解器时，超时至少应为 60 秒，以留出足够的挑战求解时间。
    3. 此特性可与代理及其他隐匿选项无缝配合。

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

page = StealthyFetcher.fetch('https://example.com', page_action=scroll_page)
```
若使用异步 fetch，函数也必须是 async。
```python
from playwright.async_api import Page

async def scroll_page(page: Page):
   await page.mouse.wheel(10, 0)
   await page.mouse.move(100, 400)
   await page.mouse.up()

page = await StealthyFetcher.async_fetch('https://example.com', page_action=scroll_page)
```

### 等待条件
```python
# Wait for the selector
page = StealthyFetcher.fetch(
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


### 真实场景示例（Amazon）
仅供教育目的；此示例由 AI 生成，也展示了通过 AI 使用 Scrapling 的简便性
```python
def scrape_amazon_product(url):
    # Use StealthyFetcher to bypass protection
    page = StealthyFetcher.fetch(url)

    # Extract product details
    return {
        'title': page.css('#productTitle::text').get().clean(),
        'price': page.css('.a-price .a-offscreen::text').get(),
        'rating': page.css('[data-feature-name="averageCustomerReviews"] .a-popover-trigger .a-color-base::text').get(),
        'reviews_count': page.css('#acrCustomerReviewText::text').re_first(r'[\d,]+'),
        'features': [
            li.get().clean() for li in page.css('#feature-bullets li span::text')
        ],
        'availability': page.css('#availability')[0].get_all_text(strip=True),
        'images': [
            img.attrib['src'] for img in page.css('#altImages img')
        ]
    }
```

## 会话管理

若要在相同配置下保持浏览器打开以发起多个请求，使用 `StealthySession`/`AsyncStealthySession` 类。这些类可接受 `fetch` 函数的所有参数，便于为整个会话指定配置。

```python
from scrapling.fetchers import StealthySession

# Create a session with default configuration
with StealthySession(
    headless=True,
    real_chrome=True,
    block_webrtc=True,
    solve_cloudflare=True
) as session:
    # Make multiple requests with the same browser instance
    page1 = session.fetch('https://example1.com')
    page2 = session.fetch('https://example2.com') 
    page3 = session.fetch('https://nopecha.com/demo/cloudflare')
    
    # All requests reuse the same tab on the same browser instance
```

### 异步会话用法

```python
import asyncio
from scrapling.fetchers import AsyncStealthySession

async def scrape_multiple_sites():
    async with AsyncStealthySession(
        real_chrome=True,
        block_webrtc=True,
        solve_cloudflare=True,
        timeout=60000,  # 60 seconds for Cloudflare challenges
        max_pages=3
    ) as session:
        # Make async requests with shared browser configuration
        pages = await asyncio.gather(
            session.fetch('https://site1.com'),
            session.fetch('https://site2.com'), 
            session.fetch('https://protected-site.com')
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

## 使用 Camoufox 作为引擎

自 0.3.13 之前，此抓取器使用 [Camoufox](https://github.com/daijro/camoufox) 的定制版本作为引擎，后因多种原因被 [patchright](https://github.com/Kaliiiiiiiiii-Vinyzu/patchright) 取代。若 Camoufox 在你的设备上稳定、无高内存问题且你想继续使用，可以这样做。

首先，若尚未安装，需安装 Camoufox 库、浏览器及 Firefox 系统依赖：
```commandline
pip install camoufox
playwright install-deps firefox
camoufox fetch
```
然后继承 `StealthySession` 并按如下设置：
```python
from scrapling.fetchers import StealthySession
from playwright.sync_api import sync_playwright
from camoufox.utils import launch_options as generate_launch_options

class StealthySession(StealthySession):
    def start(self):
        """Create a browser for this instance and context."""
        if not self.playwright:
            self.playwright = sync_playwright().start()
            # Configure camoufox run options here
            launch_options = generate_launch_options(**{"headless": True, "user_data_dir": ''})
            # Here's an example, part of what we have been doing before v0.3.13
            launch_options = generate_launch_options(**{
                "geoip": False,
                "proxy": self._config.proxy,
                "headless": self._config.headless,
                "humanize": True if self._config.solve_cloudflare else False,  # Better enable humanize for Cloudflare, otherwise it's up to you
                "i_know_what_im_doing": True,  # To turn warnings off with the user configurations
                "allow_webgl": self._config.allow_webgl,
                "block_webrtc": self._config.block_webrtc,
                "os": None,
                "user_data_dir": self._config.user_data_dir,
                "firefox_user_prefs": {
                    # This is what enabling `enable_cache` does internally, so we do it from here instead
                    "browser.sessionhistory.max_entries": 10,
                    "browser.sessionhistory.max_total_viewers": -1,
                    "browser.cache.memory.enable": True,
                    "browser.cache.disk_cache_ssl": True,
                    "browser.cache.disk.smart_size.enabled": True,
                },
                # etc...
            })
            self.context = self.playwright.firefox.launch_persistent_context(**launch_options)
        else:
            raise RuntimeError("Session has been already started")
```
之后可照常使用，甚至用于解决 Cloudflare 挑战：
```python
with StealthySession(solve_cloudflare=True, headless=True) as session:
    page = session.fetch('https://sergiodemo.com/security/challenge/legacy-challenge')
    if page.css('#page-not-found-404'):
        print('Cloudflare challenge solved successfully!')
```

`AsyncStealthySession` 类逻辑相同，略有差异：
```python
from scrapling.fetchers import AsyncStealthySession
from playwright.async_api import async_playwright
from camoufox.utils import launch_options as generate_launch_options

class AsyncStealthySession(AsyncStealthySession):
    async def start(self):
        """Create a browser for this instance and context."""
        if not self.playwright:
            self.playwright = await async_playwright().start()
            # Configure camoufox run options here
            launch_options = generate_launch_options(**{"headless": True, "user_data_dir": ''})
            # or set the launch options as in the above example
            self.context = await self.playwright.firefox.launch_persistent_context(**launch_options)
        else:
            raise RuntimeError("Session has been already started")
 
async with AsyncStealthySession(solve_cloudflare=True, headless=True) as session:
    page = await session.fetch('https://sergiodemo.com/security/challenge/legacy-challenge')
    if page.css('#page-not-found-404'):
        print('Cloudflare challenge solved successfully!')
```

祝使用愉快! :)

## 何时使用

在以下情况使用 StealthyFetcher：

- 绕过反机器人防护
- 需要可靠的浏览器指纹
- 需要完整 JavaScript 支持
- 需要自动隐匿特性
- 需要浏览器自动化
- 应对 Cloudflare 防护
