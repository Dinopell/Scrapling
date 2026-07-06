# Scrapling MCP 服务器指南

<iframe width="560" height="315" src="https://www.youtube.com/embed/qyFk3ZNwOxE?si=3FHzgcYCb66iJ6e3" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

**Scrapling MCP 服务器**是一项新功能，将 Scrapling 强大的 Web 抓取能力直接带入你喜爱的 AI 聊天机器人或 AI 智能体。该集成让你可通过 Claude 的 AI 界面或任何支持 MCP 的界面，以对话方式进行网站抓取、数据提取并绕过反机器人防护。

## 功能

Scrapling MCP 服务器提供十个强大的 Web 抓取工具：

### 🚀 基础 HTTP 抓取
- **`get`**：快速 HTTP 请求，支持浏览器指纹伪装，生成与 TLS 版本、HTTP/3 等匹配的真实浏览器请求头！
- **`bulk_get`**：上述工具的异步版本，可同时抓取多个 URL！

### 🌐 动态内容抓取
- **`fetch`**：使用 Chromium/Chrome 浏览器快速获取动态内容，对请求/浏览器有完全控制，等等！
- **`bulk_fetch`**：上述工具的异步版本，可在不同浏览器标签页中同时抓取多个 URL！

### 🔒 隐蔽抓取
- **`stealthy_fetch`**：使用我们的隐蔽浏览器绕过 Cloudflare Turnstile/Interstitial 及其他反机器人系统，对请求/浏览器有完全控制！
- **`bulk_stealthy_fetch`**：上述工具的异步版本，可在不同浏览器标签页中同时隐蔽抓取多个 URL！

### 📸 截图
- **`screenshot`**：使用已打开的浏览器会话捕获 PNG 或 JPEG 截图，以模型真正能看到的图像内容块返回（而非 base64 字符串）。支持全页捕获、JPEG 质量及常用就绪控制（`wait`、`wait_selector`、`network_idle`）。

### 🔌 会话管理
- **`open_session`**：创建持久浏览器会话（动态或隐蔽），在多次 fetch 调用间保持打开，避免每次启动新浏览器的开销。
- **`close_session`**：关闭持久浏览器会话并释放资源。
- **`list_sessions`**：列出所有活动浏览器会话及其详情。

### 核心能力
- **智能内容提取**：将网页/元素转换为 Markdown、HTML，或提取干净文本内容
- **CSS 选择器支持**：在将内容交给 AI 之前，使用 Scrapling 引擎精确瞄准特定元素
- **反机器人绕过**：处理 Cloudflare Turnstile、Interstitial 及其他防护
- **代理支持**：使用代理实现匿名与地域定向
- **浏览器伪装**：通过 TLS 指纹识别、匹配该版本的真实浏览器请求头等模拟真实浏览器
- **并行处理**：并发抓取多个 URL 提高效率
- **会话持久化**：在多次请求间复用浏览器会话以提升性能
- **广告拦截**：所有基于浏览器的工具自动拦截约 3,500 个已知广告与追踪域名的请求，节省 Token 并加快页面加载
- **提示注入防护**：自动清理可用于提示注入攻击的隐藏内容（CSS 隐藏元素、aria-hidden、零宽字符、HTML 注释、template 标签）

#### 但为何使用 Scrapling MCP 服务器而非其他可用工具？

除隐蔽能力与绕过 Cloudflare Turnstile/Interstitial 外，Scrapling 的服务器是唯一让你选择特定元素再传给 AI 的服务器，可大幅节省时间与 Token！

其他服务器的工作方式是：提取内容后全部传给 AI 以提取你想要的字段。这导致 AI 消耗远超所需的 Token（来自无关内容）。Scrapling 允许你在将内容传给 AI 之前传入 CSS 选择器以缩小范围，使整个流程更快更高效。

若你不会编写/使用 CSS 选择器，别担心。你可以在提示中让 AI 编写选择器以匹配可能的字段，并观看它尝试不同组合直到找到正确的一个，我们将在示例部分展示。

## 安装

安装带 MCP 支持的 Scrapling，然后再次确认浏览器依赖已安装。

```bash
# 安装带 MCP 服务器依赖的 Scrapling
pip install "scrapling[ai]"

# 安装浏览器依赖
scrapling install
```

或直接从 Docker 镜像仓库使用镜像：
```bash
docker pull pyd4vinci/scrapling
```
或从 GitHub 镜像仓库下载：
```bash
docker pull ghcr.io/d4vinci/scrapling:latest
```

## 配置 MCP 服务器

此处说明如何将 Scrapling MCP 服务器添加到 [Claude Desktop](https://claude.ai/download) 与 [Claude Code](https://www.anthropic.com/claude-code)，但相同逻辑适用于任何其他支持 MCP 的聊天机器人：

### Claude Desktop

1. 打开 Claude Desktop
2. 点击左上角汉堡菜单（☰）→ Settings → Developer → Edit Config
3. 添加 Scrapling MCP 服务器配置：
```json
"ScraplingServer": {
  "command": "scrapling",
  "args": [
    "mcp"
  ]
}
```
若这是你添加的第一个 MCP 服务器，将文件内容设为：
```json
{
  "mcpServers": {
    "ScraplingServer": {
      "command": "scrapling",
      "args": [
        "mcp"
      ]
    }
  }
}
```
根据[官方文章](https://modelcontextprotocol.io/quickstart/user)，此操作会在不存在时创建新配置文件，或打开现有配置。文件位于

1. **MacOS**：`~/Library/Application Support/Claude/claude_desktop_config.json`
2. **Windows**：`%APPDATA%\Claude\claude_desktop_config.json`

为确保正常工作，请使用 `scrapling` 可执行文件的完整路径。打开终端执行：

1. **MacOS**：`which scrapling`
2. **Windows**：`where scrapling`

例如，在我的 Mac 上返回 `/Users/<MyUsername>/.venv/bin/scrapling`，因此最终使用的配置为：
```json
{
  "mcpServers": {
    "ScraplingServer": {
      "command": "/Users/<MyUsername>/.venv/bin/scrapling",
      "args": [
        "mcp"
      ]
    }
  }
}
```
#### Docker
若使用 Docker 镜像，配置类似
```json
{
  "mcpServers": {
    "ScraplingServer": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm", "pyd4vinci/scrapling", "mcp"
      ]
    }
  }
}
```

相同逻辑适用于 [Cursor](https://cursor.com/docs/context/mcp)、[WindSurf](https://windsurf.com/university/tutorials/configuring-first-mcp-server) 等。

### Claude Code
此处配置更简单。若已安装 [Claude Code](https://www.anthropic.com/claude-code)，打开终端执行：

```bash
claude mcp add ScraplingServer "/Users/<MyUsername>/.venv/bin/scrapling" mcp
```
同上，获取 Scrapling 可执行文件路径，打开终端执行：

1. **MacOS**：`which scrapling`
2. **Windows**：`where scrapling`

Anthropic 关于[如何将 MCP 服务器添加到 Claude Code](https://docs.anthropic.com/en/docs/claude-code/mcp#option-1%3A-add-a-local-stdio-server) 的主文章有更多细节。


然后，添加服务器后，需要完全退出并重启你使用的应用。在 Claude Desktop 中，你应在聊天输入框右下角看到 MCP 服务器指示器（🔧），或在聊天输入框的 `Search and tools` 下拉菜单中看到 `ScraplingServer`。

### Streamable HTTP
自 0.3.6 版本起，我们增加了让 MCP 服务器使用「Streamable HTTP」传输模式而非传统「stdio」传输的能力。

因此，不使用以下命令（「stdio」方式）：
```bash
scrapling mcp
```
使用以下命令启用「Streamable HTTP」传输模式：
```bash
scrapling mcp --http
```
因此，服务器监听的默认主机为「0.0.0.0」，端口为 8000，均可如下配置：
```bash
scrapling mcp --http --host '127.0.0.1' --port 8000
```

## 示例

下面展示我们测试 MCP 服务器时使用的一些提示示例，但你可能比我们更有创意，提示工程也更好 :)

我们将从简单提示逐步过渡到更复杂的提示。示例使用 Claude Desktop，但相同逻辑当然适用于其他工具。

1. **基础 Web 抓取**

    将网页主内容提取为 Markdown：
    
    ```
    Scrape the main content from https://example.com and convert it to markdown format.
    ```
    
    Claude 将使用 `get` 工具获取页面并返回干净可读内容。若失败，除非另有指示，否则将每秒重试一次，共 3 次。若因防护或动态网站等原因无法获取内容，将自动尝试其他工具。若 Claude 未自动这样做，可在提示中加入该要求。
    
    同一提示的优化版本：
    ```
    Use regular requests to scrape the main content from https://example.com and convert it to markdown format.
    ```
    这告诉 Claude 在此使用哪个工具，无需猜测。有时它会自行使用普通请求，有时又会无理由假设浏览器更适合该网站。经验法则：应始终告诉 Claude 使用哪个工具，以节省时间、金钱并获得一致结果。

2. **定向数据提取**

    使用 CSS 选择器提取特定元素：
    
    ```
    Get all product titles from https://shop.example.com using the CSS selector '.product-title'. If the request fails, retry up to 5 times every 10 seconds.
    ```
    
    服务器将仅提取匹配选择器的元素，并以结构化列表返回。注意我设置了工具在网站连接问题时最多重试 5 次，但默认设置对大多数情况已足够。

3. **电商数据采集**

    另一个稍复杂的提示示例：
    ```
    Extract product information from these e-commerce URLs using bulk browser fetches:
    - https://shop1.com/product-a
    - https://shop2.com/product-b  
    - https://shop3.com/product-c
    
    Get the product names, prices, and descriptions from each page.
    ```
    
    Claude 将使用 `bulk_fetch` 并发抓取所有 URL，然后分析提取的数据。

4. **更高级的工作流**

    假设我想获取 PlayStation 商店首页当前所有动作游戏的 URL，可使用如下提示：
    ```
    Extract the URLs of all games in this page, then do a bulk request to them and return a list of all action games: https://store.playstation.com/en-us/pages/browse
    ```
    注意我指示它对收集的所有 URL 使用批量请求。若未提及，有时按预期工作，有时会对每个 URL 单独请求，耗时显著更长。此提示约需一分钟完成。
    
    但因我不够具体，它在此使用了 `stealthy_fetch`，第二步使用了 `bulk_stealthy_fetch`，不必要地消耗了大量 Token。更好的提示是：
    ```
    Use normal requests to extract the URLs of all games in this page, then do a bulk request to them and return a list of all action games: https://store.playstation.com/en-us/pages/browse
    ```
    若你会写 CSS 选择器，可指示 Claude 将选择器应用于你想要的元素，任务将几乎立即完成。
    ```
    Use normal requests to extract the URLs of all games on the page below, then perform a bulk request to them and return a list of all action games.
    The selector for games in the first page is `[href*="/concept/"]` and the selector for the genre in the second request is `[data-qa="gameInfo#releaseInformation#genre-value"]`.
    
    URL: https://store.playstation.com/en-us/pages/browse
    ```

5. **从有 Cloudflare 防护的网站获取数据**

    若你认为目标网站有 Cloudflare 防护，应直接告诉 Claude，而非让它自行发现。
    ```
    What's the price of this product? Be cautious, as it utilizes Cloudflare's Turnstile protection. Make the browser visible while you work.

    https://ao.com/product/oo101uk-ninja-woodfire-outdoor-pizza-oven-brown-99357-685.aspx
    ```

6. **长工作流**

    例如可使用如下提示：
    ```
    Extract all product URLs for the following category, then return the prices and details for the first 3 products.
    
    https://www.arnotts.ie/furniture/bedroom/bed-frames/
    ```
    但更好的提示是：
    ```
    Go to the following category URL and extract all product URLs using the CSS selector "a". Then, fetch the first 3 product pages in parallel and extract each product's price and details.
    
    Keep the output in markdown format to reduce irrelevant content.
    
    Category URL:
    https://www.arnotts.ie/furniture/bedroom/bed-frames/
    ```

7. **使用持久会话**

    从同一网站抓取多页时，使用持久浏览器会话以避免每次请求启动新浏览器的开销：
    ```
    Open a stealthy browser session with 5 pages maximum pool, then use it to scrape the main details in bulk from the first 5 product pages on https://shop.example.com. Close the session when you're done.
    ```
    Claude 将使用 `open_session` 创建持久浏览器，在同时打开所有页面时将 `session_id` 传给 `bulk_stealthy_fetch` 调用，最后调用 `close_session`。这比为每页启动新浏览器快得多。

    !!! danger
    
        使用持久会话时，完成后务必关闭会话，否则会一直保持打开！


8. **在长流程中使用持久会话**

    另一个让 Claude 思考的长测试示例：

    ```
    Use Scrapling MCP to do the following in this order:

    1. Open a stealthy browser session with headless mode off.
    2. Go to this page and collect the number of stars: https://github.com/D4Vinci/Scrapling
    3. From the README, get the URL that shows the number of downloads and go to it.
    4. Get the number of downloads and the top 3 countries from the graph.
    5. Prepare a report with the results.
    6. Close the browser.
    ```

以此类推，你懂的。创造力是关键。

## 最佳实践

以下是一些技术建议。

### 1. 选择正确的工具
- **`get`**：快速、简单网站
- **`fetch`**：有 JavaScript/动态内容的网站
- **`stealthy_fetch`**：有防护的网站、Cloudflare、反机器人系统

### 2. 优化性能
- 对多个 URL 使用批量工具
- 禁用不必要的资源
- 设置合适的超时
- 使用 CSS 选择器进行定向提取

### 3. 处理动态内容
- 对 SPA 使用 `network_idle`
- 对特定元素设置 `wait_selector`
- 对加载慢的网站增加超时

### 4. 数据质量
- 使用 `main_content_only=true` 避免导航/广告
- 根据用例选择合适的 `extraction_type`

### 5. 提示注入防护
启用 `main_content_only`（默认）时，MCP 服务器会自动清理抓取内容。这会剥离恶意网站可能用于向 AI 上下文注入指令的隐藏内容：

- **CSS 隐藏元素**：`display:none`、`visibility:hidden`、`opacity:0`、`font-size:0`、`height:0`、`width:0`
- **无障碍隐藏元素**：`aria-hidden="true"`
- **Template 标签**：`<template>` 元素
- **HTML 注释**：`<!-- ... -->`
- **零宽字符**：如零宽空格等不可见 Unicode 字符

此防护在所有 MCP 工具响应上自动运行。为最大保护，请保持 `main_content_only=true`（默认）。

### 6. 对多次请求使用会话
- 抓取多页时使用 `open_session` 创建持久浏览器会话
- 将 `session_id` 传给 `fetch` 或 `stealthy_fetch` 调用以复用同一浏览器
- 完成后始终用 `close_session` 关闭会话以释放资源
- 使用 `list_sessions` 查看哪些会话仍活动
- 动态会话的 `session_id` 只能用于 `fetch`/`bulk_fetch`，隐蔽会话只能用于 `stealthy_fetch`/`bulk_stealthy_fetch`
- 向 `open_session` 传入自定义 `session_id` 为会话赋予有意义名称（如 `"search"`、`"checkout"`），而非随机十六进制默认值。若所选 ID 已被使用，`open_session` 会抛出异常，便于提前发现冲突

### 7. 捕获截图
- `screenshot` 仅通过现有浏览器会话工作，因此需先调用 `open_session`（`dynamic` 或 `stealthy` 均可）
- 图像以真正的 `ImageContent` 块返回，而非 JSON 中的 base64 字符串，模型可直接看到页面
- 需要首屏以下全部内容时使用 `full_page=True`；默认仅捕获可见视口
- 当不需要像素级完美颜色时，选择 `image_type="jpeg"` 并设置 `quality`（0-100）以减小负载
- `fetch` 使用的相同 `wait`、`wait_selector`、`network_idle` 与 `timeout` 控制在此也可用

## 法律与伦理考量

⚠️ **重要准则：**

- **检查 robots.txt**：访问 `https://website.com/robots.txt` 查看抓取规则
- **遵守速率限制**：不要用请求压垮服务器
- **服务条款**：阅读并遵守网站条款
- **版权**：尊重知识产权
- **隐私**：注意个人数据保护法律
- **商业用途**：确保获得业务用途许可

---

*由 Scrapling 团队用 ❤️ 打造。祝抓取愉快！*
