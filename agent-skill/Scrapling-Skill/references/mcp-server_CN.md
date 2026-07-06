# Scrapling MCP Server

Scrapling MCP 服务器通过 MCP 协议暴露十个工具。它支持基于 CSS 选择器的内容收窄（在返回结果前仅提取相关元素以减少 token 消耗）、三级抓取能力（纯 HTTP、浏览器渲染、隐身/反机器人绕过）、持久化浏览器会话管理，以及以真实图片内容块形式返回的页面截图。

所有抓取工具均返回 `ResponseModel`，字段包括：`status`（int）、`content`（字符串列表）、`url`（str）。`screenshot` 工具返回 MCP 内容块列表：先是 `ImageContent`（截图字节），随后是 `TextContent`（重定向后的 URL）。

## 工具

### `get` -- HTTP 请求（单个 URL）

快速 HTTP GET，带浏览器指纹模拟（TLS、请求头）。适用于静态页面且无/低机器人防护的场景。

**主要参数：**

| Parameter           | Type                               | Default      | Description                                                        |
|---------------------|------------------------------------|--------------|--------------------------------------------------------------------|
| `url`               | str                                | required     | 要抓取的 URL                                                       |
| `extraction_type`   | `"markdown"` / `"html"` / `"text"` | `"markdown"` | 输出格式                                                           |
| `css_selector`      | str or null                        | null         | 用于收窄内容的 CSS 选择器（在 `main_content_only` 之后应用）       |
| `main_content_only` | bool                               | true         | 限制为 `<body>` 内容                                               |
| `impersonate`       | str                                | `"chrome"`   | 要模拟的浏览器指纹                                                 |
| `proxy`             | str or null                        | null         | 代理 URL，例如 `"http://user:pass@host:port"`                      |
| `proxy_auth`        | dict or null                       | null         | `{"username": "...", "password": "..."}`                           |
| `auth`              | dict or null                       | null         | HTTP 基本认证，格式与 proxy_auth 相同                              |
| `timeout`           | number                             | 30           | 超时时间（秒）                                                     |
| `retries`           | int                                | 3            | 失败时的重试次数                                                   |
| `retry_delay`       | int                                | 1            | 重试间隔（秒）                                                     |
| `stealthy_headers`  | bool                               | true         | 生成逼真的浏览器请求头及 Google referer                            |
| `http3`             | bool                               | false        | 使用 HTTP/3（可能与 `impersonate` 冲突）                           |
| `follow_redirects`  | bool or "safe"                     | "safe"       | 是否跟随重定向。"safe" 会拒绝重定向到内网/私有 IP                    |
| `max_redirects`     | int                                | 30           | 最大重定向次数（-1 表示不限）                                      |
| `headers`           | dict or null                       | null         | 自定义请求头                                                       |
| `cookies`           | dict or null                       | null         | 请求 Cookie                                                        |
| `params`            | dict or null                       | null         | 查询字符串参数                                                     |
| `verify`            | bool                               | true         | 验证 HTTPS 证书                                                    |

### `bulk_get` -- HTTP 请求（多个 URL）

`get` 的异步并发版本。参数相同，但 `url` 替换为 `urls`（字符串列表）。所有 URL 并行抓取。返回 `ResponseModel` 列表。

### `fetch` -- 浏览器抓取（单个 URL）

通过 Playwright 打开 Chromium 浏览器以渲染 JavaScript。适用于动态/SPA 页面且无/低机器人防护的场景。

**主要参数（除共享参数外）：**

| Parameter             | Type                | Default      | Description                                                                     |
|-----------------------|---------------------|--------------|---------------------------------------------------------------------------------|
| `url`                 | str                 | required     | 要抓取的 URL                                                                    |
| `extraction_type`     | str                 | `"markdown"` | `"markdown"` / `"html"` / `"text"`                                              |
| `css_selector`        | str or null         | null         | 提取前收窄内容                                                                  |
| `main_content_only`   | bool                | true         | 限制为 `<body>`                                                                 |
| `headless`            | bool                | true         | 隐藏运行浏览器（true）或可见运行（false）                                       |
| `proxy`               | str or dict or null | null         | 字符串 URL 或 `{"server": "...", "username": "...", "password": "..."}`         |
| `timeout`             | number              | 30000        | 超时时间（**毫秒**）                                                            |
| `wait`                | number              | 0            | 页面加载后、提取前的额外等待（毫秒）                                            |
| `wait_selector`       | str or null         | null         | 提取前等待的 CSS 选择器                                                         |
| `wait_selector_state` | str                 | `"attached"` | wait_selector 的状态：`"attached"` / `"visible"` / `"hidden"` / `"detached"`   |
| `network_idle`        | bool                | false        | 等待 500ms 内无网络活动                                                         |
| `disable_resources`   | bool                | false        | 阻止字体、图片、媒体、样式表等以提升速度                                        |
| `google_search`       | bool                | true         | 设置 Google referer 请求头                                                      |
| `real_chrome`         | bool                | false        | 使用本地安装的 Chrome，而非内置 Chromium                                        |
| `cdp_url`             | str or null         | null         | 通过 CDP URL 连接已有浏览器                                                     |
| `extra_headers`       | dict or null        | null         | 附加请求头                                                                      |
| `useragent`           | str or null         | null         | 自定义 User-Agent（为 null 时自动生成）                                         |
| `cookies`             | list or null        | null         | Playwright 格式的 Cookie                                                        |
| `timezone_id`         | str or null         | null         | 浏览器时区，例如 `"America/New_York"`                                           |
| `locale`              | str or null         | null         | 浏览器区域设置，例如 `"en-GB"`                                                  |
| `session_id`          | str or null         | null         | 复用 `open_session` 创建的持久会话，而非新建浏览器                              |

### `bulk_fetch` -- 浏览器抓取（多个 URL）

`fetch` 的并发浏览器版本。参数相同（含 `session_id`），但 `url` 替换为 `urls`（字符串列表）。每个 URL 在独立浏览器标签页中打开。返回 `ResponseModel` 列表。

### `stealthy_fetch` -- 隐身浏览器抓取（单个 URL）

带指纹伪装的反机器人绕过抓取器。适用于 Cloudflare Turnstile/Interstitial 或其他强防护站点。

**附加参数（除 `fetch` 中的参数外）：**

| Parameter          | Type         | Default | Description                                                      |
|--------------------|--------------|---------|------------------------------------------------------------------|
| `solve_cloudflare` | bool         | false   | 自动解决 Cloudflare Turnstile/Interstitial 挑战                  |
| `hide_canvas`      | bool         | false   | 为 canvas 操作添加噪声以防止指纹识别                             |
| `block_webrtc`     | bool         | false   | 强制 WebRTC 遵守代理设置（防止 IP 泄露）                         |
| `allow_webgl`      | bool         | true    | 保持 WebGL 启用（禁用会被 WAF 检测到）                           |
| `additional_args`  | dict or null | null    | 额外的 Playwright 上下文参数（覆盖 Scrapling 默认值）            |
| `session_id`       | str or null  | null    | 复用 `open_session` 创建的持久隐身会话                           |

同时接受 `fetch` 的全部参数。

### `bulk_stealthy_fetch` -- 隐身浏览器抓取（多个 URL）

并发隐身版本。参数与 `stealthy_fetch` 相同（含 `session_id`），但 `url` 替换为 `urls`（字符串列表）。返回 `ResponseModel` 列表。

### `open_session` -- 创建持久浏览器会话

打开一个跨多次 fetch 调用保持存活的浏览器会话，避免每次启动新浏览器的开销。返回 `SessionCreatedModel`，包含 `session_id`、`session_type`、`created_at`、`is_alive` 和 `message`。

**主要参数：**

| Parameter          | Type                        | Default      | Description                                                                                           |
|--------------------|-----------------------------|--------------|-------------------------------------------------------------------------------------------------------|
| `session_type`     | `"dynamic"` / `"stealthy"`  | required     | 要创建的浏览器会话类型                                                                                |
| `session_id`       | str or null                 | null         | 会话的自定义 ID。省略时生成 12 位随机十六进制 ID。若已被占用则抛出异常                                |
| `headless`         | bool                        | true         | 隐藏或可见运行浏览器                                                                                  |
| `max_pages`        | int                         | 5            | 最大并发浏览器标签页数（1-50）                                                                        |
| `proxy`            | str or dict or null         | null         | 本会话中所有请求的代理                                                                                |
| `timeout`          | number                      | 30000        | 默认超时（毫秒）                                                                                      |
| `solve_cloudflare` | bool                        | false        | （仅 Stealthy）自动解决 Cloudflare 挑战                                                               |
| `hide_canvas`      | bool                        | false        | （仅 Stealthy）Canvas 指纹噪声                                                                        |
| `block_webrtc`     | bool                        | false        | （仅 Stealthy）阻止 WebRTC IP 泄露                                                                    |
| `allow_webgl`      | bool                        | true         | （仅 Stealthy）保持 WebGL 启用                                                                        |

另含所有其他浏览器会话参数（`google_search`、`real_chrome`、`cdp_url`、`locale`、`timezone_id`、`useragent`、`extra_headers`、`cookies`、`disable_resources`、`network_idle`、`wait_selector`、`wait_selector_state`）。

dynamic 会话只能用于 `fetch`/`bulk_fetch`。stealthy 会话只能用于 `stealthy_fetch`/`bulk_stealthy_fetch`。

### `close_session` -- 关闭持久浏览器会话

关闭会话并释放浏览器资源。用完后务必关闭会话。

| Parameter    | Type | Default  | Description                      |
|--------------|------|----------|----------------------------------|
| `session_id` | str  | required | 来自 `open_session` 的会话 ID  |

返回 `SessionClosedModel`，包含 `session_id` 和 `message`。

### `list_sessions` -- 列出活跃会话

返回 `SessionInfo` 对象列表，每个对象包含 `session_id`、`session_type`、`created_at` 和 `is_alive`。

无参数。

### `screenshot` -- 捕获页面截图

在已有浏览器会话中导航到 URL，并返回 MCP `ImageContent` 块（模型可直接查看的字节，而非 JSON 中的 base64 字符串），随后是携带重定向后 URL 的 `TextContent` 块。

需要已打开的浏览器会话。先调用 `open_session`，再在此处传入 `session_id`。`dynamic` 和 `stealthy` 会话均可使用。

| Parameter             | Type                  | Default      | Description                                                                          |
|-----------------------|-----------------------|--------------|--------------------------------------------------------------------------------------|
| `url`                 | str                   | required     | 要导航并截图的 URL                                                                   |
| `session_id`          | str                   | required     | 通过 `open_session` 创建的已打开浏览器会话 ID                                        |
| `image_type`          | `"png"` / `"jpeg"`    | `"png"`      | 图片格式。使用 `"jpeg"` 可减小负载                                                   |
| `full_page`           | bool                  | false        | 截取完整可滚动页面，而非仅视口                                                       |
| `quality`             | int or null           | null         | JPEG 质量 0-100。与 `image_type="png"` 一起传入会抛出异常                            |
| `wait`                | number                | 0            | 页面加载后、截图前的额外等待（毫秒）                                                 |
| `wait_selector`       | str or null           | null         | 截图前等待的 CSS 选择器                                                              |
| `wait_selector_state` | str                   | `"attached"` | `wait_selector` 的状态：`"attached"` / `"visible"` / `"hidden"` / `"detached"`      |
| `network_idle`        | bool                  | false        | 等待 500ms 内无网络活动                                                              |
| `timeout`             | number                | 30000        | 超时时间（毫秒）                                                                     |

## 工具选择指南

| Scenario                                 | Tool                                                          |
|------------------------------------------|---------------------------------------------------------------|
| 静态页面，无机器人防护                   | `get`                                                         |
| 多个静态页面                             | `bulk_get`                                                    |
| JavaScript 渲染 / SPA 页面               | `fetch`                                                       |
| 多个 JS 渲染页面                         | `bulk_fetch`                                                  |
| Cloudflare 或强反机器人防护              | `stealthy_fetch`（Turnstile 需设 `solve_cloudflare=true`）    |
| 多个受保护页面                           | `bulk_stealthy_fetch`                                         |
| 同一站点的多个页面                       | `open_session` + 带 `session_id` 的 `fetch`/`stealthy_fetch`  |
| 需要页面截图                             | `open_session` + 带 `session_id` 的 `screenshot`              |

优先使用 `get`（最快、资源消耗最低）。若内容需要 JS 渲染，再升级到 `fetch`。仅在遭拦截时使用 `stealthy_fetch`。同一站点多页抓取时，使用持久会话以避免重复启动浏览器的开销。

## 内容提取技巧

- 使用 `css_selector` 在结果到达模型前收窄内容——可显著节省 token。
- `main_content_only=true`（默认）通过限制为 `<body>` 去除导航/页脚。
- `extraction_type="markdown"`（默认）可读性最佳。需要最小输出时用 `"text"`，需要保留结构时用 `"html"`。
- 若 `css_selector` 匹配多个元素，全部会出现在 `content` 列表中。

## 提示词注入防护

当 `main_content_only=true`（默认）时，服务器会自动清理抓取内容，防止恶意网站的提示词注入。会移除：

- CSS 隐藏元素（`display:none`、`visibility:hidden`、`opacity:0`、`font-size:0`、`height:0`、`width:0`）
- `aria-hidden="true"` 元素
- `<template>` 标签
- HTML 注释
- 零宽 Unicode 字符

为获得最大防护，请保持 `main_content_only=true`。

## 广告拦截

所有基于浏览器的工具（`fetch`、`bulk_fetch`、`stealthy_fetch`、`bulk_stealthy_fetch`）及持久会话（`open_session`）会自动拦截约 3,500 个已知广告与追踪域名的请求。MCP 服务器中始终启用，以节省 token 并加快页面加载。无需配置。

## 设置

启动服务器（stdio 传输，大多数 MCP 客户端使用）：

```bash
scrapling mcp
```

或使用 Streamable HTTP 传输：

```bash
scrapling mcp --http
scrapling mcp --http --host 127.0.0.1 --port 8000
```

Docker 替代方案：

```bash
docker pull pyd4vinci/scrapling
docker run -i --rm pyd4vinci/scrapling mcp
```

向客户端注册时，MCP 服务器名称为 `ScraplingServer`。命令为 `scrapling` 二进制路径，参数为 `mcp`。
