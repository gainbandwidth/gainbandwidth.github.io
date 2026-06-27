---
title: "MewCode 骨架篇（六）：MCP 协议集成"
date: 2026-06-27
draft: false
tags: [MewCode, Agent, LLM]
categories: [AI Agent]
summary: "MCP 双传输、延迟加载与 85% Token 节省"
ShowToc: true
---

## 概述

[Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 是当前最流行的 Agent 外部工具扩展协议。MewCode 在 `mewcode/mcp/` 模块中实现了一套完整的 MCP 客户端集成，涵盖三个核心问题：

1. **怎么连** —— stdio 与 Streamable HTTP 双传输适配
2. **怎么管** —— `MCPManager` 统一管理多个 MCP Server 的生命周期
3. **怎么省** —— 延迟加载（Deferred Loading）机制，在工具多时节省高达 85% 的 Token 开销

整个模块只有四个文件，约 300 行代码，却撑起了 Agent 与外部世界交互的桥梁。

## 核心设计

### 双传输：stdio 与 Streamable HTTP

MCP 协议定义了多种传输方式，MewCode 实现了最常用的两种。`MCPClient` 通过配置中的 `is_stdio` 属性自动选择：

```python
# config.py
@property
def is_stdio(self) -> bool:
    return self.command is not None
```

**stdio 传输**适用于本地进程——启动一个子进程，通过标准输入/输出通信：

```python
async def _connect_stdio(self) -> tuple[Any, Any]:
    params = StdioServerParameters(
        command=self.config.command,
        args=self.config.args,
        env=build_child_env(self.config.env),
    )
    devnull = open(os.devnull, "w")
    self._stack.callback(devnull.close)
    read, write = await self._stack.enter_async_context(
        stdio_client(params, errlog=devnull)
    )
    return read, write
```

注意 `errlog=devnull` 的设计——子进程的 stderr 输出被丢弃，避免 MCP Server 的日志干扰 Agent 的正常运行。`build_child_env()` 负责将配置中的环境变量与系统环境合并，并解析 `${VAR}` 形式的变量引用。

**Streamable HTTP 传输**适用于远程服务——通过 HTTP 长连接通信，支持自定义 headers（如鉴权 Token）：

```python
async def _connect_http(self) -> tuple[Any, Any]:
    resolved_headers = {
        k: resolve_env_vars(v) for k, v in self.config.headers.items()
    }
    http_client = httpx.AsyncClient(
        headers=resolved_headers,
        follow_redirects=True,
    )
    await self._stack.enter_async_context(http_client)
    result = await self._stack.enter_async_context(
        streamable_http_client(self.config.url, http_client=http_client)
    )
    return result[0], result[1]
```

Headers 中的 `${API_KEY}` 会在运行时通过 `resolve_env_vars()` 被替换为真实的环境变量值，避免敏感信息硬编码在配置文件中。

两种传输都使用 `AsyncExitStack` 管理异步上下文，确保连接在关闭时被正确清理。

### MCPManager：多服务器生命周期管理

`MCPManager` 是所有 MCP 连接的总调度中心，其核心数据结构很简单：

```python
class MCPManager:
    def __init__(self) -> None:
        self._configs: dict[str, MCPServerConfig] = {}
        self._clients: dict[str, MCPClient] = {}
```

**启动阶段**，`register_all_tools()` 遍历所有配置，逐一建立连接并注册工具：

```python
async def register_all_tools(self, registry: ToolRegistry) -> list[str]:
    errors: list[str] = []
    for name, config in self._configs.items():
        try:
            client = MCPClient(config)
            await client.connect()
            self._clients[name] = client
            tools = await client.list_tools()
            for tool_def in tools:
                wrapper = MCPToolWrapper(name, tool_def, client)
                registry.register(wrapper)
        except Exception as e:
            errors.append(f"MCP server '{name}': {e}")
    return errors
```

关键设计：某个 MCP Server 连接失败不会导致整个启动过程崩溃，而是收集错误信息返回给调用方，其他 Server 照常工作。

**运行时**，`get_client()` 负责连接的健康检查和自动重连——如果 client 已断开，会自动创建新实例并重新连接。

**关闭阶段**，`shutdown()` 遍历所有 client 调用 `close()`，`MCPClient.close()` 内部通过 `_cleanup_stack()` 安全退出 `AsyncExitStack`，并特别处理了 `cancel scope` 异常，避免关闭时的 RuntimeError。

### 工具名称前缀规则

`MCPToolWrapper` 在包装 MCP 工具时，会自动加上 `mcp_{server_name}_` 前缀：

```python
self.name = f"mcp_{server_name}_{tool_def.name}"
```

例如，GitHub MCP Server 提供的 `create_issue` 工具会被注册为 `mcp_github_create_issue`。这解决了多 Server 场景下的工具名冲突问题——不同 Server 可能提供同名工具（如 `search`），前缀确保了全局唯一性。

同时，`MCPToolWrapper` 保留了原始工具名用于实际调用：

```python
@property
def mcp_tool_name(self) -> str:
    return self._tool_def.name
```

`execute()` 内部使用的是 `self._tool_def.name`（原始名），而不是带前缀的 `self.name`。

### 延迟加载：ToolSearch + select

这是 MewCode MCP 集成中最精巧的设计。每个 MCP 工具在注册时都会被标记为延迟加载：

```python
# tool_wrapper.py
class MCPToolWrapper(Tool):
    def __init__(self, ...):
        self.should_defer = True  # 所有 MCP 工具默认延迟
```

同时，`ToolSearch` 自身被豁免：

```python
# tool_search.py
class ToolSearchTool(Tool):
    should_defer = False  # ToolSearch 自身永远不延迟加载
```

**运行时的工作流程**：

1. Agent 每轮对话前调用 `get_all_schemas()`，该方法跳过所有 `should_defer=True` 且未被 `mark_discovered` 的工具
2. 同时通过 `get_deferred_tool_names()` 收集所有延迟工具的名称，以 system reminder 的形式告知 LLM：

```
The following deferred tools are available via ToolSearch.
Their schemas are NOT loaded - use ToolSearch with
query "select:<name>[,<name>...]" to load tool schemas before calling them:
mcp_github_create_issue
mcp_github_search_issues
mcp_slack_send_message
...
```

3. LLM 需要某个工具时，调用 `ToolSearch` 并传入 `select:mcp_github_create_issue`
4. `ToolSearch.execute()` 识别 `select:` 前缀，调用 `find_deferred_by_names()` 返回完整 schema
5. Registry 通过 `mark_discovered()` 将该工具标记为已发现，后续 `get_all_schemas()` 会包含它的完整 schema

除了精确的 `select:` 模式，还支持模糊搜索——LLM 传入关键词，`search_deferred()` 基于名称和描述进行评分排序：

```python
def search_deferred(self, query, max_results, protocol):
    # 名称完全匹配 +10，描述匹配 +5，部分匹配 +3/+1
    scored.sort(key=lambda x: x[0], reverse=True)
```

### 85% Token 节省原理

来算一笔账。假设集成了 3 个 MCP Server，共提供 20 个工具。每个工具的完整 JSON Schema（包含 name、description、input_schema 的 properties 和 required 定义）大约 300~500 tokens。

**传统方式**：每轮 API 调用都在 system prompt 中携带全部 20 个工具的完整 schema → 约 8000 tokens。

**延迟加载方式**：
- 只发送工具名称列表（每个约 15 tokens）→ 约 300 tokens
- 加上一段 system reminder（约 80 tokens）
- 合计约 380 tokens

节省比例：`(8000 - 380) / 8000 ≈ 95%`。保守估计（考虑描述文本较长等场景），也能达到 **85% 以上**。

更重要的是，Agent 一次任务通常只会用到 2~3 个 MCP 工具。通过 `select:` 按需加载后，只有这几个工具的 schema 被注入后续轮次，大幅降低了无关信息对 LLM 决策的干扰。

## 设计思考与取舍

**为什么不用懒连接（lazy connect）？** `register_all_tools()` 在启动时就会连接所有 Server 并拉取工具列表，而不是等到第一次调用时才连。这是因为 Agent 需要知道有哪些工具可用才能做规划。不过 `get_client()` 中的自动重连机制弥补了运行时断连的问题。

**为什么 `is_concurrency_safe = False`？** 所有 MCP 工具都被标记为非并发安全。这是保守策略——MewCode 无法预知外部 MCP Server 是否支持并发调用，因此默认串行执行，由调用方（Agent 循环）保证安全性。

**错误处理策略**。`MCPToolWrapper.execute()` 中，如果 client 已断开，会尝试自动重连；如果工具调用本身抛异常，会将 `_alive` 标记为 False，确保下次调用时触发重连。错误信息被包装为 `ToolResult(is_error=True)` 返回给 LLM，而不是直接抛出——这让 Agent 有机会自我修正，而不是直接崩溃。

> 📖 本文是 MewCode 系列的第 06 篇，[返回项目主页](/projects/stateful-agent-system/) 查看全部模块详解。
