# 代理管理与拦截处理

Scrapling 的 `ProxyRotator` 在多个请求间管理代理轮换。它适用于所有会话类型，并与 spider 的被拦截请求重试系统集成。

## ProxyRotator

`ProxyRotator` 类管理代理列表并自动轮换。通过 `proxy_rotator` 参数可将其传给任意会话类型：

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

每个请求会自动使用轮换中的下一个代理。所用代理保存在 `response.meta["proxy"]` 中，便于追踪哪个代理抓取了哪一页。


浏览器会话同时支持字符串和字典两种代理格式：

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

**重要说明：**

1. 不能在同一会话上同时使用 `proxy_rotator` 与静态的 `proxy` 或 `proxies` 参数。配置会话时选一种方式，必要时可在单个请求上覆盖。
2. 默认情况下，所有基于浏览器的会话使用带标签页池的持久浏览器上下文。但由于浏览器无法按标签页设置代理，使用 `ProxyRotator` 时，fetcher 会为每个代理自动打开独立上下文，每个上下文一个标签页。标签页任务完成后，标签页及其上下文都会关闭。

## 自定义轮换策略

默认情况下，`ProxyRotator` 使用循环轮换——按顺序遍历代理，末尾后从头开始。

你可以提供自定义策略函数来改变行为，但签名须与下面一致：

```python
from scrapling.core._types import ProxyType

def my_strategy(proxies: list, current_index: int) -> tuple[ProxyType, int]:
    ...
```

它接收代理列表和当前索引，必须返回所选代理及下一个索引。

下面是一些可用的自定义轮换策略示例。

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

可通过关键字参数 `proxy=` 为单个请求覆盖轮换器：

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

当某些页面需要特定代理时很有用（例如面向特定地区的地理定位代理）。

## 被拦截请求处理

spider 内置被拦截请求检测与重试。默认将以下 HTTP 状态码视为被拦截：`401`、`403`、`407`、`429`、`444`、`500`、`502`、`503`、`504`。

重试机制工作流程如下：

1. 响应返回后，spider 调用 `is_blocked(response)` 方法。
2. 若判定为拦截，会复制该请求并调用 `retry_blocked_request()`，以便你在重试前修改请求。
3. 重试请求以 `dont_filter=True`（绕过去重）和更低优先级重新入队，因此不会立即再次重试。
4. 上述过程最多重复 `max_blocked_retries` 次（默认：3）。

**提示：**

1. 重试时会自动清除请求上之前的 `proxy`/`proxies` kwargs，以便轮换器分配新代理。
2. `max_blocked_retries` 属性与会话级重试不同，计数器不共享。

### 自定义拦截检测

重写 `is_blocked()` 以添加自定义检测逻辑：

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

上面的做法是保持拦截检测逻辑不变，让 spider 主要使用 requests，直到被拦截后再切换到隐身浏览器。


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
上述逻辑是：先用廉价代理（如数据中心代理）发起请求，被拦截后改用更高质量代理（如住宅或移动代理）重试。
