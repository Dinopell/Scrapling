# 高级用法

## 并发控制

spider 系统使用三个类属性控制爬取强度：

| Attribute                        | Default | Description                                                      |
|----------------------------------|---------|------------------------------------------------------------------|
| `concurrent_requests`            | `4`     | Maximum number of requests being processed at the same time      |
| `concurrent_requests_per_domain` | `0`     | Maximum concurrent requests per domain (0 = no per-domain limit) |
| `download_delay`                 | `0.0`   | Seconds to wait before each request                              |
| `robots_txt_obey`               | `False` | Respect robots.txt rules (Disallow, Crawl-delay, Request-rate)   |

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

设置 `concurrent_requests_per_domain` 后，每个域名在全局限制之外还有各自的并发限制器。同时爬取多个域名时很有用——可在全局保持较高并发的同时，对每个域名保持礼貌。

**提示：** `download_delay` 会在每个请求前加入固定等待，与域名无关。适合简单的速率限制。

### 使用 uvloop

`start()` 方法接受 `use_uvloop` 参数，在可用时使用更快的 [uvloop](https://github.com/MagicStack/uvloop)/[winloop](https://github.com/nicktimko/winloop) 事件循环实现：

```python
result = MySpider().start(use_uvloop=True)
```

可提升 I/O 密集型爬取的吞吐量。需单独安装 `uvloop`（Linux/macOS）或 `winloop`（Windows）。

## 暂停与恢复

spider 通过检查点支持优雅暂停与恢复。在 spider 构造函数中传入 `crawldir` 目录即可启用：

```python
spider = MySpider(crawldir="crawl_data/my_spider")
result = spider.start()

if result.paused:
    print("Crawl was paused. Run again to resume.")
else:
    print("Crawl completed!")
```

### 工作原理

1. **暂停**：爬取过程中按 `Ctrl+C`。spider 会等待所有进行中的请求完成，保存检查点（待处理请求 + 已见请求指纹集合），然后退出。
2. **强制停止**：再次按 `Ctrl+C` 可立即停止，不等待活跃任务。
3. **恢复**：使用相同 `crawldir` 再次运行 spider。它会检测检查点，恢复队列与已见集合，从中断处继续，并跳过 `start_requests()`。
4. **清理**：爬取正常完成（非暂停）时，检查点文件会自动删除。

**爬取过程中也会定期保存检查点（默认每 5 分钟）。**

可按如下方式修改间隔：

```python
# Save checkpoint every 2 minutes
spider = MySpider(crawldir="crawl_data/my_spider", interval=120.0)
```

磁盘写入为原子操作，因此完全安全。

**提示：** 爬取过程中按 `Ctrl+C` 总会让 spider 优雅关闭，即使未启用检查点系统。不等待再次按键则会强制立即关闭。

### 判断是否正在恢复

`on_start()` 钩子会收到 `resuming` 标志：

```python
async def on_start(self, resuming: bool = False):
    if resuming:
        self.logger.info("Resuming from checkpoint!")
    else:
        self.logger.info("Starting fresh crawl")
```

## 开发模式

迭代 spider 的 `parse()` 逻辑时，每次运行都重新请求目标服务器既慢又嘈杂。开发模式在首次运行时将每个响应缓存到磁盘，后续运行从磁盘回放，这样你可以反复调整选择器并重新运行 spider，而无需任何网络请求。

在 spider 上设置 `development_mode = True` 即可启用：

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    development_mode = True

    async def parse(self, response: Response):
        yield {"title": response.css("title::text").get("")}
```

首次运行正常抓取并将每个响应存盘。之后每次运行从缓存提供相同请求，完全跳过网络。

### 缓存位置

默认情况下，响应缓存在当前工作目录（运行 spider 的位置，**而非** spider 脚本所在位置）下的 `.scrapling_cache/{spider.name}/`。可用 `development_cache_dir` 覆盖：

```python
class MySpider(Spider):
    name = "my_spider"
    start_urls = ["https://example.com"]
    development_mode = True
    development_cache_dir = "/tmp/my_spider_cache"
```

### 工作原理

1. **缓存键**：每个响应以请求指纹为键，因此任何影响指纹的属性（`fp_include_kwargs`、`fp_include_headers`、`fp_keep_fragments`）变更都会触发全新抓取。
2. **存储格式**：每个响应一个 JSON 文件，命名为 `{fingerprint_hex}.json`。正文 base64 编码以精确保留二进制内容。写入为原子操作（临时文件 + 重命名）。
3. **回放**：缓存命中时，引擎完全跳过网络，包括 `download_delay`、速率限制和 `is_blocked()` 重试路径。缓存响应直接进入你的回调。
4. **统计**：缓存请求仍计入 `requests_count`、`response_bytes` 及各状态码计数，统计输出与正常爬取一致。另有 `cache_hits` 和 `cache_misses` 两个计数器反映缓存表现。

### 清除缓存

没有自动过期。要强制全新爬取，删除缓存目录或直接调用 manager 的 `clear()` 方法。

**警告：** 开发模式仅供开发，不用于生产。缓存响应永不过期，回放会绕过速率限制与被拦截请求重试。不要将 `development_mode = True` 的 spider 用于上线。

## 流式输出

对于长时间运行的 spider 或需要实时访问抓取条目的应用，使用 `stream()` 方法而非 `start()`：

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
- 条目在抓取时逐个 yield，而非收集到列表中
- 迭代过程中可通过 `spider.stats` 获取实时统计

**注意：** 可通过 `spider.stats` 访问的完整统计列表见下文[此处](#results--statistics)。

也可与检查点系统配合使用，便于在 spider 之上构建 UI——具备实时数据且可暂停/恢复。

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
在上述代码中也可使用 `spider.pause()` 关闭 spider。若未启用检查点系统，则仅会结束爬取。

## 生命周期钩子

spider 提供多个可在爬取不同阶段自定义行为的钩子：

### on_start

爬取开始前调用。用于加载数据、初始化资源等准备工作：

```python
async def on_start(self, resuming: bool = False):
    self.logger.info("Spider starting up")
    # Load seed URLs from a database, initialize counters, etc.
```

### on_close

爬取结束后调用（无论完成或暂停）。用于清理：

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

每个抓取条目加入结果前调用。返回条目（可修改）以保留，返回 `None` 则丢弃：

```python
async def on_scraped_item(self, item: dict) -> dict | None:
    # Drop items without a title
    if not item.get("title"):
        return None

    # Modify items (e.g., add timestamps)
    item["scraped_at"] = "2026-01-01"
    return item
```

**提示：** 此钩子也可用于将条目送入自定义管道并从 spider 结果中丢弃。

### start_requests

重写 `start_requests()` 以自定义初始请求生成，替代 `start_urls`：

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

`start()` 返回的 `CrawlResult` 包含抓取条目和详细统计：

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

spider 内置可通过 `self.logger` 访问的日志器。已按 spider 名称预配置，并支持多种自定义选项：

| Attribute             | Default                                                      | Description                                        |
|-----------------------|--------------------------------------------------------------|----------------------------------------------------|
| `logging_level`       | `logging.DEBUG`                                              | Minimum log level                                  |
| `logging_format`      | `"[%(asctime)s]:({spider_name}) %(levelname)s: %(message)s"` | Log message format                                 |
| `logging_date_format` | `"%Y-%m-%d %H:%M:%S"`                                        | Date format in log messages                        |
| `log_file`            | `None`                                                       | Path to a log file (in addition to console output) |

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

若日志文件目录不存在会自动创建。控制台与文件输出使用相同格式。
