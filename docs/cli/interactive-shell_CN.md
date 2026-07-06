# Scrapling 交互式 Shell 指南

<script src="https://asciinema.org/a/736339.js" id="asciicast-736339" async data-autoplay="1" data-loop="1" data-cols="225" data-rows="40" data-start-at="00:06" data-speed="1.5" data-theme="tango"></script>

**面向开发者与数据科学家的强大 Web 抓取 REPL**

Scrapling 交互式 Shell 是基于 IPython 的增强环境，专为 Web 抓取任务设计。它提供对 Scrapling 全部功能的即时访问、巧妙快捷方式、自动页面管理，以及 curl 命令转换等高级工具。

!!! success "前置条件"

    1. 你已完成或阅读 [Fetcher 基础](../fetching/choosing_CN.md) 页面，了解 [Response 对象](../fetching/choosing_CN.md#response-object) 是什么以及应使用哪个 Fetcher。
    2. 你已完成或阅读 [查询元素](../parsing/selection_CN.md) 页面，了解如何从 [Selector](../parsing/main_classes_CN.md#selector)/[Response](../fetching/choosing_CN.md#response-object) 对象中查找/提取元素。
    3. 你已完成或阅读 [主要类](../parsing/main_classes_CN.md) 页面，了解 [Response](../fetching/choosing_CN.md#response-object) 类从 [Selector](../parsing/main_classes_CN.md#selector) 类继承了哪些属性/方法。
    4. 你已完成或阅读 Fetcher 章节中至少一页，以便在此发起请求：[HTTP 请求](../fetching/static_CN.md)、[动态网站](../fetching/dynamic_CN.md) 或 [带强防护的动态网站](../fetching/stealthy_CN.md)。


## 为何使用交互式 Shell？

交互式 Shell 将 Web 抓取从缓慢的「写脚本-运行」循环转变为快速探索式体验。非常适合：

- **快速原型**：即时测试抓取策略
- **数据探索**：交互式浏览网站并提取数据
- **学习 Scrapling**：实时试验各项功能
- **调试爬虫**：逐步执行请求并检查结果
- **转换工作流**：一行代码将浏览器 DevTools 中的 curl 命令转换为 Fetcher 请求

## 入门

### 启动 Shell

```bash
# 启动交互式 Shell
scrapling shell

# 执行代码后退出（适合脚本化）
scrapling shell -c "get('https://quotes.toscrape.com'); print(len(page.css('.quote')))"

# 设置日志级别
scrapling shell --loglevel info
```

启动后，你会看到 Scrapling 横幅，可立即开始抓取，如上方视频所示：

```python
# 无需导入——一切就绪！
get('https://news.ycombinator.com')

# 探索页面结构
page.css('a')[:5]  # 查看前 5 个链接

# 优化选择器
stories = page.css('.titleline>a')
len(stories)  # 30

# 提取特定数据
for story in stories[:3]:
...     title = story.text
...     url = story['href']
...     print(f"{title}: {url}")

# 尝试不同方法
titles = page.css('.titleline>a::text')  # 直接提取文本
urls = page.css('.titleline>a::attr(href)')  # 直接提取属性
```

## 内置快捷方式

Shell 提供便捷快捷方式，省去样板代码：

- **`get(url, **kwargs)`** - HTTP GET 请求（替代 `Fetcher.get`）
- **`post(url, **kwargs)`** - HTTP POST 请求（替代 `Fetcher.post`）
- **`put(url, **kwargs)`** - HTTP PUT 请求（替代 `Fetcher.put`）
- **`delete(url, **kwargs)`** - HTTP DELETE 请求（替代 `Fetcher.delete`）
- **`fetch(url, **kwargs)`** - 基于浏览器的获取（替代 `DynamicFetcher.fetch`）
- **`stealthy_fetch(url, **kwargs)`** - 隐蔽浏览器获取（替代 `StealthyFetcher.fetch`）

最常用的类无需导入即可使用，包括 `Fetcher`、`AsyncFetcher`、`DynamicFetcher`、`StealthyFetcher` 和 `Selector`。

### 智能页面管理

Shell 自动跟踪你的请求与页面：

- **当前页面访问**

    `page` 与 `response` 命令会自动更新为最后获取的页面：
    
    ```python
    get('https://quotes.toscrape.com')
    # 'page' 与 'response' 均指向最后获取的页面
    page.url  # 'https://quotes.toscrape.com'
    response.status  # 输出 200；与 page.status 相同
    ```

- **页面历史**

    `pages` 命令跟踪最近五个页面（为 `Selectors` 对象）：
    
    ```python
     get('https://site1.com')
     get('https://site2.com') 
     get('https://site3.com')
    
     # 访问最近 5 个页面
     len(pages)  # 带 `page` 历史的 `Selectors` 对象 -> 3
     pages[0].url  # 历史中第一个页面 -> 'https://site1.com'
     pages[-1].url  # 最近页面 -> 'https://site3.com'
    
     # 处理历史页面
     for i, old_page in enumerate(pages):
    ...     print(f"Page {i}: {old_page.url} - {old_page.status}")
    ```

## 其他实用命令

### 页面可视化

在浏览器中查看抓取的页面：

```python
 get('https://quotes.toscrape.com')
 view(page)  # 在默认浏览器中打开页面 HTML
```

### Curl 命令集成

Shell 提供若干函数，帮助将浏览器 DevTools 中的 curl 命令转换为 `Fetcher` 请求：`uncurl` 与 `curl2fetcher`。

首先，需要像下面这样复制请求为 curl 命令：

<img src="../assets/scrapling_shell_curl.png" title="从 Chrome 复制请求为 curl 命令" alt="从 Chrome 复制请求为 curl 命令" style="width: 70%;"/>

- **将 Curl 命令转换为请求对象**

    ```python
     curl_cmd = '''curl 'https://scrapling.requestcatcher.com/post' \
    ...   -X POST \
    ...   -H 'Content-Type: application/json' \
    ...   -d '{"name": "test", "value": 123}' '''
    
     request = uncurl(curl_cmd)
     request.method  # -> 'post'
     request.url  # -> 'https://scrapling.requestcatcher.com/post'
     request.headers  # -> {'Content-Type': 'application/json'}
    ```

- **直接执行 Curl 命令**

    ```python
     # 一步转换并执行
     curl2fetcher(curl_cmd)
     page.status  # -> 200
     page.json()['json']  # -> {'name': 'test', 'value': 123}
    ```

### IPython 功能

Shell 继承 IPython 的全部能力：

```python
# Magic 命令
%time page = get('https://example.com')  # 计时执行
%history  # 显示命令历史
%save filename.py 1-10  # 将命令 1-10 保存到文件

# 各处均支持 Tab 补全
page.c<TAB>  # 显示：css, cookies, headers 等
Fetcher.<TAB>  # 显示所有 Fetcher 方法

# 对象检查
get? # 显示 get 文档
```

## 示例

以下是 AI 生成的若干示例：

#### 电商数据采集

```python
# 从产品列表页开始
catalog = get('https://shop.example.com/products')

# 查找产品链接
product_links = catalog.css('.product-link::attr(href)')
print(f"Found {len(product_links)} products")

# 先抽样几个产品
for link in product_links[:3]:
...     product = get(f"https://shop.example.com{link}")
...     name = product.css('.product-name::text').get('')
...     price = product.css('.price::text').get('')
...     print(f"{name}: {price}")

# 使用会话提高效率进行扩展
from scrapling.fetchers import FetcherSession
with FetcherSession() as session:
...     products = []
...     for link in product_links:
...         product = session.get(f"https://shop.example.com{link}")
...         products.append({
...             'name': product.css('.product-name::text').get(''),
...             'price': product.css('.price::text').get(''),
...             'url': link
...         })
```

#### API 集成与测试

```python
>>> # 交互式测试 API 端点
>>> response = get('https://jsonplaceholder.typicode.com/posts/1')
>>> response.json()
{'userId': 1, 'id': 1, 'title': 'sunt aut...', 'body': 'quia et...'}

>>> # 测试 POST 请求
>>> new_post = post('https://jsonplaceholder.typicode.com/posts', 
...                 json={'title': 'Test Post', 'body': 'Test content', 'userId': 1})
>>> new_post.json()['id']
101

>>> # 使用不同数据测试
>>> updated = put(f'https://jsonplaceholder.typicode.com/posts/{new_post.json()["id"]}',
...               json={'title': 'Updated Title'})
```

## 获取帮助

若终端内帮助不足，可以：

- [Scrapling 文档](https://scrapling.readthedocs.io/)
- [Discord 社区](https://discord.gg/EMgGbDceNQ)
- [GitHub Issues](https://github.com/D4Vinci/Scrapling/issues)  

就是这样！祝抓取愉快！Shell 让 Web 抓取像对话一样简单。
