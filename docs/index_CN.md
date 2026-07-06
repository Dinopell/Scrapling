<style>
.md-typeset h1 {
  display: none;
}
[data-md-color-scheme="default"] .only-dark { display: none; }
[data-md-color-scheme="slate"] .only-light { display: none; }
</style>

<br/>
<div align="center">
    <a href="https://scrapling.readthedocs.io/en/latest/" alt="poster">
        <img alt="Scrapling" src="assets/cover_light.svg" class="only-light">
        <img alt="Scrapling" src="assets/cover_dark.svg" class="only-dark">
    </a>
</div>

<h2 align="center"><i>面向现代 Web 的轻松爬虫框架</i></h2><br>

Scrapling 是一个自适应 Web 爬虫框架，可处理从单次请求到大规模爬取的一切需求。

其解析器能从网站变更中学习，并在页面更新时自动重新定位你的元素。其 Fetcher 可开箱即用地绕过 Cloudflare Turnstile 等反机器人系统。其 Spider 框架让你用几行 Python 代码即可扩展到并发、多会话爬取，并支持暂停/恢复与自动代理轮换——一个库，零妥协。

极速爬取，实时统计与流式输出。由爬虫开发者打造，面向爬虫开发者与普通用户，每个人都能找到适合自己的功能。

```python
from scrapling.fetchers import Fetcher, StealthyFetcher, DynamicFetcher
StealthyFetcher.adaptive = True
page = StealthyFetcher.fetch('https://example.com', headless=True, network_idle=True)  # 低调获取网站！
products = page.css('.product', auto_save=True)                                        # 抓取能经受网站改版的数据！
products = page.css('.product', adaptive=True)                                         # 之后若网站结构变化，传入 `adaptive=True` 即可重新定位！
```
或扩展到完整爬取
```python
from scrapling.spiders import Spider, Response

class MySpider(Spider):
  name = "demo"
  start_urls = ["https://example.com/"]

  async def parse(self, response: Response):
      for item in response.css('.product'):
          yield {"title": item.css('h2::text').get()}

MySpider().start()
```

## 顶级赞助商

<style>
.ad {
    width:240px;
    height:100px;
}

</style>

<!-- sponsors -->
<div style="text-align: center;">
  <a href="https://proxidize.com/?utm_source=github&utm_medium=sponsorship&utm_campaign=scrapling&utm_content=d4vinci" target="_blank" title="Clean Proxies with No Nonsense.">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/proxidize.png" class="ad">
  </a>
  <a href="https://coldproxy.com/" target="_blank" title="Residential, IPv6 & Datacenter Proxies for Web Scraping">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/coldproxy.png" class="ad">
  </a>
  <a href="https://hypersolutions.co/?utm_source=github&utm_medium=readme&utm_campaign=scrapling" target="_blank" title="Bot Protection Bypass API for Akamai, DataDome, Incapsula & Kasada">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/HyperSolutions.png" class="ad">
  </a>
  <a href="https://birdproxies.com/t/scrapling" target="_blank" title="At Bird Proxies, we eliminate your pains such as banned IPs, geo restriction, and high costs so you can focus on your work.">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/BirdProxies.jpg" class="ad">
  </a>
  <a href="https://evomi.com?utm_source=github&utm_medium=banner&utm_campaign=d4vinci-scrapling" target="_blank" title="Evomi is your Swiss Quality Proxy Provider, starting at $0.49/GB">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/evomi.png" class="ad">
  </a>
  <a href="https://tikhub.io/?utm_source=github.com/D4Vinci/Scrapling&utm_medium=marketing_social&utm_campaign=retargeting&utm_content=carousel_ad" target="_blank" title="Unlock the Power of Social Media Data & AI">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/TikHub.jpg" class="ad">
  </a>
  <a href="https://petrosky.io/d4vinci" target="_blank" title="PetroSky delivers cutting-edge VPS hosting.">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/petrosky.png" class="ad">
  </a>
  <a href="https://substack.thewebscraping.club/p/scrapling-hands-on-guide?utm_source=github&utm_medium=repo&utm_campaign=scrapling" target="_blank" title="The #1 newsletter dedicated to Web Scraping">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/TWSC.png" class="ad">
  </a>
  <a href="https://www.swiftproxy.net/?ref=D4Vinci" target="_blank" title="Scalable Solutions for Web Data Access">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/SwiftProxy.png" class="ad">
  </a>
  <a href="https://9proxy.com/pricing?tab=traffic&utm_source=Github&utm_campaign=D4vinci" target="_blank" title="Top-Tier Residential Proxy Solution for the Highest Success Rate">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/9proxy.jpg" class="ad">
  </a>
  <a href="https://go.nodemaven.com/scraplingjune" target="_blank" title="Proxies with the Highest IP Scores">
    <img src="https://raw.githubusercontent.com/D4Vinci/Scrapling/main/images/NodeMaven.svg" class="ad">
  </a>
  <br />
  <br />
</div>
<!-- /sponsors -->

<i><sub>想在这里展示你的广告？点击[这里](https://github.com/sponsors/D4Vinci)，选择方案，并享受其余权益！</sub></i>

## 核心特性

### Spiders — 完整爬取框架
- 🕷️ **类 Scrapy 的 Spider API**：通过 `start_urls`、异步 `parse` 回调以及 `Request`/`Response` 对象定义 Spider。
- ⚡ **并发爬取**：可配置并发上限、按域名限流与下载延迟。
- 🔄 **多会话支持**：在单个 Spider 中统一使用 HTTP 请求与隐蔽无头浏览器——按 ID 将请求路由到不同会话。
- 💾 **暂停与恢复**：基于检查点的爬取持久化。按 Ctrl+C 优雅关闭；重启后从断点继续。
- 📡 **流式模式**：通过 `async for item in spider.stream()` 实时流式输出抓取项与统计——适合 UI、管道与长时间爬取。
- 🛡️ **被拦截请求检测**：自动检测并重试被拦截的请求，逻辑可自定义。
- 🤖 **robots.txt 合规**：可选 `robots_txt_obey` 标志，遵守 `Disallow`、`Crawl-delay` 与 `Request-rate` 指令，并按域名缓存。
- 🧪 **开发模式**：首次运行将响应缓存到磁盘，后续运行回放——无需重复请求目标服务器即可迭代 `parse()` 逻辑。
- 📦 **内置导出**：通过钩子与自定义管道，或使用内置 JSON/JSONL 的 `result.items.to_json()` / `result.items.to_jsonl()` 导出结果。

### 高级网站获取与会话支持
- **HTTP 请求**：通过 `Fetcher` 类进行快速且隐蔽的 HTTP 请求。可模拟浏览器 TLS 指纹与请求头，并支持 HTTP/3。
- **动态加载**：通过 `DynamicFetcher` 类以完整浏览器自动化获取动态网站，支持 Playwright 的 Chromium 与 Google Chrome。
- **反机器人绕过**：`StealthyFetcher` 具备高级隐蔽能力与指纹伪装，可轻松通过各类 Cloudflare Turnstile/Interstitial 自动化防护。
- **会话管理**：`FetcherSession`、`StealthySession`、`DynamicSession` 提供跨请求持久会话，管理 Cookie 与状态。
- **代理轮换**：内置 `ProxyRotator`，支持循环或自定义轮换策略，适用于所有会话类型，并支持按请求覆盖代理。
- **域名与广告拦截**：在基于浏览器的 Fetcher 中拦截指定域名（及其子域名）请求，或启用内置广告拦截（约 3,500 个已知广告/追踪域名）。
- **DNS 泄漏防护**：可选 DNS-over-HTTPS，通过 Cloudflare DoH 路由 DNS 查询，在使用代理时防止 DNS 泄漏。
- **异步支持**：所有 Fetcher 与专用异步会话类均完整支持异步。

### 自适应抓取与 AI 集成
- 🔄 **智能元素追踪**：网站变更后，通过智能相似度算法重新定位元素。
- 🎯 **灵活智能选择**：CSS 选择器、XPath 选择器、基于过滤器的搜索、文本搜索、正则搜索等。
- 🔍 **查找相似元素**：自动定位与已找到元素相似的其他元素。
- 🤖 **供 AI 使用的 MCP 服务器**：内置 MCP 服务器，用于 AI 辅助 Web 抓取与数据提取。该 MCP 服务器具备强大自定义能力，在将内容交给 AI（Claude/Cursor 等）之前，先利用 Scrapling 提取目标内容，从而加快操作并减少 Token 消耗。（[演示视频](https://www.youtube.com/watch?v=qyFk3ZNwOxE)）

### 高性能且久经考验的架构
- 🚀 **极速**：优化性能，超越多数 Python 爬虫库。
- 🔋 **内存高效**：优化数据结构与惰性加载，内存占用极小。
- ⚡ **快速 JSON 序列化**：比标准库快约 10 倍。
- 🏗️ **久经考验**：不仅拥有 92% 测试覆盖率与完整类型提示，过去一年还被数百名爬虫开发者日常使用。

### 对开发者/爬虫工程师友好的体验
- 🎯 **交互式 Web 抓取 Shell**：可选内置 IPython Shell，集成 Scrapling、快捷方式与新工具，加速脚本开发，例如将 curl 请求转换为 Scrapling 请求，并在浏览器中查看请求结果。
- 🚀 **直接在终端使用**：可选地，无需写一行代码即可用 Scrapling 抓取 URL！
- 🛠️ **丰富的导航 API**：支持父级、兄弟与子女节点遍历的高级 DOM 导航。
- 🧬 **增强文本处理**：内置正则、清洗方法与优化的字符串操作。
- 📝 **自动生成选择器**：为任意元素生成稳健的 CSS/XPath 选择器。
- 🔌 **熟悉的 API**：类似 Scrapy/BeautifulSoup，使用与 Scrapy/Parsel 相同的伪元素。
- 📘 **完整类型覆盖**：完整类型提示，IDE 支持与代码补全出色。每次变更都会用 **PyRight** 与 **MyPy** 自动扫描整个代码库。
- 🔋 **现成 Docker 镜像**：每次发布都会自动构建并推送包含所有浏览器的 Docker 镜像。


## Star 历史
Scrapling 自发布以来 GitHub Star 稳步增长（见下图）。

<div id="chartContainer">
  <a href="https://github.com/D4Vinci/Scrapling">
    <img id="chartImage" alt="Star History Chart" loading="lazy" src="https://api.star-history.com/svg?repos=D4Vinci/Scrapling&type=Date" height="400"/>
  </a>
</div>

<script>
const observer = new MutationObserver((mutations) => {
  mutations.forEach((mutation) => {
    if (mutation.attributeName === 'data-md-color-media') {
      const colorMedia = document.body.getAttribute('data-md-color-media');
      const isDarkScheme = document.body.getAttribute('data-md-color-scheme') === 'slate';
      const chartImg = document.querySelector('#chartImage');
      const baseUrl = 'https://api.star-history.com/svg?repos=D4Vinci/Scrapling&type=Date';
      
      if (colorMedia === '(prefers-color-scheme)' ? isDarkScheme : colorMedia.includes('dark')) {
        chartImg.src = `${baseUrl}&theme=dark`;
      } else {
        chartImg.src = baseUrl;
      }
    }
  });
});

observer.observe(document.body, {
  attributes: true,
  attributeFilter: ['data-md-color-media', 'data-md-color-scheme']
});
</script>


## 安装
Scrapling 需要 Python 3.10 或更高版本：

```bash
pip install scrapling
```

!!! warning

    此安装仅包含解析引擎及其依赖，不含任何 Fetcher 或命令行依赖。因此，若仅安装此项，从 `scrapling.fetchers` 或 `scrapling.spiders` 导入（如上文示例）会引发 `ModuleNotFoundError`。若需使用 Fetcher 或 Spider，请先按下方说明安装 Fetcher 依赖。

### 可选依赖

1. 若需使用下方任一额外功能、Fetcher 或其类，需先安装 Fetcher 依赖与浏览器依赖：
    ```bash
    pip install "scrapling[fetchers]"
    
    scrapling install           # 常规安装
    scrapling install  --force  # 强制重装
    ```

    这将下载所有浏览器及其系统依赖与指纹操控依赖。

    也可在代码中安装，而非运行命令：
    ```python
    from scrapling.cli import install
    
    install([], standalone_mode=False)          # 常规安装
    install(["--force"], standalone_mode=False) # 强制重装
    ```

2. 额外功能：


     - 安装 MCP 服务器功能：
       ```bash
       pip install "scrapling[ai]"
       ```
     - 安装 Shell 功能（Web 抓取 Shell 与 `extract` 命令）：
         ```bash
         pip install "scrapling[shell]"
         ```
     - 安装全部：
         ```bash
         pip install "scrapling[all]"
         ```
     安装任一额外组件后，若尚未安装浏览器依赖，请记得运行 `scrapling install`

### Docker
也可从 DockerHub 拉取包含所有额外组件与浏览器的镜像：
```bash
docker pull pyd4vinci/scrapling
```
或从 GitHub 镜像仓库拉取：
```bash
docker pull ghcr.io/d4vinci/scrapling:latest
```
该镜像由 GitHub Actions 基于仓库 main 分支自动构建并推送。

## 文档组织方式
Scrapling 文档较为详尽，我们尽量遵循 [Diátaxis 文档框架](https://diataxis.fr/)。

## 支持

如果你喜欢 Scrapling 并希望支持其发展：

- ⭐ 为 [GitHub 仓库](https://github.com/D4Vinci/Scrapling) 点 Star
- 🚀 关注 [Twitter](https://x.com/Scrapling_dev) 并加入 [Discord 服务器](https://discord.gg/EMgGbDceNQ)
- 💝 考虑[赞助项目或请我喝咖啡](donate_CN.md) :wink:
- 🐛 通过 [GitHub Issues](https://github.com/D4Vinci/Scrapling/issues) 报告 Bug 与建议功能

## 许可证

本项目采用 BSD-3 许可证。详见 [LICENSE](https://github.com/D4Vinci/Scrapling/blob/main/LICENSE) 文件。
