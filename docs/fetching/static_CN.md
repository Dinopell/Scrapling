# HTTP 请求

`Fetcher` 类通过高性能 `curl_cffi` 库提供快速轻量的 HTTP 请求，并具备多种隐匿能力。

!!! success "前置条件"

    1. 你已完成或阅读过[抓取器基础](../fetching/choosing_CN.md)页面，了解 [Response 对象](../fetching/choosing_CN.md#response-object) 是什么以及该用哪种抓取器。
    2. 你已完成或阅读过[查询元素](../parsing/selection_CN.md)页面，了解如何从 [Selector](../parsing/main_classes_CN.md#selector)/[Response](../fetching/choosing_CN.md#response-object) 对象中查找/提取元素。
    3. 你已完成或阅读过[核心类](../parsing/main_classes_CN.md)页面，了解 [Response](../fetching/choosing_CN.md#response-object) 类从 [Selector](../parsing/main_classes_CN.md#selector) 类继承了哪些属性/方法。

## 基本用法
导入此 Fetcher 的主要方式与其他抓取器相同。

```python
from scrapling.fetchers import Fetcher
```
解析选项配置见[此处](choosing_CN.md#parser-configuration-in-all-fetchers)

### 共用参数
此处所有请求方法共享部分参数，先说明如下。

- **url**：目标 URL
- **stealthy_headers**：若启用（默认），会创建并添加真实浏览器请求头，同时设置 Google referer 头。
- **follow_redirects**：控制重定向行为。**默认为 `"safe"`**，即跟随重定向但拒绝指向内网/私有 IP 的重定向（SSRF 防护）。传 `True` 可无限制跟随所有重定向，传 `False` 完全禁用重定向。
- **timeout**：等待每个请求完成的秒数。**默认 30 秒**。
- **retries**：失败请求的重试次数。**默认 3 次**。
- **retry_delay**：重试间隔秒数。**默认 1 秒**。
- **impersonate**：模拟特定浏览器的 TLS 指纹。可接受浏览器字符串或列表，如 `"chrome110"`、`"firefox102"`、`"safari15_5"` 指定版本，或 `"chrome"`、`"firefox"`、`"safari"`、`"edge"` 自动使用最新版本。使请求在 TLS 层面看起来像真实浏览器。传入字符串列表时每次请求随机选一个。**默认使用最新可用 Chrome 版本**。
- **http3**：使用 HTTP/3 协议。**默认为 False**。与 `impersonate` 同用可能有问题。
- **cookies**：请求使用的 Cookie。可为 `name→value` 字典或字典列表。
- **proxy**：本请求使用的代理，路由所有流量（HTTP 和 HTTPS）。格式：`http://username:password@localhost:8030`。
- **proxy_auth**：代理 HTTP 基本认证，(username, password) 元组。
- **proxies**：代理字典，格式：`{"http": proxy_url, "https": proxy_url}`。
- **proxy_rotator**：用于自动代理轮换的 `ProxyRotator` 实例。不能与 `proxy` 或 `proxies` 同时使用。
- **headers**：请求头，可覆盖 `stealthy_headers` 生成的任何头
- **max_redirects**：最大重定向次数。**默认 30**，-1 表示无限制。
- **verify**：是否验证 HTTPS 证书。**默认为 True**。
- **cert**：客户端证书的 (cert, key) 文件名元组。
- **selector_config**：创建最终 `Selector`/`Response` 类时使用的自定义解析参数字典。

!!! note "注意："

    1. 当前可模拟的浏览器包括（`"edge"`、`"chrome"`、`"chrome_android"`、`"safari"`、`"safari_beta"`、`"safari_ios"`、`"safari_ios_beta"`、`"firefox"`、`"tor"`）<br/>
    2. 可模拟的浏览器及对应版本会在参数自动补全中显示，并随 `curl_cffi` 更新而更新。<br/>
    3. 若启用 `impersonate` 或 `stealthy_headers` 任一参数，抓取器会自动生成与所用浏览器版本匹配的真实浏览器请求头。

除此之外，若方法本身尚不支持，可传入 `curl_cffi` 支持的任意参数以进一步自定义。

### HTTP 方法
各方法根据方法类型有额外参数，如 GET 的 `params`、POST/PUT/DELETE 的 `data`/`json`。

示例最能说明问题：

> 因此：不支持 `OPTIONS` 和 `HEAD` 方法。
#### GET
```python
from scrapling.fetchers import Fetcher
# Basic GET
page = Fetcher.get('https://example.com')
page = Fetcher.get('https://scrapling.requestcatcher.com/get', stealthy_headers=True)
page = Fetcher.get('https://scrapling.requestcatcher.com/get', proxy='http://username:password@localhost:8030')
# With parameters
page = Fetcher.get('https://example.com/search', params={'q': 'query'})

# With headers
page = Fetcher.get('https://example.com', headers={'User-Agent': 'Custom/1.0'})
# Basic HTTP authentication
page = Fetcher.get("https://example.com", auth=("my_user", "password123"))
# Browser impersonation
page = Fetcher.get('https://example.com', impersonate='chrome')
# HTTP/3 support
page = Fetcher.get('https://example.com', http3=True)
```
异步请求只需小幅调整
```python
from scrapling.fetchers import AsyncFetcher
# Basic GET
page = await AsyncFetcher.get('https://example.com')
page = await AsyncFetcher.get('https://scrapling.requestcatcher.com/get', stealthy_headers=True)
page = await AsyncFetcher.get('https://scrapling.requestcatcher.com/get', proxy='http://username:password@localhost:8030')
# With parameters
 page = await AsyncFetcher.get('https://example.com/search', params={'q': 'query'})
>>>
# With headers
page = await AsyncFetcher.get('https://example.com', headers={'User-Agent': 'Custom/1.0'})
# Basic HTTP authentication
page = await AsyncFetcher.get("https://example.com", auth=("my_user", "password123"))
# Browser impersonation
page = await AsyncFetcher.get('https://example.com', impersonate='chrome110')
# HTTP/3 support
page = await AsyncFetcher.get('https://example.com', http3=True)
```
毋庸置疑，所有情况下的 `page` 对象都是 [Response](choosing_CN.md#response-object) 对象，即我们所说的 [Selector](../parsing/main_classes_CN.md#selector)，可直接使用
```python
page.css('.something.something')

page = Fetcher.get('https://api.github.com/events')
>>> page.json()
[{'id': '<redacted>',
  'type': 'PushEvent',
  'actor': {'id': '<redacted>',
   'login': '<redacted>',
   'display_login': '<redacted>',
   'gravatar_id': '',
   'url': 'https://api.github.com/users/<redacted>',
   'avatar_url': 'https://avatars.githubusercontent.com/u/<redacted>'},
  'repo': {'id': '<redacted>',
...
```
#### POST
```python
from scrapling.fetchers import Fetcher
# Basic POST
page = Fetcher.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'}, params={'q': 'query'})
page = Fetcher.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'}, stealthy_headers=True)
page = Fetcher.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'}, proxy='http://username:password@localhost:8030', impersonate="chrome")
# Another example of form-encoded data
page = Fetcher.post('https://example.com/submit', data={'username': 'user', 'password': 'pass'}, http3=True)
# JSON data
page = Fetcher.post('https://example.com/api', json={'key': 'value'})
```
异步请求只需小幅调整
```python
from scrapling.fetchers import AsyncFetcher
# Basic POST
page = await AsyncFetcher.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'})
page = await AsyncFetcher.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'}, stealthy_headers=True)
page = await AsyncFetcher.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'}, proxy='http://username:password@localhost:8030', impersonate="chrome")
# Another example of form-encoded data
page = await AsyncFetcher.post('https://example.com/submit', data={'username': 'user', 'password': 'pass'}, http3=True)
# JSON data
page = await AsyncFetcher.post('https://example.com/api', json={'key': 'value'})
```
#### PUT
```python
from scrapling.fetchers import Fetcher
# Basic PUT
page = Fetcher.put('https://example.com/update', data={'status': 'updated'})
page = Fetcher.put('https://example.com/update', data={'status': 'updated'}, stealthy_headers=True, impersonate="chrome")
page = Fetcher.put('https://example.com/update', data={'status': 'updated'}, proxy='http://username:password@localhost:8030')
# Another example of form-encoded data
page = Fetcher.put("https://scrapling.requestcatcher.com/put", data={'key': ['value1', 'value2']})
```
异步请求只需小幅调整
```python
from scrapling.fetchers import AsyncFetcher
# Basic PUT
page = await AsyncFetcher.put('https://example.com/update', data={'status': 'updated'})
page = await AsyncFetcher.put('https://example.com/update', data={'status': 'updated'}, stealthy_headers=True, impersonate="chrome")
page = await AsyncFetcher.put('https://example.com/update', data={'status': 'updated'}, proxy='http://username:password@localhost:8030')
# Another example of form-encoded data
page = await AsyncFetcher.put("https://scrapling.requestcatcher.com/put", data={'key': ['value1', 'value2']})
```

#### DELETE
```python
from scrapling.fetchers import Fetcher
page = Fetcher.delete('https://example.com/resource/123')
page = Fetcher.delete('https://example.com/resource/123', stealthy_headers=True, impersonate="chrome")
page = Fetcher.delete('https://example.com/resource/123', proxy='http://username:password@localhost:8030')
```
异步请求只需小幅调整
```python
from scrapling.fetchers import AsyncFetcher
page = await AsyncFetcher.delete('https://example.com/resource/123')
page = await AsyncFetcher.delete('https://example.com/resource/123', stealthy_headers=True, impersonate="chrome")
page = await AsyncFetcher.delete('https://example.com/resource/123', proxy='http://username:password@localhost:8030')
```

## 会话管理

若要用相同配置发起多个请求，使用 `FetcherSession` 类。可在同步和异步代码中使用；该类自动检测并切换会话类型，无需不同导入。

`FetcherSession` 几乎可接受与方法相同的所有参数，便于为整个会话指定配置，之后可轻松为某个请求选用不同配置，如下例所示。

```python
from scrapling.fetchers import FetcherSession

# Create a session with default configuration
with FetcherSession(
    impersonate='chrome',
    http3=True,
    stealthy_headers=True,
    timeout=30,
    retries=3
) as session:
    # Make multiple requests with the same settings and the same cookies
    page1 = session.get('https://scrapling.requestcatcher.com/get')
    page2 = session.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'})
    page3 = session.get('https://api.github.com/events')

    # All requests share the same session and connection pool
```

也可将 `ProxyRotator` 与 `FetcherSession` 配合，在多个请求间自动轮换代理：

```python
from scrapling.fetchers import FetcherSession, ProxyRotator

rotator = ProxyRotator([
    'http://proxy1:8080',
    'http://proxy2:8080',
    'http://proxy3:8080',
])

with FetcherSession(proxy_rotator=rotator, impersonate='chrome') as session:
    # Each request automatically uses the next proxy in rotation
    page1 = session.get('https://example.com/page1')
    page2 = session.get('https://example.com/page2')

    # You can check which proxy was used via the response metadata
    print(page1.meta['proxy'])
```

也可通过直接向请求方法传入 `proxy=` 为特定请求覆盖会话代理（或轮换器）：

```python
with FetcherSession(proxy='http://default-proxy:8080') as session:
    # Uses the session proxy
    page1 = session.get('https://example.com/page1')

    # Override the proxy for this specific request
    page2 = session.get('https://example.com/page2', proxy='http://special-proxy:9090')
```

异步示例

```python
async with FetcherSession(impersonate='firefox', http3=True) as session:
    # All standard HTTP methods available
    response = await session.get('https://example.com')
    response = await session.post('https://scrapling.requestcatcher.com/post', json={'data': 'value'})
    response = await session.put('https://scrapling.requestcatcher.com/put', data={'update': 'info'})
    response = await session.delete('https://scrapling.requestcatcher.com/delete')
```
或更好
```python
import asyncio
from scrapling.fetchers import FetcherSession

# Async session usage
async with FetcherSession(impersonate="safari") as session:
    urls = ['https://example.com/page1', 'https://example.com/page2']

    tasks = [
        session.get(url) for url in urls
    ]

    pages = await asyncio.gather(*tasks)
```

`Fetcher` 类在每次请求时使用 `FetcherSession` 创建临时会话。

### 会话优势

- **快得多**：比每次请求单独创建会话快约 10 倍
- **Cookie 持久化**：跨请求自动处理 Cookie
- **资源效率**：多请求时内存和 CPU 占用更优
- **集中配置**：统一管理请求设置

## 示例
面向爬虫新手的完整示例

### 基本 HTTP 请求

```python
from scrapling.fetchers import Fetcher

# Make a request
page = Fetcher.get('https://example.com')

# Check the status
if page.status == 200:
    # Extract title
    title = page.css('title::text').get()
    print(f"Page title: {title}")

    # Extract all links
    links = page.css('a::attr(href)').getall()
    print(f"Found {len(links)} links")
```

### 商品爬取

```python
from scrapling.fetchers import Fetcher

def scrape_products():
    page = Fetcher.get('https://example.com/products')
    
    # Find all product elements
    products = page.css('.product')
    
    results = []
    for product in products:
        results.append({
            'title': product.css('.title::text').get(),
            'price': product.css('.price::text').re_first(r'\d+\.\d{2}'),
            'description': product.css('.description::text').get(),
            'in_stock': product.has_class('in-stock')
        })
    
    return results
```

### 下载文件

```python
from scrapling.fetchers import Fetcher

page = Fetcher.get('https://raw.githubusercontent.com/D4Vinci/Scrapling/main/docs/assets/main_cover.png')
with open(file='main_cover.png', mode='wb') as f:
   f.write(page.body)
```

### 分页处理

```python
from scrapling.fetchers import Fetcher

def scrape_all_pages():
    base_url = 'https://example.com/products?page={}'
    page_num = 1
    all_products = []
    
    while True:
        # Get current page
        page = Fetcher.get(base_url.format(page_num))
        
        # Find products
        products = page.css('.product')
        if not products:
            break
            
        # Process products
        for product in products:
            all_products.append({
                'name': product.css('.name::text').get(),
                'price': product.css('.price::text').get()
            })
            
        # Next page
        page_num += 1
        
    return all_products
```

### 表单提交

```python
from scrapling.fetchers import Fetcher

# Submit login form
response = Fetcher.post(
    'https://example.com/login',
    data={
        'username': 'user@example.com',
        'password': 'password123'
    }
)

# Check login success
if response.status == 200:
    # Extract user info
    user_name = response.css('.user-name::text').get()
    print(f"Logged in as: {user_name}")
```

### 表格提取

```python
from scrapling.fetchers import Fetcher

def extract_table():
    page = Fetcher.get('https://example.com/data')
    
    # Find table
    table = page.css('table')[0]
    
    # Extract headers
    headers = [
        th.text for th in table.css('thead th')
    ]
    
    # Extract rows
    rows = []
    for row in table.css('tbody tr'):
        cells = [td.text for td in row.css('td')]
        rows.append(dict(zip(headers, cells)))
        
    return rows
```

### 导航菜单

```python
from scrapling.fetchers import Fetcher

def extract_menu():
    page = Fetcher.get('https://example.com')
    
    # Find navigation
    nav = page.css('nav')[0]
    
    menu = {}
    for item in nav.css('li'):
        links = item.css('a')
        if links:
            link = links[0]
            menu[link.text] = {
                'url': link['href'],
                'has_submenu': bool(item.css('.submenu'))
            }
            
    return menu
```

## 何时使用

在以下情况使用 `Fetcher`：

- 需要快速 HTTP 请求。
- 希望开销最小。
- 不需要 JavaScript 执行（网站可通过请求爬取）。
- 需要一定隐匿特性（例如目标站有防护但未使用 JavaScript 挑战）。

在以下情况使用 `FetcherSession`：

- 向同一或不同站点发起多个请求。
- 需要在请求间保持 Cookie/认证。
- 需要连接池以提升性能。
- 需要跨请求的一致配置。
- 处理需要会话状态的 API。

在以下情况使用其他抓取器：

- 需要浏览器自动化。
- 需要高级反机器人/隐匿能力。
- 需要 JavaScript 支持或与动态内容交互。
