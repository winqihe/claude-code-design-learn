# 07 - MCP 协议：Claude 的万能插头

## 一句话总结

MCP 就是一个"万能插头"，让 Claude 能连接任何外部工具——数据库、API、文件系统、第三方服务——不需要为每个工具写专门的代码。

---

## 通俗理解

想象 Claude 是一个电器：

```
没有 MCP:
  每连接一个新设备，就要专门做一根线
  ├── 连数据库 → 写专门的数据库集成代码
  ├── 连 Slack → 写专门的 Slack 集成代码
  ├── 连 GitHub → 写专门的 GitHub 集成代码
  └── 每个都要维护，累死

有 MCP:
  一个万能插头搞定所有设备
  ├── 数据库装了 MCP Server → 自动可用
  ├── Slack 装了 MCP Server → 自动可用
  ├── GitHub 装了 MCP Server → 自动可用
  └── Claude 自动发现并使用这些工具
```

MCP = **Model Context Protocol** = 模型上下文协议
就像 USB 统一了所有外设接口，MCP 统一了 AI 和外部工具的接口。

---

## 四种连接方式

```
1. Stdio (本地连接)
   ┌──────────┐    stdin/stdout    ┌──────────┐
   │  Claude  │◄════════════════►│  本地工具  │
   └──────────┘                    └──────────┘
   就像用 USB 线直连，最快最稳

2. SSE (远程连接 - 旧版)
   ┌──────────┐    HTTP + SSE     ┌──────────┐
   │  Claude  │◄════════════════►│  远程 API │
   └──────────┘                    └──────────┘
   就像用 WiFi，方便但稍慢

3. StreamableHTTP (远程连接 - 新版)
   ┌──────────┐    HTTP 流式       ┌──────────┐
   │  Claude  │◄════════════════►│  远程服务 │
   └──────────┘                    └──────────┘
   就像 WiFi 6，比旧版更快

4. WebSocket (双向实时)
   ┌──────────┐    WebSocket      ┌──────────┐
   │  Claude  │◄════════════════►│  实时服务 │
   └──────────┘                    └──────────┘
   就像蓝牙，双向实时通信
```

---

## 工作流程

```
第一步: 连接
  Claude 启动时，读取配置文件中的 MCP Server 列表
  → 逐个连接

第二步: 发现
  向每个 Server 问: "你有什么工具？"
  → Server 返回工具列表 (名称、描述、参数格式)

第三步: 注册
  把 MCP 工具包装成 Claude 的内置工具格式
  → 命名: mcp__服务器名__工具名
  → 加入工具列表

第四步: 使用
  Claude 想用某个工具时:
  → 通过 MCP 协议发送请求到 Server
  → Server 执行并返回结果
  → Claude 看到结果，继续思考
```

---

## Python 实现

```python
class StdioTransport:
    """本地进程通信"""

    async def connect(self, command, args):
        self.process = await asyncio.create_subprocess_exec(
            command, *args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
        )

    async def send(self, message):
        self.process.stdin.write(json.dumps(message).encode() + b"\n")

    async def receive(self):
        line = await self.process.stdout.readline()
        return json.loads(line)


class SSETransport:
    """远程 HTTP 通信"""

    async def connect(self, url):
        self.session = aiohttp.ClientSession()
        self.url = url

    async def send(self, message):
        await self.session.post(f"{self.url}/message", json=message)

    async def receive(self):
        async with self.session.get(f"{self.url}/sse") as resp:
            async for line in resp.content:
                if line.startswith(b"data: "):
                    return json.loads(line[6:])
```

---

## OAuth 认证：安全的远程连接

连接远程 MCP Server 可能需要认证（比如访问 GitHub API 需要 token）。

```
第一次连接:
  → Server 返回 401 (未认证)
  → Claude 引导用户去授权页面
  → 用户授权后获得 token
  → 保存 token，以后自动使用

Token 过期:
  → Server 返回 401
  → 自动用 refresh token 刷新
  → 刷新失败 → 让用户重新授权
```

---

## 内容安全：防止 MCP 返回垃圾数据

MCP Server 可能返回超大内容，撑爆 Claude 的上下文窗口。

```python
MAX_SIZE = 100_000  # 100KB

def truncate_if_needed(content):
    if len(content) > MAX_SIZE:
        return content[:MAX_SIZE] + "\n...[内容已截断]"
    return content
```

就像邮箱限制附件大小，超过就拒绝或截断。

---

## 一张图总结

```
┌──────────────────────────────────────────────────────┐
│              MCP = AI 的 USB 接口                     │
│                                                      │
│  ┌──────────────────────────────────────────┐        │
│  │           Claude Code (Host)              │        │
│  │                                          │        │
│  │  内置工具    MCP 工具 (自动发现)           │        │
│  │  ├── Read    ├── mcp__github__create_issue│        │
│  │  ├── Write   ├── mcp__db__query          │        │
│  │  ├── Bash    ├── mcp__slack__send_message│        │
│  │  └── Grep    └── mcp__weather__get       │        │
│  └────┬──────────┬──────────┬───────────────┘        │
│       │          │          │                         │
│  ┌────▼───┐ ┌────▼───┐ ┌───▼────┐                    │
│  │GitHub  │ │Database│ │Slack   │  ← MCP Servers     │
│  │Server  │ │Server  │ │Server  │                    │
│  └────────┘ └────────┘ └────────┘                    │
│                                                      │
│  关键特性:                                            │
│  ├── 4 种传输方式 (本地/远程/流式/WebSocket)          │
│  ├── 自动发现工具 (不需要手动配置)                    │
│  ├── OAuth 认证 (安全连接远程服务)                    │
│  └── 内容截断 (防止撑爆上下文)                        │
└──────────────────────────────────────────────────────┘
```

---

## 面试怎么答

**Q: MCP 解决了什么问题？**
A: 标准化工具扩展。没有 MCP，每个外部工具都需要写专门的集成代码。有了 MCP，任何实现 MCP 协议的服务都能自动被 Claude 使用。就像 USB 统一了外设接口。

**Q: 为什么有四种传输方式？**
A: 不同场景需要不同通信模式。本地工具用 stdio（最快），远程 API 用 HTTP（方便），实时通信用 WebSocket（双向）。就像你用 USB 连鼠标，用 WiFi 连打印机，用蓝牙连耳机。
