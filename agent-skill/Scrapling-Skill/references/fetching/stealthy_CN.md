# StealthyFetcher

`StealthyFetcher` 是基于浏览器的隐匿型 fetcher，与 [DynamicFetcher](dynamic_CN.md) 类似，使用 [Playwright 的 API](https://playwright.dev/python/docs/intro)。它增加高级反机器人防护绕过能力，多数自动处理。与 `DynamicFetcher` 共享相同的浏览器自动化模型，使用 [Playwright 的 Page API](https://playwright.dev/python/docs/api/class-page) 进行页面交互。

## 基本用法
导入此 Fetcher 的主要方式只有一种，与其他 fetcher 相同。

```python
from scrapling.fetchers import StealthyFetcher
```
了解如何配置解析选项，请参见[此处](choosing_CN.md#parser-configuration-in-all-fetchers)

**说明：** `fetch` 方法的异步版本为 `async_fetch`。

## 它能做什么？

`StealthyFetcher` 类是 [DynamicFetcher](dynamic_CN.md) 的隐匿版本，主要能力包括：

1. 可轻松自动绕过各类 Cloudflare Turnstile/Interstitial。
2. 绕过 CDP runtime 泄漏与 WebRTC 泄漏。
3. 隔离 JS 执行、移除多种 Playwright 指纹，并阻止通过部分已知机器人行为进行的检测。
4. 生成 canvas 噪声以防止 canvas 指纹。
5. 自动修补检测无头模式的已知方法，并提供选项以抵御时区不匹配攻击。
6. 以及其他反防护选项……

## 完整参数列表
Scrapling 为此 fetcher 及其会话类提供许多选项。在查看[示例](#examples)前，以下是完整参数列表


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
|  solve_cloudflare   | 启用后，fetcher 会在返回响应前解决各类 Cloudflare Turnstile/Interstitial 挑战。                                                                                                      |    ✔️    |
|    block_webrtc     | 强制 WebRTC 遵守代理设置，防止本地 IP 地址泄漏。                                                                                                                                                           |    ✔️    |
|     hide_canvas     | 为 canvas 操作添加随机噪声以防止指纹。                                                                                                                                                                    |    ✔️    |
|     allow_webgl     | 默认启用。禁用时将完全关闭 WebGL 与 WebGL 2.0 支持。不建议禁用 WebGL，许多 WAF 会检查 WebGL 是否启用。                                                                     |    ✔️    |
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

在会话类中，这些参数可全局设置。仍可通过传入部分可在浏览器标签页级别配置的参数，为单次请求单独配置，例如：`google_search`、`timeout`、`wait`、`page_action`、`page_setup`、`extra_headers`、`disable_resources`、`wait_selector`、`wait_selector_state`、`network_idle`、`load_dom`、`solve_cloudflare`、`blocked_domains`、`proxy` 和 `selector_config`。

**说明：**

1. 参数基本与 [DynamicFetcher](dynamic_CN.md) 相同，另增：`solve_cloudflare`、`block_webrtc`、`hide_canvas` 和 `allow_webgl`。
2. 测试中 `disable_resources` 对部分网站约提速 25%，并有助于节省代理流量，但需谨慎，可能导致部分网站永远无法完成加载。
3. `google_search` 默认对所有请求启用，将 referer 设为 `https://www.google.com/`。与 `extra_headers` 同时使用时，其 referer 优先级高于后者。
4. 若未设置 user agent 且启用了无头模式，fetcher 会为同浏览器版本生成真实 user agent 并使用。若未设置 user agent 且未启用无头模式，fetcher 会使用浏览器默认 user agent，与最新版标准浏览器相同。

## 示例

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

`solve_cloudflare` 参数启用自动检测并解决各类 Cloudflare Turnstile/Interstitial 挑战：

- JavaScript 挑战（托管型）
- 交互式挑战（点击验证框）
- 隐形挑战（后台自动验证）

甚至可解决嵌入 captcha 的自定义页面。

**重要说明：**

1. 对使用自定义实现的网站，有时需在解决 captcha 后使用 `wait_selector`，确保 Scrapling 等待真实网站内容加载。部分网站可能是典型的边缘情况，我们正尽力让 solver 尽可能通用。
2. 使用 Cloudflare solver 时，timeout 至少应为 60 秒，以留出足够挑战解决时间。
3. 此功能可与代理及其他隐匿选项无缝配合。

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

page = StealthyFetcher.fetch('https://example.com', page_action=scroll_page)
```
若使用异步 fetch 版本，函数也必须是 async。
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


### 真实场景示例（Amazon）
仅供学习；此示例由 AI 生成，也展示了通过 AI 使用 Scrapling 的便捷性
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

若要在相同配置下保持浏览器打开并发起多次请求，请使用 `StealthySession`/`AsyncStealthySession` 类。这些类可接受 `fetch` 函数的全部参数，便于为整个会话指定配置。

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

在以下情况使用 StealthyFetcher：

- 绕过反机器人防护
- 需要可靠的浏览器指纹
- 需要完整 JavaScript 支持
- 希望自动隐匿功能
- 需要浏览器自动化
- 应对 Cloudflare 防护
