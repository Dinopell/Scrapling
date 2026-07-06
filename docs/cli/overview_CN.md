# 命令行界面

自 v0.3 起，Scrapling 提供强大的命令行界面，具备三大核心能力：

1. **交互式 Shell**：基于 IPython 的交互式 Web 抓取 Shell，提供多种快捷方式与实用工具
2. **Extract 命令**：在终端中无需编程即可抓取网站
3. **实用命令**：安装与管理工具

```bash
# 启动交互式 Shell
scrapling shell

# 将页面内容转换为 Markdown 并保存到文件
scrapling extract get "https://example.com" content.md

# 获取任意命令的帮助
scrapling --help
scrapling extract --help
```

## 要求
本节需要安装额外的 `shell` 依赖组，例如：
```bash
pip install "scrapling[shell]"
```
并运行以下命令安装 Fetcher 依赖：
```bash
scrapling install
```
这将下载所有浏览器及其系统依赖与指纹操控依赖。
