# Scrapling 示例

这些示例抓取 [quotes.toscrape.com](https://quotes.toscrape.com)——一个安全、专用于练习的抓取沙箱——并演示 Scrapling 的全部工具，从纯 HTTP 到完整浏览器自动化与 spider。

所有示例会收集**全部 10 页共 100 条名言**。

## 快速开始

确保已安装 Scrapling：

```bash
pip install "scrapling[all]>=0.4.9"
scrapling install --force
```

## 示例

| 文件                     | 工具              | 类型                        | 适用场景                              |
|--------------------------|-------------------|-----------------------------|---------------------------------------|
| `01_fetcher_session.py`  | `FetcherSession`  | Python - 持久 HTTP          | API、快速多页抓取                     |
| `02_dynamic_session.py`  | `DynamicSession`  | Python - 浏览器自动化       | 动态/SPA 页面                         |
| `03_stealthy_session.py` | `StealthySession` | Python - 隐身浏览器         | Cloudflare、指纹绕过                  |
| `04_spider.py`           | `Spider`          | Python - 自动爬取           | 多页爬取、整站抓取                    |

## 运行

**Python 脚本：**

```bash
python examples/01_fetcher_session.py
python examples/02_dynamic_session.py  # 打开可见浏览器
python examples/03_stealthy_session.py # 打开可见隐身浏览器
python examples/04_spider.py           # 自动爬取全部页面，导出 quotes.json
```

## 升级指南

从最快、最轻量的方式开始，仅在必要时升级：

```
get / FetcherSession
  └─ 若需 JS → fetch / DynamicSession
       └─ 若被拦截 → stealthy-fetch / StealthySession
            └─ 若多页 → Spider
```
