# 07 - MCP 协议集成 (mcp/client.ts)

> 源文件: `src/services/mcp/client.ts`
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

```typescript
// src/services/mcp/client.ts:7-21
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'
import { WebSocketTransport } from '../../utils/mcpWebSocketTransport.js'
```

```
┌──────────────────────────────────────────────────────────────┐
│  四种传输方式                                                │
│                                                              │
│  1. StdioClientTransport                                    │
│     ├── 本地子进程通信                                       │
│     ├── 通过 stdin/stdout 交换 JSON-RPC 消息                 │
│     ├── 适合: 本地工具 (如文件系统操作)                      │
│     └── 配置: { command: "node", args: ["server.js"] }      │
│                                                              │
│  2. SSEClientTransport                                      │
│     ├── Server-Sent Events                                   │
│     ├── HTTP POST 发送请求，SSE 接收响应                     │
│     ├── 适合: 远程 API 服务                                  │
│     └── 配置: { url: "https://api.example.com/mcp" }        │
│                                                              │
│  3. StreamableHTTPClientTransport                            │
│     ├── HTTP 流式传输                                        │
│     ├── 更高效的替代方案 (SSE 的升级版)                      │
│     └── 适合: 高吞吐量远程服务                               │
│                                                              │
│  4. WebSocketTransport (自定义实现)                           │
│     ├── WebSocket 全双工通信                                 │
│     ├── 适合: 需要双向实时通信的场景                          │
│     └── 支持 mTLS 加密                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. MCP 工具注册流程

```
┌──────────────────────────────────────────────────────────────┐
│  MCP 工具注册流程                                            │
│                                                              │
│  1. 连接 MCP Server                                         │
│     ├── 使用对应的 Transport 建立连接                        │
│     ├── 发送 initialize 请求                                  │
│     └── 交换能力声明 (capabilities)                          │
│                                                              │
│  2. 发现工具                                                 │
│     ├── 调用 tools/list 获取工具列表                         │
│     ├── 每个工具包含: name, description, inputSchema        │
│     └── 使用 ListToolsResultSchema 验证响应                 │
│                                                              │
│  3. 注册为 MCPTool                                           │
│     ├── 每个 MCP 工具 → 一个 MCPTool 实例                   │
│     ├── MCPTool 实现了 Tool 接口                             │
│     └── 加入 tools 列表                                      │
│                                                              │
│  4. 动态刷新                                                 │
│     ├── refreshTools() 回调在每轮之间检查                    │
│     ├── 新连接的 MCP Server 的工具自动可用                   │
│     └── query.ts:1660 调用 refreshTools                     │
│                                                              │
│  5. 工具调用                                                 │
│     ├── Claude 调用 MCPTool.call()                          │
│     ├── MCPTool 通过 MCP 协议转发到 Server                   │
│     ├── Server 执行并返回结果                                │
│     └── MCPTool 将结果转换为 ToolResult                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. MCPTool — 桥接 Tool 接口

```typescript
// MCP 工具被包装为 Tool 接口
export const MCPTool = buildTool({
  name: `mcp__${serverName}__${toolName}`,
  inputSchema: convertMCPSchemaToZod(tool.inputSchema),
  mcpInfo: { serverName, toolName },

  async call(args, context, canUseTool, parentMessage, onProgress) {
    // 通过 MCP 协议调用远程工具
    const result = await mcpClient.callTool({
      name: toolName,
      arguments: args,
    })
    return { data: result }
  },

  isMcp: true,  // 标记为 MCP 工具
})
```

---

## 5. OAuth 认证

```typescript
// src/services/mcp/client.ts:63-64
import {
  checkAndRefreshOAuthTokenIfNeeded,
  getClaudeAIOAuthTokens,
  handleOAuth401Error,
} from '../../utils/auth.js'
```

```
┌──────────────────────────────────────────────────────────┐
│  MCP OAuth 认证流程                                       │
│                                                          │
│  1. MCP Server 返回 401 错误                              │
│  2. handleOAuth401Error() 处理                            │
│  3. 检查是否有缓存的 OAuth token                          │
│  4. 如果 token 过期 → checkAndRefreshOAuthTokenIfNeeded  │
│  5. 用新 token 重试请求                                   │
│  6. 如果刷新失败 → 提示用户重新授权                       │
│                                                          │
│  Token 来源:                                              │
│  ├── getClaudeAIOAuthTokens() — 从 claude.ai 获取         │
│  └── getOauthConfig() — OAuth 配置                        │
└──────────────────────────────────────────────────────────┘
```

---

## 6. 内容验证与截断

```typescript
// src/services/mcp/client.ts:87
import {
  getContentSizeEstimate,
  mcpContentNeedsTruncation,
  truncateMcpContentIfNeeded,
} from '../../utils/mcpValidation.js'
```

```
┌──────────────────────────────────────────────────────────┐
│  MCP 内容处理                                              │
│                                                          │
│  大内容处理:                                              │
│  ├── getContentSizeEstimate() — 估算内容大小              │
│  ├── mcpContentNeedsTruncation() — 判断是否需要截断       │
│  ├── truncateMcpContentIfNeeded() — 截断过大内容          │
│  └── persistBinaryContent() — 持久化二进制内容到磁盘      │
│                                                          │
│  Unicode 安全:                                            │
│  └── recursivelySanitizeUnicode() — 清理危险 Unicode 字符│
│                                                          │
│  图片处理:                                                │
│  └── maybeResizeAndDownsampleImageBuffer() — 调整图片大小│
└──────────────────────────────────────────────────────────┘
```

---

## 7. 代理支持

```typescript
// src/services/mcp/client.ts:95-96
import {
  getProxyFetchOptions,
  getWebSocketProxyAgent,
  getWebSocketProxyUrl,
} from '../../utils/proxy.js'
```

MCP 客户端支持通过 HTTP/SOCKS 代理连接远程 MCP Server。

---

## 8. MCP 资源系统

除了工具，MCP 还支持资源 (Resources)：

```
┌──────────────────────────────────────────────────────────┐
│  MCP Resources                                             │
│                                                          │
│  ListMcpResourcesTool — 列出可用资源                      │
│  ReadMcpResourceTool — 读取资源内容                        │
│  McpAuthTool — MCP 认证管理                               │
│                                                          │
│  资源类型:                                                │
│  ├── 文件资源 — 服务器上的文件                             │
│  ├── API 响应 — 缓存的 API 响应                           │
│  └── 自定义数据 — 服务器暴露的任何数据                     │
│                                                          │
│  用途:                                                   │
│  ├── Claude 可以读取 MCP Server 暴露的数据                │
│  ├── 补充文件系统工具无法访问的信息                        │
│  └── 通过 ListResourcesResultSchema 验证响应              │
└──────────────────────────────────────────────────────────┘
```

---

## 9. 面试要点

1. **MCP 解决了什么问题？** — 标准化工具扩展。没有 MCP，每个外部工具都需要自定义集成代码。有了 MCP，任何实现 MCP 协议的服务都可以自动被 Claude Code 使用。

2. **为什么有四种传输方式？** — 不同场景需要不同的通信模式。本地工具用 stdio 最高效，远程 API 用 SSE/HTTP 更合适，需要双向实时通信用 WebSocket。

3. **MCP 工具如何与内置工具共存？** — MCP 工具被包装为 MCPTool，实现了和内置工具相同的 Tool 接口。它们被加入同一个 tools 列表，LLM 可以无缝调用。

4. **动态刷新为什么重要？** — MCP Server 可能在会话期间连接或断开。每轮查询之间调用 refreshTools() 确保新连接的工具立即可用。
