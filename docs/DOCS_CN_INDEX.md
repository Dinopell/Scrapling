# Scrapling 中文文档索引

本目录下的英文文档保持不变；每个英文文档均有对应的 **`_CN.md`** 中文版本。

> 建议从 [overview_CN.md](overview_CN.md) 或 [index_CN.md](index_CN.md) 开始阅读。

---

## 入门

| 中文文档 | 说明 |
|----------|------|
| [index_CN.md](index_CN.md) | 项目首页（功能概览） |
| [overview_CN.md](overview_CN.md) | 快速入门：解析 + 抓取 |
| [benchmarks_CN.md](benchmarks_CN.md) | 性能基准测试 |
| [donate_CN.md](donate_CN.md) | 赞助与支持 |

## 解析（Parsing）

| 中文文档 | 说明 |
|----------|------|
| [parsing/selection_CN.md](parsing/selection_CN.md) | CSS、XPath、文本选择 |
| [parsing/main_classes_CN.md](parsing/main_classes_CN.md) | Selector、Response 核心类 |
| [parsing/adaptive_CN.md](parsing/adaptive_CN.md) | 自适应抓取（页面改版容错） |

## 抓取（Fetching）

| 中文文档 | 说明 |
|----------|------|
| [fetching/choosing_CN.md](fetching/choosing_CN.md) | 如何选择 Fetcher |
| [fetching/static_CN.md](fetching/static_CN.md) | HTTP 请求（Fetcher） |
| [fetching/dynamic_CN.md](fetching/dynamic_CN.md) | 动态页面（Playwright） |
| [fetching/stealthy_CN.md](fetching/stealthy_CN.md) | 反检测 / Cloudflare 绕过 |

## 爬虫（Spiders）

| 中文文档 | 说明 |
|----------|------|
| [spiders/getting-started_CN.md](spiders/getting-started_CN.md) | Spider 入门 |
| [spiders/architecture_CN.md](spiders/architecture_CN.md) | 架构与数据流 |
| [spiders/requests-responses_CN.md](spiders/requests-responses_CN.md) | Request / Response |
| [spiders/sessions_CN.md](spiders/sessions_CN.md) | 多 Session 配置 |
| [spiders/proxy-blocking_CN.md](spiders/proxy-blocking_CN.md) | 代理与拦截处理 |
| [spiders/generic-templates_CN.md](spiders/generic-templates_CN.md) | 通用模板 |
| [spiders/advanced_CN.md](spiders/advanced_CN.md) | 高级功能 |

## 命令行（CLI）

| 中文文档 | 说明 |
|----------|------|
| [cli/overview_CN.md](cli/overview_CN.md) | CLI 总览 |
| [cli/interactive-shell_CN.md](cli/interactive-shell_CN.md) | 交互式 Shell |
| [cli/extract-commands_CN.md](cli/extract-commands_CN.md) | extract 命令 |

## API 参考

| 中文文档 | 说明 |
|----------|------|
| [api-reference/selector_CN.md](api-reference/selector_CN.md) | Selector API |
| [api-reference/response_CN.md](api-reference/response_CN.md) | Response API |
| [api-reference/fetchers_CN.md](api-reference/fetchers_CN.md) | Fetchers API |
| [api-reference/spiders_CN.md](api-reference/spiders_CN.md) | Spiders API |
| [api-reference/proxy-rotation_CN.md](api-reference/proxy-rotation_CN.md) | 代理轮换 API |
| [api-reference/custom-types_CN.md](api-reference/custom-types_CN.md) | 自定义类型 |
| [api-reference/mcp-server_CN.md](api-reference/mcp-server_CN.md) | MCP Server API |

## AI / MCP

| 中文文档 | 说明 |
|----------|------|
| [ai/mcp-server_CN.md](ai/mcp-server_CN.md) | MCP 服务器使用指南 |

## 教程

| 中文文档 | 说明 |
|----------|------|
| [tutorials/migrating_from_beautifulsoup_CN.md](tutorials/migrating_from_beautifulsoup_CN.md) | 从 BeautifulSoup 迁移 |
| [tutorials/replacing_ai_CN.md](tutorials/replacing_ai_CN.md) | 替代 AI 抓取方案 |

## 开发扩展

| 中文文档 | 说明 |
|----------|------|
| [development/adaptive_storage_system_CN.md](development/adaptive_storage_system_CN.md) | 自定义 adaptive 存储 |
| [development/scrapling_custom_types_CN.md](development/scrapling_custom_types_CN.md) | 自定义类型扩展 |

## Agent Skill 中文文档

| 中文文档 | 说明 |
|----------|------|
| [../agent-skill/README_CN.md](../agent-skill/README_CN.md) | Agent Skill 说明 |
| [../agent-skill/Scrapling-Skill/examples/README_CN.md](../agent-skill/Scrapling-Skill/examples/README_CN.md) | 示例脚本说明 |

---

## 推荐阅读顺序

```
1. overview_CN.md          → 整体了解
2. parsing/selection_CN.md → 学会解析 HTML
3. fetching/choosing_CN.md → 选对 Fetcher
4. fetching/static_CN.md   → HTTP 抓取
5. spiders/getting-started_CN.md → 写第一个 Spider
6. parsing/adaptive_CN.md  → 自适应特性
```

## 命名规则

- 英文原文：`foo.md`
- 中文译文：`foo_CN.md`
- 英文原文**不会被修改**
