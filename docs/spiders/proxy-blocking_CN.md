# 代理管理与拦截处理

## 简介

!!! success "前置条件"

    1. 你已阅读 [入门指南](getting-started_CN.md) 页面，了解如何创建和运行基础 spider。
    2. 你已阅读 [Session](sessions_CN.md) 页面，了解如何配置 session。

大规模爬取时，通常需要在多个代理之间轮换，以避免速率限制和拦截。Scrapling 的 `ProxyRotator` 让这变得简单。它适用于所有 session 类型，并与 spider 的被拦截请求重试系统集成。

如果你不了解代理是什么或如何选择合适的代理，[这篇指南可能有帮助](https://substack.thewebscraping.club/p/everything-about-proxies)。

## ProxyRotator

`ProxyRotator` 类管理代理列表并自动轮换。通过 `proxy_rotator` 参数将其传递给任意 session 类型：

```python
from scrapling.spiders import Spider, Response
from scrapling.fetchers import FetcherSession, ProxyRotator

class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]

    def configure_sessions(self, manager):
        rotator = ProxyRotator([
            "http://proxy1:8080",
            "http://proxy2:8080",
            "http://user:pass@proxy3:8080",
        ])
        manager.add("default", FetcherSession(proxy_rotator=rotator))

    async def parse(self, response: Response):
        # Check which proxy was used
        print(f"Proxy used: {response.meta.get('proxy')}")
        yield {"title": response.css("title::text").get("")}
```

每个请求会自动获得轮换中的下一个代理。使用的代理存储在 `response.meta["proxy"]` 中，便于追踪哪个代理获取了哪个页面。


与浏览器 session 一起使用时，需要做一些调整，如下所示：

```python
from scrapling.fetchers import AsyncDynamicSession, AsyncStealthySession, ProxyRotator

# String proxies work for all session types
rotator = ProxyRotator([
    "http://proxy1:8080",
    "http://proxy2:8080",
])

# Dict proxies (Playwright format) work for browser sessions
rotator = ProxyRotator([
    {"server": "http://proxy1:8080", "username": "user", "password": "pass"},
    {"server": "http://proxy2:8080"},
])

# Then inside the spider
def configure_sessions(self, manager):
    rotator = ProxyRotator(["http://proxy1:8080", "http://proxy2:8080"])
    manager.add("browser", AsyncStealthySession(proxy_rotator=rotator))
```

!!! info

    1. 你不能在同一 session 上同时使用 `proxy_rotator` 参数与静态的 `proxy` 或 `proxies` 参数。配置 session 时选择一种方式，之后如需可按请求覆盖，下文会说明。
    2. 请记住，默认情况下，所有基于浏览器的 session 使用带标签池的持久浏览器上下文。但由于浏览器无法为每个标签页设置代理，使用 `ProxyRotator` 时，fetcher 会为每个代理自动打开单独的上下文，每个上下文一个标签页。标签页任务完成后，标签页及其上下文都会关闭。

## 自定义轮换策略

默认情况下，`ProxyRotator` 使用循环轮换，按顺序遍历代理，到末尾后从头开始。

你可以提供自定义策略函数来改变此行为，但必须符合以下签名：

```python
from scrapling.core._types import ProxyType

def my_strategy(proxies: list, current_index: int) -> tuple[ProxyType, int]:
    ...
```

它接收代理列表和当前索引，必须返回选中的代理和下一个索引。

以下是一些可用的自定义轮换策略示例。

### 随机轮换

```python
import random
from scrapling.fetchers import ProxyRotator

def random_strategy(proxies, current_index):
    idx = random.randint(0, len(proxies) - 1)
    return proxies[idx], idx

rotator = ProxyRotator(
    ["http://proxy1:8080", "http://proxy2:8080", "http://proxy3:8080"],
    strategy=random_strategy,
)
```

### 加权轮换

```python
import random

def weighted_strategy(proxies, current_index):
    # First proxy gets 60% of traffic, others split the rest
    weights = [60] + [40 // (len(proxies) - 1)] * (len(proxies) - 1)
    proxy = random.choices(proxies, weights=weights, k=1)[0]
    return proxy, current_index  # Index doesn't matter for weighted

rotator = ProxyRotator(proxies, strategy=weighted_strategy)
```


## 按请求覆盖代理

你可以通过传递 `proxy=` 关键字参数为单个请求覆盖轮换器：

```python
async def parse(self, response: Response):
    # This request uses the rotator's next proxy
    yield response.follow("/page1", callback=self.parse_page)

    # This request uses a specific proxy, bypassing the rotator
    yield response.follow(
        "/special-page",
        callback=self.parse_page,
        proxy="http://special-proxy:8080",
    )
```

当某些页面需要特定代理时很有用（例如，用于地区特定内容的地理位置代理）。

## 被拦截请求处理

spider 内置被拦截请求检测和重试。默认情况下，以下 HTTP 状态码被视为被拦截：`401`、`403`、`407`、`429`、`444`、`500`、`502`、`503`、`504`。

重试系统的工作方式如下：

1. 响应返回后，spider 调用 `is_blocked(response)` 方法。
2. 若被拦截，它会复制请求并调用 `retry_blocked_request()` 方法，以便你在重试前修改请求。
3. 重试的请求会以 `dont_filter=True`（绕过去重）和较低优先级重新入队，因此不会立即重试。
4. 此过程最多重复 `max_blocked_retries` 次（默认：3）。

!!! tip

    1. 重试时，之前的 `proxy`/`proxies` kwargs 会自动从请求中清除，以便轮换器分配新代理。
    2. `max_blocked_retries` 属性与 session 重试不同，不共享计数器。

### 自定义拦截检测

重写 `is_blocked()` 以添加你自己的检测逻辑：

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]

    async def is_blocked(self, response: Response) -> bool:
        # Check status codes (default behavior)
        if response.status in {403, 429, 503}:
            return True

        # Check response content
        body = response.body.decode("utf-8", errors="ignore")
        if "access denied" in body.lower() or "rate limit" in body.lower():
            return True

        return False

    async def parse(self, response: Response):
        yield {"title": response.css("title::text").get("")}
```

### 自定义重试

重写 `retry_blocked_request()` 以在重试前修改请求。`max_blocked_retries` 属性控制被拦截请求的重试次数（默认：3）：

```python
from scrapling.spiders import Spider, SessionManager, Request, Response
from scrapling.fetchers import FetcherSession, AsyncStealthySession


class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    max_blocked_retries = 5

    def configure_sessions(self, manager: SessionManager) -> None:
        manager.add('requests', FetcherSession(impersonate=['chrome', 'firefox', 'safari']))
        manager.add('stealth', AsyncStealthySession(block_webrtc=True), lazy=True)

    async def retry_blocked_request(self, request: Request, response: Response) -> Request:
        request.sid = "stealth"
        self.logger.info(f"Retrying blocked request: {request.url}")
        return request

    async def parse(self, response: Response):
        yield {"title": response.css("title::text").get("")}
```

上面的做法是：拦截检测逻辑保持不变，spider 主要使用 requests，直到被拦截后切换到 stealth 浏览器。


综合示例：

```python
from scrapling.spiders import Spider, SessionManager, Request, Response
from scrapling.fetchers import FetcherSession, AsyncStealthySession, ProxyRotator


cheap_proxies = ProxyRotator([ "http://proxy1:8080", "http://proxy2:8080"])

# A format acceptable by the browser
expensive_proxies = ProxyRotator([
    {"server": "http://residential_proxy1:8080", "username": "user", "password": "pass"},
    {"server": "http://residential_proxy2:8080", "username": "user", "password": "pass"},
    {"server": "http://mobile_proxy1:8080", "username": "user", "password": "pass"},
    {"server": "http://mobile_proxy2:8080", "username": "user", "password": "pass"},
])


class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    max_blocked_retries = 5

    def configure_sessions(self, manager: SessionManager) -> None:
        manager.add('requests', FetcherSession(impersonate=['chrome', 'firefox', 'safari'], proxy_rotator=cheap_proxies))
        manager.add('stealth', AsyncStealthySession(block_webrtc=True, proxy_rotator=expensive_proxies), lazy=True)

    async def retry_blocked_request(self, request: Request, response: Response) -> Request:
        request.sid = "stealth"
        self.logger.info(f"Retrying blocked request: {request.url}")
        return request

    async def parse(self, response: Response):
        yield {"title": response.css("title::text").get("")}
```
上述逻辑是：请求使用廉价代理（如数据中心代理），直到被拦截后，使用更高质量的代理（如住宅或移动代理）重试。
