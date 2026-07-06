# 高级用法

## 简介

!!! success "前置条件"

    1. 你已阅读 [入门指南](getting-started_CN.md) 页面，了解如何创建和运行基础 spider。

本页介绍 spider 系统的高级功能：并发控制、暂停/恢复、流式处理、生命周期钩子、统计和日志。

## 并发控制

spider 系统使用三个类属性控制爬取的激进程度：

| 属性                        | 默认值 | 说明                                                      |
|----------------------------------|---------|------------------------------------------------------------------|
| `concurrent_requests`            | `4`     | 同时处理的最大请求数      |
| `concurrent_requests_per_domain` | `0`     | 每个域名的最大并发请求数（0 = 无按域名限制） |
| `download_delay`                 | `0.0`   | 每次请求前的等待秒数                              |
| `robots_txt_obey`               | `False` | 遵守 robots.txt 规则（Disallow、Crawl-delay、Request-rate）   |

```python
class PoliteSpider(Spider):
    name = "polite"
    start_urls = ["https://example.com"]

    # Be gentle with the server
    concurrent_requests = 4
    concurrent_requests_per_domain = 2
    download_delay = 1.0  # Wait 1 second between requests

    async def parse(self, response: Response):
        yield {"title": response.css("title::text").get("")}
```

设置 `concurrent_requests_per_domain` 后，每个域名除了全局限制外还有自己的并发限制器。同时爬取多个域名时很有用——你可以允许较高的全局并发，同时对每个域名保持礼貌。

!!! tip

    `download_delay` 参数在每次请求前添加固定等待，与域名无关。用于简单的速率限制。

### 使用 uvloop

`start()` 方法接受 `use_uvloop` 参数，在可用时使用更快的 [uvloop](https://github.com/MagicStack/uvloop)/[winloop](https://github.com/nicktimko/winloop) 事件循环实现：

```python
result = MySpider().start(use_uvloop=True)
```

这可以提高 I/O 密集型爬取的吞吐量。你需要单独安装 `uvloop`（Linux/macOS）或 `winloop`（Windows）。

## 暂停与恢复

spider 通过检查点支持优雅暂停与恢复。要启用，向 spider 构造函数传入 `crawldir` 目录：

```python
spider = MySpider(crawldir="crawl_data/my_spider")
result = spider.start()

if result.paused:
    print("Crawl was paused. Run again to resume.")
else:
    print("Crawl completed!")
```

### 工作原理

1. **暂停**：爬取期间按 `Ctrl+C`。spider 等待所有进行中的请求完成，保存检查点（待处理请求 + 已见请求指纹集合），然后退出。
2. **强制停止**：再次按 `Ctrl+C` 可立即停止，不等待活跃任务。
3. **恢复**：使用相同 `crawldir` 再次运行 spider。它会检测检查点，恢复队列和已见集合，从中断处继续，跳过 `start_requests()`。
4. **清理**：爬取正常完成（非暂停）时，检查点文件会自动删除。

**爬取期间也会定期保存检查点（默认每 5 分钟）。** 

可按如下方式更改间隔：

```python
# Save checkpoint every 2 minutes
spider = MySpider(crawldir="crawl_data/my_spider", interval=120.0)
```

磁盘写入是原子的，因此完全安全。

!!! tip

    爬取期间按 `Ctrl+C` 总会使 spider 优雅关闭，即使未启用检查点系统。不等待再次按下会强制 spider 立即关闭。

### 判断是否正在恢复

`on_start()` 钩子接收 `resuming` 标志：

```python
async def on_start(self, resuming: bool = False):
    if resuming:
        self.logger.info("Resuming from checkpoint!")
    else:
        self.logger.info("Starting fresh crawl")
```

## 开发模式

迭代 spider 的 `parse()` 逻辑时，每次运行都重新请求目标服务器既慢又嘈杂。开发模式在首次运行时将每个响应缓存到磁盘，后续运行从磁盘重放，这样你可以调整选择器并反复运行 spider，而无需发起任何网络请求。

通过在 spider 上设置 `development_mode = True` 启用：

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    development_mode = True

    async def parse(self, response: Response):
        yield {"title": response.css("title::text").get("")}
```

首次运行正常获取并将每个响应存储到磁盘。后续每次运行从缓存提供相同请求，完全跳过网络。

### 缓存位置

默认情况下，响应缓存在当前工作目录（运行 spider 的位置，**而非** spider 脚本所在位置）下的 `.scrapling_cache/{spider.name}/` 中。可用 `development_cache_dir` 覆盖位置：

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    development_mode = True
    development_cache_dir = "/tmp/my_spider_cache"
```

### 工作原理

1. **缓存键**：每个响应以请求的指纹为键，因此任何影响指纹的属性（`fp_include_kwargs`、`fp_include_headers`、`fp_keep_fragments`）的更改都会产生新的获取。
2. **存储格式**：每个响应一个 JSON 文件，命名为 `{fingerprint_hex}.json`。body 经 base64 编码以精确保留二进制内容。写入是原子的（临时文件 + 重命名）。
3. **重放**：缓存命中时，引擎完全跳过网络，包括 `download_delay`、速率限制和 `is_blocked()` 重试路径。缓存的响应直接进入你的回调。
4. **统计**：缓存的请求仍计入 `requests_count`、`response_bytes` 和各状态码计数器，因此统计输出与正常爬取相同。两个额外计数器 `cache_hits` 和 `cache_misses` 可查看缓存表现。

### 清除缓存

没有自动过期。要强制全新爬取，删除缓存目录或直接调用 manager 的 `clear()` 方法。

!!! warning

    开发模式用于开发，不用于生产。缓存的响应永不过期，重放会绕过速率限制和被拦截请求重试。不要带着 `development_mode = True` 发布 spider。

## 流式处理

对于长时间运行的 spider 或需要实时访问爬取条目的应用，使用 `stream()` 方法代替 `start()`：

```python
import anyio

async def main():
    spider = MySpider()
    async for item in spider.stream():
        print(f"Got item: {item}")
        # Access real-time stats
        print(f"Items so far: {spider.stats.items_scraped}")
        print(f"Requests made: {spider.stats.requests_count}")

anyio.run(main)
```

与 `start()` 的主要区别：

- `stream()` 必须在异步上下文中调用
- 条目在爬取时逐个产出，而非收集到列表中
- 迭代期间可通过 `spider.stats` 访问实时统计

!!! abstract 

    可通过 `spider.stats` 访问的全部统计列表见下文[此处](#results--statistics)。

你也可以与检查点系统一起使用，便于在 spider 之上构建 UI——具有实时数据且可暂停/恢复的界面。

```python
import anyio

async def main():
    spider = MySpider(crawldir="crawl_data/my_spider")
    async for item in spider.stream():
        print(f"Got item: {item}")
        # Access real-time stats
        print(f"Items so far: {spider.stats.items_scraped}")
        print(f"Requests made: {spider.stats.requests_count}")

anyio.run(main)
```
你也可以在上述代码中使用 `spider.pause()` 关闭 spider。若未启用检查点系统就使用它，只会关闭爬取。

## 生命周期钩子

spider 提供多个可在爬取不同阶段添加自定义行为的钩子：

### on_start

爬取开始前调用。用于加载数据或初始化资源等设置任务：

```python
async def on_start(self, resuming: bool = False):
    self.logger.info("Spider starting up")
    # Load seed URLs from a database, initialize counters, etc.
```

### on_close

爬取结束后调用（无论完成还是暂停）。用于清理：

```python
async def on_close(self):
    self.logger.info("Spider shutting down")
    # Close database connections, flush buffers, etc.
```

### on_error

请求因异常失败时调用。用于错误追踪或自定义恢复逻辑：

```python
async def on_error(self, request: Request, error: Exception):
    self.logger.error(f"Failed: {request.url} - {error}")
    # Log to error tracker, save failed URL for later, etc.
```

### on_scraped_item

每个爬取条目加入结果前调用。返回条目（可修改）以保留，或返回 `None` 以丢弃：

```python
async def on_scraped_item(self, item: dict) -> dict | None:
    # Drop items without a title
    if not item.get("title"):
        return None

    # Modify items (e.g., add timestamps)
    item["scraped_at"] = "2026-01-01"
    return item
```

!!! tip

    此钩子也可用于将条目导向你自己的管道，并从 spider 中丢弃它们。

### start_requests

重写 `start_requests()` 以自定义初始请求生成，代替使用 `start_urls`：

```python
async def start_requests(self):
    # POST request to log in first
    yield Request(
        "https://example.com/login",
        method="POST",
        data={"user": "admin", "pass": "secret"},
        callback=self.after_login,
    )

async def after_login(self, response: Response):
    # Now crawl the authenticated pages
    yield response.follow("/dashboard", callback=self.parse)
```

## 结果与统计

`start()` 返回的 `CrawlResult` 包含爬取条目和详细统计：

```python
result = MySpider().start()

# Items
print(f"Total items: {len(result.items)}")
result.items.to_json("output.json", indent=True)

# Did the crawl complete?
print(f"Completed: {result.completed}")
print(f"Paused: {result.paused}")

# Statistics
stats = result.stats
print(f"Requests: {stats.requests_count}")
print(f"Failed: {stats.failed_requests_count}")
print(f"Blocked: {stats.blocked_requests_count}")
print(f"Offsite filtered: {stats.offsite_requests_count}")
print(f"Robots.txt disallowed: {stats.robots_disallowed_count}")
print(f"Cache hits: {stats.cache_hits}")
print(f"Cache misses: {stats.cache_misses}")
print(f"Items scraped: {stats.items_scraped}")
print(f"Items dropped: {stats.items_dropped}")
print(f"Response bytes: {stats.response_bytes}")
print(f"Duration: {stats.elapsed_seconds:.1f}s")
print(f"Speed: {stats.requests_per_second:.1f} req/s")
```

### 详细统计

`CrawlStats` 对象跟踪细粒度信息：

```python
stats = result.stats

# Status code distribution
print(stats.response_status_count)
# {'status_200': 150, 'status_404': 3, 'status_403': 1}

# Bytes downloaded per domain
print(stats.domains_response_bytes)
# {'example.com': 1234567, 'api.example.com': 45678}

# Requests per session
print(stats.sessions_requests_count)
# {'http': 120, 'stealth': 34}

# Proxies used during the crawl
print(stats.proxies)
# ['http://proxy1:8080', 'http://proxy2:8080']

# Log level counts
print(stats.log_levels_counter)
# {'debug': 200, 'info': 50, 'warning': 3, 'error': 1, 'critical': 0}

# Timing information
print(stats.start_time)       # Unix timestamp when crawl started
print(stats.end_time)         # Unix timestamp when crawl finished
print(stats.download_delay)   # The download delay used (seconds)

# Concurrency settings used
print(stats.concurrent_requests)             # Global concurrency limit
print(stats.concurrent_requests_per_domain)  # Per-domain concurrency limit

# Custom stats (set by your spider code)
print(stats.custom_stats)
# {'login_attempts': 3, 'pages_with_errors': 5}

# Export everything as a dict
print(stats.to_dict())
```

## 日志

spider 有内置日志器，可通过 `self.logger` 访问。它已预配置 spider 名称，并支持多种自定义选项：

| 属性             | 默认值                                                      | 说明                                        |
|-----------------------|--------------------------------------------------------------|----------------------------------------------------|
| `logging_level`       | `logging.DEBUG`                                              | 最低日志级别                                  |
| `logging_format`      | `"[%(asctime)s]:({spider_name}) %(levelname)s: %(message)s"` | 日志消息格式                                 |
| `logging_date_format` | `"%Y-%m-%d %H:%M:%S"`                                        | 日志消息中的日期格式                        |
| `log_file`            | `None`                                                       | 日志文件路径（除控制台输出外） |

```python
import logging

class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    logging_level = logging.INFO
    log_file = "logs/my_spider.log"

    async def parse(self, response: Response):
        self.logger.info(f"Processing {response.url}")
        yield {"title": response.css("title::text").get("")}
```

日志文件目录不存在时会自动创建。控制台和文件输出使用相同格式。
