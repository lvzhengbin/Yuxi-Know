---
type: pitfall
keywords: [MCP, streamable_http, 连接超时, API挂起, 智能体加载, get_graph, 缓存, asyncio, httpx]
related_files:
  - src/services/mcp_service.py
  - src/agents/chatbot/graph.py
  - src/agents/deep_agent/graph.py
  - src/agents/reporter/graph.py
  - src/agents/common/base.py
severity: high
date: 2026-03-17
---

# MCP HTTP 服务器连接超时导致智能体 API 无限挂起

## 问题描述

进入智能体（Agent）页面后，左侧边栏无限 loading，`GET /api/chat/agent` 接口永远不返回响应。

### 触发场景

- 配置了 `streamable_http` 类型的 MCP 服务器（如 `sequentialthinking`，地址为远程 HTTPS URL）
- 远程 MCP 服务器响应缓慢或无法连接
- 服务刚启动，智能体图（graph）尚未构建

### 根本原因

API 挂起的链路如下：

```
GET /api/chat/agent
  → AgentManager.get_agents_info()
  → asyncio.gather(agent.get_info() for each agent)   ← 3 个智能体并发
    → agent.check_checkpointer()
    → agent.get_graph()                                ← 每次请求都重新构建！
    → get_tools_from_all_servers()
      → get_mcp_tools("sequentialthinking")            ← streamable_http 传输
        → MultiServerMCPClient 连接远程服务器
        → httpx 默认超时 30 秒 (DEFAULT_STREAMABLE_HTTP_TIMEOUT)
        → 连接失败时等待满 30 秒才超时
```

两个复合问题：
1. **无缓存**：`get_graph()` 没有缓存机制，每次请求都重新构建所有智能体的图，每次都要重新连接 MCP 服务器。
2. **无超时**：`langchain_mcp_adapters` 对 HTTP 类型 MCP 服务器默认 30 秒超时，`asyncio.wait_for` 的取消信号还会触发 MCP client 的异步清理代码（同样可能挂起），双重挂起。

## 影响范围

- 所有使用了 HTTP 类型 MCP 服务器（`sse`、`streamable_http`）的智能体
- 只要远程 MCP 服务器网络不稳定，`GET /api/chat/agent` 就会超时（最长 30s × 服务器数量）
- 影响前端所有依赖智能体列表的页面

## 解决方案

### 修复 1：智能体图构建缓存（性能优化，必须）

在每个智能体的 `get_graph()` 方法中加入缓存逻辑：

```python
async def get_graph(self, **kwargs):
    if self.graph is not None:   # ← 命中缓存，直接返回
        return self.graph

    # ... 构建图 ...

    self.graph = graph           # ← 保存缓存
    return graph
```

涉及文件：
- `src/agents/chatbot/graph.py` — `ChatbotAgent.get_graph()`
- `src/agents/deep_agent/graph.py` — `DeepAgent.get_graph()`
- `src/agents/reporter/graph.py` — `SqlReporterAgent.get_graph()`

效果：首次请求构建图后，后续请求直接复用，无需重连 MCP 服务器。

### 修复 2：HTTP 类型 MCP 服务器设置连接超时（可靠性保障，必须）

在 `src/services/mcp_service.py` 的 `get_mcp_tools()` 中，为 HTTP 类传输协议设置超时：

```python
# 仅对 HTTP 类型的 MCP 服务器设置超时，避免无限等待
transport = client_config.get("transport", "")
if transport in ("sse", "streamable_http", "streamable-http", "http") \
        and client_config.get("timeout") is None:
    client_config["timeout"] = 10  # 10 秒超时（httpx transport 层生效）
```

并在 `get_tools_from_all_servers()` 中包一层 `asyncio.wait_for`：

```python
async def get_tools_from_all_servers() -> list:
    all_tools = []
    for server_name in MCP_SERVERS.keys():
        try:
            tools = await asyncio.wait_for(get_mcp_tools(server_name), timeout=10.0)
            all_tools.extend(tools)
        except asyncio.TimeoutError:
            logger.warning(f"MCP server '{server_name}' timed out, skipping")
        except Exception as e:
            logger.warning(f"Failed to get tools from MCP server '{server_name}': {e}")
    return all_tools
```

> ⚠️ **注意**：`timeout` 参数只能给 HTTP 类型传输，`stdio` 类型（如 `mcp-server-chart` 用 `npx` 启动）传入 `timeout` 会报错：
> `TypeError: _create_stdio_session() got an unexpected keyword argument 'timeout'`

### 为什么不能只用 asyncio.wait_for

`asyncio.wait_for` 取消任务时，MCP client 的 `__aexit__` 清理代码（断开连接）本身也是异步的，如果服务器没有响应，清理代码同样会挂起。只有在 **httpx transport 层**设置 `timeout`，才能让连接在 socket 层快速失败，不需要 asyncio 取消机制介入。

## 验证方法

```bash
# 查看 API 响应时间
time curl -s -u admin:password http://localhost:8000/api/chat/agent | python3 -m json.tool

# 期望：首次 ~5 秒（构建图），后续 <100ms（缓存命中）
# 问题：挂起超过 30 秒
```

## 相关代码位置

| 文件 | 修改内容 |
|------|----------|
| `src/services/mcp_service.py` | `get_mcp_tools()` 添加 HTTP timeout；`get_tools_from_all_servers()` 添加 `asyncio.wait_for` |
| `src/agents/chatbot/graph.py` | `ChatbotAgent.get_graph()` 添加 `self.graph` 缓存 |
| `src/agents/deep_agent/graph.py` | `DeepAgent.get_graph()` 添加 `self.graph` 缓存 |
| `src/agents/reporter/graph.py` | `SqlReporterAgent.get_graph()` 添加 `self.graph` 缓存 |
