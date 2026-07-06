# Scrapling Extract 命令指南

**在终端中无需编程即可进行 Web 抓取！**

`scrapling extract` 命令让你无需编写任何代码，即可在终端中直接下载并提取网站内容。适合初学者、研究人员以及需要快速 Web 数据提取的任何人。

!!! success "前置条件"

    1. 你已完成或阅读 [Fetcher 基础](../fetching/choosing_CN.md) 页面，了解 [Response 对象](../fetching/choosing_CN.md#response-object) 是什么以及应使用哪个 Fetcher。
    2. 你已完成或阅读 [查询元素](../parsing/selection_CN.md) 页面，了解如何从 [Selector](../parsing/main_classes_CN.md#selector)/[Response](../fetching/choosing_CN.md#response-object) 对象中查找/提取元素。
    3. 你已完成或阅读 [主要类](../parsing/main_classes_CN.md) 页面，了解 [Response](../fetching/choosing_CN.md#response-object) 类从 [Selector](../parsing/main_classes_CN.md#selector) 类继承了哪些属性/方法。
    4. 你已完成或阅读 Fetcher 章节中至少一页，以便在此发起请求：[HTTP 请求](../fetching/static_CN.md)、[动态网站](../fetching/dynamic_CN.md) 或 [带强防护的动态网站](../fetching/stealthy_CN.md)。


## 什么是 Extract 命令组？

extract 命令是一组简单的终端工具，可：

- **下载网页**并将其内容保存到文件。
- **将 HTML 转换为可读格式**，如 Markdown，保持 HTML，或仅提取页面文本内容。
- **支持自定义 CSS 选择器**以提取页面特定部分。
- **处理 HTTP 请求与浏览器获取**
- **高度可定制**，支持自定义请求头、Cookie、代理及其余选项。代码中几乎所有可用选项在命令行中也可访问。

!!! tip "面向 AI 的模式"

    所有 extract 命令均支持 `--ai-targeted` 标志。启用后，仅提取正文主内容，剥离噪声标签（script、style、noscript、svg），移除可用于提示注入的隐藏元素（CSS 隐藏、aria-hidden、template 标签），剥离零宽 Unicode 字符，并移除 HTML 注释。对于浏览器命令（`fetch`/`stealthy-fetch`），还会自动启用广告拦截。当输出将交给 AI 模型时，此模式非常理想。

## 快速开始

- **基本网站下载**

    将网站文本内容下载为干净可读文本：
    ```bash
    scrapling extract get "https://example.com" page_content.txt
    ```
    这将发起 HTTP GET 请求，并将网页文本内容保存到 `page_content.txt`。

- **保存为不同格式**

    通过更改文件扩展名选择输出格式：
    ```bash
    # 将 HTML 内容转换为 Markdown 后保存（适合文档）
    scrapling extract get "https://blog.example.com" article.md
    
    # 将 HTML 内容原样保存
    scrapling extract get "https://example.com" page.html
    
    # 将网页的干净文本内容保存到文件
    scrapling extract get "https://example.com" content.txt
  
    # 或使用 Docker 镜像，例如：
    docker run -v $(pwd)/output:/output pyd4vinci/scrapling extract get "https://blog.example.com" /output/article.md 
    ```

- **提取特定内容**

    所有命令均可通过 `--css-selector` 或下文示例中的 `-s` 使用 CSS 选择器提取页面特定部分。

## 可用命令

可通过 `scrapling extract --help` 显示可用命令，得到如下列表：
```bash
Usage: scrapling extract [OPTIONS] COMMAND [ARGS]...

  Fetch web pages using various fetchers and extract full/selected HTML content as HTML, Markdown, or extract text content.

Options:
  --help  Show this message and exit.

Commands:
  get             Perform a GET request and save the content to a file.
  post            Perform a POST request and save the content to a file.
  put             Perform a PUT request and save the content to a file.
  delete          Perform a DELETE request and save the content to a file.
  fetch           Use DynamicFetcher to fetch content with browser...
  stealthy-fetch  Use StealthyFetcher to fetch content with advanced...
```

下文将逐一详细介绍各命令。

### HTTP 请求

1. **GET 请求**

    下载网站内容最常用的命令：
    
    ```bash
    scrapling extract get [URL] [OUTPUT_FILE] [OPTIONS]
    ```
    
    **示例：**
    ```bash
    # 基本下载
    scrapling extract get "https://news.site.com" news.md
    
    # 自定义超时下载
    scrapling extract get "https://example.com" content.txt --timeout 60
    
    # 使用 CSS 选择器仅提取特定内容
    scrapling extract get "https://blog.example.com" articles.md --css-selector "article"
   
    # 带 Cookie 发送请求
    scrapling extract get "https://scrapling.requestcatcher.com" content.md --cookies "session=abc123; user=john"
   
    # 添加 User-Agent
    scrapling extract get "https://api.site.com" data.json -H "User-Agent: MyBot 1.0"
    
    # 添加多个请求头
    scrapling extract get "https://site.com" page.html -H "Accept: text/html" -H "Accept-Language: en-US"
    ```
    通过 `scrapling extract get --help` 获取该命令的可用选项：
    ```bash
    Usage: scrapling extract get [OPTIONS] URL OUTPUT_FILE
    
      Perform a GET request and save the content to a file.
    
      The output file path can be an HTML file, a Markdown file of the HTML content, or the text content itself. Use file extensions (`.html`/`.md`/`.txt`) respectively.
    
    Options:
      -H, --headers TEXT                             HTTP headers in format "Key: Value" (can be used multiple times)
      --cookies TEXT                                 Cookies string in format "name1=value1;name2=value2"
      --timeout INTEGER                              Request timeout in seconds (default: 30)
      --proxy TEXT                                   Proxy URL in format "http://username:password@host:port"
      -s, --css-selector TEXT                        CSS selector to extract specific content from the page. It returns all matches.
      -p, --params TEXT                              Query parameters in format "key=value" (can be used multiple times)
      --follow-redirects / --no-follow-redirects     Whether to follow redirects (default: True)
      --verify / --no-verify                         Whether to verify SSL certificates (default: True)
      --impersonate TEXT                             Browser to impersonate (e.g., chrome, firefox).
      --stealthy-headers / --no-stealthy-headers     Use stealthy browser headers (default: True)
      --ai-targeted                                  Extract only main content and sanitize hidden elements for AI consumption (default: False)
      --help                                         Show this message and exit.
    
    ```
    请注意，这些选项对其他请求命令同样适用，因此无需重复说明。

2. **POST 请求**
    
    ```bash
    scrapling extract post [URL] [OUTPUT_FILE] [OPTIONS]
    ```
    
    **示例：**
    ```bash
    # 提交表单数据
    scrapling extract post "https://api.site.com/search" results.html --data "query=python&type=tutorial"
    
    # 发送 JSON 数据
    scrapling extract post "https://api.site.com" response.json --json '{"username": "test", "action": "search"}'
    ```
    通过 `scrapling extract post --help` 获取该命令的可用选项：
    ```bash
    Usage: scrapling extract post [OPTIONS] URL OUTPUT_FILE
    
      Perform a POST request and save the content to a file.
    
      The output file path can be an HTML file, a Markdown file of the HTML content, or the text content itself. Use file extensions (`.html`/`.md`/`.txt`) respectively.
    
    Options:
      -d, --data TEXT                                Form data to include in the request body (as string, ex: "param1=value1&param2=value2")
      -j, --json TEXT                                JSON data to include in the request body (as string)
      -H, --headers TEXT                             HTTP headers in format "Key: Value" (can be used multiple times)
      --cookies TEXT                                 Cookies string in format "name1=value1;name2=value2"
      --timeout INTEGER                              Request timeout in seconds (default: 30)
      --proxy TEXT                                   Proxy URL in format "http://username:password@host:port"
      -s, --css-selector TEXT                        CSS selector to extract specific content from the page. It returns all matches.
      -p, --params TEXT                              Query parameters in format "key=value" (can be used multiple times)
      --follow-redirects / --no-follow-redirects     Whether to follow redirects (default: True)
      --verify / --no-verify                         Whether to verify SSL certificates (default: True)
      --impersonate TEXT                             Browser to impersonate (e.g., chrome, firefox).
      --stealthy-headers / --no-stealthy-headers     Use stealthy browser headers (default: True)
      --ai-targeted                                  Extract only main content and sanitize hidden elements for AI consumption (default: False)
      --help                                         Show this message and exit.
    
    ```

3. **PUT 请求**
    
    ```bash
    scrapling extract put [URL] [OUTPUT_FILE] [OPTIONS]
    ```
    
    **示例：**
    ```bash
    # 发送数据
    scrapling extract put "https://scrapling.requestcatcher.com/put" results.html --data "update=info" --impersonate "firefox"
    
    # 发送 JSON 数据
    scrapling extract put "https://scrapling.requestcatcher.com/put" response.json --json '{"username": "test", "action": "search"}'
    ```
    通过 `scrapling extract put --help` 获取该命令的可用选项：
    ```bash
    Usage: scrapling extract put [OPTIONS] URL OUTPUT_FILE
    
      Perform a PUT request and save the content to a file.
    
      The output file path can be an HTML file, a Markdown file of the HTML content, or the text content itself. Use file extensions (`.html`/`.md`/`.txt`) respectively.
    
    Options:
      -d, --data TEXT                                Form data to include in the request body
      -j, --json TEXT                                JSON data to include in the request body (as string)
      -H, --headers TEXT                             HTTP headers in format "Key: Value" (can be used multiple times)
      --cookies TEXT                                 Cookies string in format "name1=value1;name2=value2"
      --timeout INTEGER                              Request timeout in seconds (default: 30)
      --proxy TEXT                                   Proxy URL in format "http://username:password@host:port"
      -s, --css-selector TEXT                        CSS selector to extract specific content from the page. It returns all matches.
      -p, --params TEXT                              Query parameters in format "key=value" (can be used multiple times)
      --follow-redirects / --no-follow-redirects     Whether to follow redirects (default: True)
      --verify / --no-verify                         Whether to verify SSL certificates (default: True)
      --impersonate TEXT                             Browser to impersonate (e.g., chrome, firefox).
      --stealthy-headers / --no-stealthy-headers     Use stealthy browser headers (default: True)
      --ai-targeted                                  Extract only main content and sanitize hidden elements for AI consumption (default: False)
      --help                                         Show this message and exit.
    ```

4. **DELETE 请求**
    
    ```bash
    scrapling extract delete [URL] [OUTPUT_FILE] [OPTIONS]
    ```
    
    **示例：**
    ```bash
    # 发送数据
    scrapling extract delete "https://scrapling.requestcatcher.com/delete" results.html
    
    # 发送 JSON 数据
    scrapling extract delete "https://scrapling.requestcatcher.com/" response.txt --impersonate "chrome"
    ```
    通过 `scrapling extract delete --help` 获取该命令的可用选项：
    ```bash
    Usage: scrapling extract delete [OPTIONS] URL OUTPUT_FILE
    
      Perform a DELETE request and save the content to a file.
    
      The output file path can be an HTML file, a Markdown file of the HTML content, or the text content itself. Use file extensions (`.html`/`.md`/`.txt`) respectively.
    
    Options:
      -H, --headers TEXT                             HTTP headers in format "Key: Value" (can be used multiple times)
      --cookies TEXT                                 Cookies string in format "name1=value1;name2=value2"
      --timeout INTEGER                              Request timeout in seconds (default: 30)
      --proxy TEXT                                   Proxy URL in format "http://username:password@host:port"
      -s, --css-selector TEXT                        CSS selector to extract specific content from the page. It returns all matches.
      -p, --params TEXT                              Query parameters in format "key=value" (can be used multiple times)
      --follow-redirects / --no-follow-redirects     Whether to follow redirects (default: True)
      --verify / --no-verify                         Whether to verify SSL certificates (default: True)
      --impersonate TEXT                             Browser to impersonate (e.g., chrome, firefox).
      --stealthy-headers / --no-stealthy-headers     Use stealthy browser headers (default: True)
      --ai-targeted                                  Extract only main content and sanitize hidden elements for AI consumption (default: False)
      --help                                         Show this message and exit.
    ```

### 浏览器获取

1. **fetch - 处理动态内容**

    适用于加载动态内容或有轻度防护的网站
    
    ```bash
    scrapling extract fetch [URL] [OUTPUT_FILE] [OPTIONS]
    ```
    
    **示例：**
    ```bash
    # 等待 JavaScript 加载内容并完成网络活动
    scrapling extract fetch "https://scrapling.requestcatcher.com/" content.md --network-idle
    
    # 等待特定内容出现
    scrapling extract fetch "https://scrapling.requestcatcher.com/" data.txt --wait-selector ".content-loaded"
    
    # 以可见浏览器模式运行（便于调试）
    scrapling extract fetch "https://scrapling.requestcatcher.com/" page.html --no-headless --disable-resources
    ```
    通过 `scrapling extract fetch --help` 获取该命令的可用选项：
    ```bash
    Usage: scrapling extract fetch [OPTIONS] URL OUTPUT_FILE
    
      Use DynamicFetcher to fetch content with browser automation.
    
      The output file path can be an HTML file, a Markdown file of the HTML content, or the text content itself. Use file extensions (`.html`/`.md`/`.txt`) respectively.
    
    Options:
      --headless / --no-headless                  Run browser in headless mode (default: True)
      --disable-resources / --enable-resources    Drop unnecessary resources for speed boost (default: False)
      --network-idle / --no-network-idle          Wait for network idle (default: False)
      --timeout INTEGER                           Timeout in milliseconds (default: 30000)
      --wait INTEGER                              Additional wait time in milliseconds after page load (default: 0)
      -s, --css-selector TEXT                     CSS selector to extract specific content from the page. It returns all matches.
      --wait-selector TEXT                        CSS selector to wait for before proceeding
      --locale TEXT                               Specify user locale. Defaults to the system default locale.
      --real-chrome/--no-real-chrome              If you have a Chrome browser installed on your device, enable this, and the Fetcher will launch an instance of your browser and use it. (default: False)
      --proxy TEXT                                Proxy URL in format "http://username:password@host:port"
      -H, --extra-headers TEXT                    Extra headers in format "Key: Value" (can be used multiple times)
      --dns-over-https / --no-dns-over-https   Route DNS through Cloudflare's DoH to prevent DNS leaks when using proxies (default: False)
      --block-ads / --no-block-ads               Block requests to known ad and tracker domains (default: False)
      --ai-targeted                              Extract only main content and sanitize hidden elements for AI consumption (default: False)
      --help                                      Show this message and exit.
    ```

2. **stealthy-fetch - 绕过防护**

    适用于有反机器人防护或 Cloudflare 防护的网站
    
    ```bash
    scrapling extract stealthy-fetch [URL] [OUTPUT_FILE] [OPTIONS]
    ```
    
    **示例：**
    ```bash
    # 绕过基本防护
    scrapling extract stealthy-fetch "https://scrapling.requestcatcher.com" content.md
    
    # 解决 Cloudflare 挑战
    scrapling extract stealthy-fetch "https://nopecha.com/demo/cloudflare" data.txt --solve-cloudflare --css-selector "#padded_content a"
    
    # 使用代理匿名访问
    scrapling extract stealthy-fetch "https://site.com" content.md --proxy "http://proxy-server:8080"
    ```
    通过 `scrapling extract stealthy-fetch --help` 获取该命令的可用选项：
    ```bash
    Usage: scrapling extract stealthy-fetch [OPTIONS] URL OUTPUT_FILE
    
      Use StealthyFetcher to fetch content with advanced stealth features.
    
      The output file path can be an HTML file, a Markdown file of the HTML content, or the text content itself. Use file extensions (`.html`/`.md`/`.txt`) respectively.
    
    Options:
      --headless / --no-headless                  Run browser in headless mode (default: True)
      --disable-resources / --enable-resources    Drop unnecessary resources for speed boost (default: False)
      --block-webrtc / --allow-webrtc             Block WebRTC entirely (default: False)
      --solve-cloudflare / --no-solve-cloudflare  Solve Cloudflare challenges (default: False)
      --allow-webgl / --block-webgl               Allow WebGL (default: True)
      --network-idle / --no-network-idle          Wait for network idle (default: False)
      --real-chrome/--no-real-chrome              If you have a Chrome browser installed on your device, enable this, and the Fetcher will launch an instance of your browser and use it. (default: False)
      --timeout INTEGER                           Timeout in milliseconds (default: 30000)
      --wait INTEGER                              Additional wait time in milliseconds after page load (default: 0)
      -s, --css-selector TEXT                     CSS selector to extract specific content from the page. It returns all matches.
      --wait-selector TEXT                        CSS selector to wait for before proceeding
      --hide-canvas / --show-canvas               Add noise to canvas operations (default: False)
      --proxy TEXT                                Proxy URL in format "http://username:password@host:port"
      -H, --extra-headers TEXT                    Extra headers in format "Key: Value" (can be used multiple times)
      --dns-over-https / --no-dns-over-https   Route DNS through Cloudflare's DoH to prevent DNS leaks when using proxies (default: False)
      --block-ads / --no-block-ads               Block requests to known ad and tracker domains (default: False)
      --ai-targeted                              Extract only main content and sanitize hidden elements for AI consumption (default: False)
      --help                                      Show this message and exit.
    ```

## 何时使用各命令

若你不是 Web 抓取专家且难以抉择，可参考以下公式：

- 对简单网站、博客或新闻文章使用 **`get`**
- 对现代 Web 应用或有动态内容的网站使用 **`fetch`**
- 对有防护的网站、Cloudflare 或反机器人系统使用 **`stealthy-fetch`**

## 法律与伦理考量

⚠️ **重要准则：**

- **检查 robots.txt**：访问 `https://website.com/robots.txt` 查看抓取规则
- **遵守速率限制**：不要用请求压垮服务器
- **服务条款**：阅读并遵守网站条款
- **版权**：尊重知识产权
- **隐私**：注意个人数据保护法律
- **商业用途**：确保获得业务用途许可

---

*祝抓取愉快！请始终尊重网站政策并遵守所有适用法律与法规。*
