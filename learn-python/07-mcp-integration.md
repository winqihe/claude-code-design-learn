# 07 - MCP 协议集成 (mcp_client.py)

> 对应源文件: `src/services/mcp/client.ts`
> Claude Code 的 Model Context Protocol 客户端实现，支持动态工具扩展。

---

## 1. MCP 协议概述

MCP (Model Context Protocol) 是 Anthropic 提出的开放协议，允许 AI 应用通过标准化接口连接外部工具和数据源。

```
┌──────────────────────────────────────────────────────────────────┐
│                    MCP 架构                                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Claude Code (MCP Host)                         │  │
│  │                                                            │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐           │  │
│  │  │ MCP Client │  │ MCP Client │  │ MCP Client │           │  │
│  │  │ (stdio)    │  │ (SSE)      │  │ (HTTP)     │           │  │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘           │  │
│  └────────┼───────────────┼───────────────┼───────────────────┘  │
│           │               │               │                      │
│     ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐              │
│     │  MCP      │   │  MCP      │   │  MCP      │              │
│     │  Server A │   │  Server B │   │  Server C │              │
│     │ (本地进程) │   │ (远程API) │   │ (Web服务) │              │
│     └───────────┘   └───────────┘   └───────────┘              │
│                                                                  │
│  MCP Server 可以暴露:                                            │
│  ├── Tools (工具) — Claude 可以调用的函数                        │
│  ├── Resources (资源) — Claude 可以读取的数据                    │
│  └── Prompts (提示模板) — 预定义的提示                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 四种传输方式

```python
from abc import ABC, abstractmethod
from typing import Any

class MCPTransport(ABC):
    @abstractmethod
    async def connect(self) -> None: ...

    @abstractmethod
    async def send(self, message: dict) -> None: ...

    @abstractmethod
    async def receive(self) -> dict: ...

    @abstractmethod
    async def close(self) -> None: ...


class StdioTransport(MCPTransport):
    """本地子进程通信，通过 stdin/stdout 交换 JSON-RPC 消息"""

    def __init__(self, command: str, args: list[str], env: dict | None = None):
        self._command = command
        self._args = args
        self._env = env or os.environ.copy()
        self._process: asyncio.subprocess.Process | None = None

    async def connect(self) -> None:
        self._process = await asyncio.create_subprocess_exec(
            self._command, *self._args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            env=self._env,
        )

    async def send(self, message: dict) -> None:
        assert self._process and self._process.stdin
        data = json.dumps(message).encode() + b"\n"
        self._process.stdin.write(data)
        await self._process.stdin.drain()

    async def receive(self) -> dict:
        assert self._process and self._process.stdout
        line = await self._process.stdout.readline()
        return json.loads(line.decode())


class SSETransport(MCPTransport):
    """Server-Sent Events，HTTP POST 发送请求，SSE 接收响应"""

    def __init__(self, url: str):
        self._url = url
        self._session: aiohttp.ClientSession | None = None

    async def connect(self) -> None:
        self._session = aiohttp.ClientSession()

    async def send(self, message: dict) -> None:
        assert self._session
        async with self._session.post(f"{self._url}/message", json=message) as resp:
            resp.raise_for_status()

    async def receive(self) -> dict:
        assert self._session
        async with self._session.get(f"{self._url}/sse") as resp:
            async for line in resp.content:
                if line.startswith(b"data: "):
                    return json.loads(line[6:])


class StreamableHTTPTransport(MCPTransport):
    """HTTP 流式传输，SSE 的升级版"""

    def __init__(self, url: str):
        self._url = url
        self._session: httpx.AsyncClient | None = None

    async def connect(self) -> None:
        self._session = httpx.AsyncClient()

    async def send(self, message: dict) -> None:
        assert self._session
        resp = await self._session.post(
            f"{self._url}/message",
            json=message,
            headers={"Content-Type": "application/json"},
        )
        resp.raise_for_status()

    async def receive(self) -> dict:
        assert self._session
        async with self._session.stream("GET", f"{self._url}/sse") as resp:
            async for line in resp.aiter_lines():
                if line.startswith("data: "):
                    return json.loads(line[6:])


class WebSocketTransport(MCPTransport):
    """WebSocket 全双工通信"""

    def __init__(self, url: str, mtls_config: dict | None = None):
        self._url = url
        self._mtls_config = mtls_config
        self._ws: websockets.WebSocketClientProtocol | None = None

    async def connect(self) -> None:
        extra_headers = {}
        if self._mtls_config:
            ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
            ssl_context.load_cert_chain(
                self._mtls_config["cert_path"],
                self._mtls_config["key_path"],
            )
            extra_headers["ssl"] = ssl_context
        self._ws = await websockets.connect(self._url, **extra_headers)

    async def send(self, message: dict) -> None:
        assert self._ws
        await self._ws.send(json.dumps(message))

    async def receive(self) -> dict:
        assert self._ws
        data = await self._ws.recv()
        return json.loads(data)
```

```
┌──────────────────────────────────────────────────────────────┐
│  四种传输方式                                                │
│                                                              │
│  1. StdioTransport                                          │
│     ├── 本地子进程通信                                       │
│     ├── 通过 stdin/stdout 交换 JSON-RPC 消息                 │
│     ├── 适合: 本地工具 (如文件系统操作)                      │
│     └── 配置: {"command": "node", "args": ["server.js"]}    │
│                                                              │
│  2. SSETransport                                            │
│     ├── Server-Sent Events                                   │
│     ├── HTTP POST 发送请求，SSE 接收响应                     │
│     ├── 适合: 远程 API 服务                                  │
│     └── 配置: {"url": "https://api.example.com/mcp"}        │
│                                                              │
│  3. StreamableHTTPTransport                                  │
│     ├── HTTP 流式传输                                        │
│     ├── 更高效的替代方案 (SSE 的升级版)                      │
│     └── 适合: 高吞吐量远程服务                               │
│                                                              │
│  4. WebSocketTransport                                      │
│     ├── WebSocket 全双工通信                                 │
│     ├── 适合: 需要双向实时通信的场景                          │
│     └── 支持 mTLS 加密                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. MCP 工具注册流程

```python
from dataclasses import dataclass, field

@dataclass
class MCPToolDefinition:
    name: str
    description: str
    input_schema: dict[str, Any]

@dataclass
class MCPServer:
    name: str
    transport: MCPTransport
    tools: list[MCPToolDefinition] = field(default_factory=list)
    resources: list[MCPResource] = field(default_factory=list)

async def discover_tools(server: MCPServer) -> list[MCPToolDefinition]:
    """调用 tools/list 获取工具列表"""
    response = await server.transport.send({
        "jsonrpc": "2.0",
        "method": "tools/list",
        "id": 1,
    })
    result = await server.transport.receive()
    tools = []
    for tool_info in result.get("result", {}).get("tools", []):
        tools.append(MCPToolDefinition(
            name=tool_info["name"],
            description=tool_info["description"],
            input_schema=tool_info["inputSchema"],
        ))
    server.tools = tools
    return tools

def register_mcp_tools(server: MCPServer) -> list[Tool]:
    """将 MCP 工具注册为 Tool 接口"""
    tools = []
    for tool_def in server.tools:
        tool = build_tool(
            name=f"mcp__{server.name}__{tool_def.name}",
            input_schema=convert_mcp_schema_to_pydantic(tool_def.input_schema),
            is_mcp=True,
            async def call(args, ctx, can_use_tool, parent_msg, on_progress):
                result = await call_mcp_tool(server, tool_def.name, args)
                return ToolResult(data=result),
            description=lambda input, opts: tool_def.description,
        )
        tools.append(tool)
    return tools
```

---

## 4. MCPTool — 桥接 Tool 接口

```python
async def call_mcp_tool(
    server: MCPServer,
    tool_name: str,
    arguments: dict,
) -> Any:
    """通过 MCP 协议调用远程工具"""
    await server.transport.send({
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {
            "name": tool_name,
            "arguments": arguments,
        },
        "id": 2,
    })
    response = await server.transport.receive()
    return response.get("result", {}).get("content", [])
```

---

## 5. OAuth 认证

```python
import time

@dataclass
class OAuthToken:
    access_token: str
    refresh_token: str
    expires_at: float

    def is_expired(self) -> bool:
        return time.time() >= self.expires_at

async def handle_oauth_401_error(
    server: MCPServer,
    token_store: dict[str, OAuthToken],
) -> OAuthToken | None:
    """处理 MCP Server 返回的 401 错误"""
    token = token_store.get(server.name)
    if token and not token.is_expired():
        return token

    new_token = await refresh_oauth_token(server.name, token)
    if new_token:
        token_store[server.name] = new_token
        return new_token

    return None

async def refresh_oauth_token(
    server_name: str,
    old_token: OAuthToken | None,
) -> OAuthToken | None:
    """刷新 OAuth token"""
    claude_tokens = await get_claude_ai_oauth_tokens()
    server_token = claude_tokens.get(server_name)
    if not server_token:
        return None

    async with aiohttp.ClientSession() as session:
        resp = await session.post(
            "https://claude.ai/oauth/token",
            json={
                "grant_type": "refresh_token",
                "refresh_token": server_token.refresh_token,
            },
        )
        if resp.status == 200:
            data = await resp.json()
            return OAuthToken(
                access_token=data["access_token"],
                refresh_token=data.get("refresh_token", server_token.refresh_token),
                expires_at=time.time() + data["expires_in"],
            )
    return None
```

---

## 6. 内容验证与截断

```python
def get_content_size_estimate(content: Any) -> int:
    """估算 MCP 内容大小"""
    if isinstance(content, str):
        return len(content.encode("utf-8"))
    if isinstance(content, list):
        return sum(get_content_size_estimate(item) for item in content)
    if isinstance(content, dict):
        total = 0
        for key, value in content.items():
            total += len(str(key))
            total += get_content_size_estimate(value)
        return total
    return len(str(content))

MAX_MCP_CONTENT_SIZE = 100_000

def truncate_mcp_content_if_needed(content: Any) -> Any:
    """截断过大的 MCP 内容"""
    size = get_content_size_estimate(content)
    if size <= MAX_MCP_CONTENT_SIZE:
        return content

    if isinstance(content, str):
        return content[:MAX_MCP_CONTENT_SIZE] + "\n...[truncated]"
    if isinstance(content, list):
        truncated = []
        current_size = 0
        for item in content:
            item_size = get_content_size_estimate(item)
            if current_size + item_size > MAX_MCP_CONTENT_SIZE:
                break
            truncated.append(item)
            current_size += item_size
        return truncated
    return content

def recursively_sanitize_unicode(text: str) -> str:
    """清理危险 Unicode 字符"""
    dangerous_ranges = [
        (0x202E, 0x202E),  # Right-to-Left Override
        (0x200F, 0x200F),  # Right-to-Left Mark
        (0x200B, 0x200F),  # Zero-Width characters
    ]
    result = []
    for char in text:
        code = ord(char)
        if any(start <= code <= end for start, end in dangerous_ranges):
            continue
        result.append(char)
    return "".join(result)
```

---

## 7. 代理支持

```python
def get_proxy_fetch_options(url: str) -> dict:
    """获取代理配置"""
    proxy_url = os.environ.get("HTTPS_PROXY") or os.environ.get("HTTP_PROXY")
    if not proxy_url:
        return {}

    return {
        "proxy": proxy_url,
        "ssl": ssl.create_default_context(),
    }

def get_websocket_proxy_url(url: str) -> str | None:
    """获取 WebSocket 代理 URL"""
    ws_proxy = os.environ.get("WSS_PROXY") or os.environ.get("WS_PROXY")
    return ws_proxy
```

---

## 8. 面试要点

1. **MCP 解决了什么问题？** — 标准化工具扩展。没有 MCP，每个外部工具都需要自定义集成代码。有了 MCP，任何实现 MCP 协议的服务都可以自动被 Claude Code 使用。

2. **为什么有四种传输方式？** — 不同场景需要不同的通信模式。本地工具用 stdio 最高效，远程 API 用 SSE/HTTP 更合适，需要双向实时通信用 WebSocket。

3. **MCP 工具如何与内置工具共存？** — MCP 工具被包装为实现了 Tool Protocol 的对象，命名格式为 `mcp__{server}__{tool}`。它们被加入同一个 tools 列表，LLM 可以无缝调用。

4. **动态刷新为什么重要？** — MCP Server 可能在会话期间连接或断开。每轮查询之间调用 `refresh_tools()` 确保新连接的工具立即可用。

5. **Python 中如何实现 StdioTransport？** — 使用 `asyncio.create_subprocess_exec` 创建子进程，通过 `stdin.write` 发送 JSON-RPC 消息，通过 `stdout.readline` 接收响应。这是 Python 异步子进程通信的标准模式。
