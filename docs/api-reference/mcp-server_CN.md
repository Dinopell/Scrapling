---
search:
  exclude: true
---

# MCP 服务器 API 参考

**Scrapling MCP 服务器**通过模型上下文协议（MCP）提供九个强大的 Web 抓取工具。该服务器将 Scrapling 能力直接集成到 AI 聊天机器人与智能体中，支持对话式 Web 抓取及高级反机器人绕过。

可通过以下命令启动 MCP 服务器：

```bash
scrapling mcp
```

或直接导入服务器类：

```python
from scrapling.core.ai import ScraplingMCPServer

server = ScraplingMCPServer()
server.serve(http=False, host="0.0.0.0", port=8000)
```

## 响应模型

所有 MCP 服务器工具返回的标准化响应结构：

## ::: scrapling.core.ai.ResponseModel
    handler: python
    :docstring:

## 会话模型

用于会话管理的模型类：

## ::: scrapling.core.ai.SessionInfo
    handler: python
    :docstring:

## ::: scrapling.core.ai.SessionCreatedModel
    handler: python
    :docstring:

## ::: scrapling.core.ai.SessionClosedModel
    handler: python
    :docstring:

## MCP 服务器类

提供所有 Web 抓取工具的主 MCP 服务器类：

## ::: scrapling.core.ai.ScraplingMCPServer
    handler: python
    :docstring:
