# 01 - 工具抽象核心 (Tool.ts)

> 源文件: `src/Tool.ts` (~792 行)
> 这是 Claude Code 整个工具系统的基石，定义了 Tool 接口、ToolUseContext、buildTool 工厂函数等核心类型。

---

## 1. 为什么 Tool.ts 是最重要的文件？

Claude Code 有 40+ 个内置工具（Bash、Read、Write、Edit、Grep、Glob、WebFetch、AgentTool...），每个工具都要处理：

- 输入验证（Zod Schema）
- 权限检查（allow/deny/ask）
- 并发安全性（能否并行执行）
- 进度报告（流式 UI 更新）
- 结果渲染（React 组件）
- 错误处理（自定义错误 UI）

`Tool.ts` 用一个统一的泛型接口解决了所有这些问题。

---

## 2. Tool 接口全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Tool<Input, Output, P>                       │
│                                                                 │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │      核心执行方法       │  │        Schema 定义             │  │
│  │                        │  │                              │  │
│  │  call(args, ctx, ...)  │  │  inputSchema: ZodSchema      │  │
│  │    → Promise<ToolResult>│  │  inputJSONSchema?: JSON      │  │
│  │                        │  │  outputSchema?: ZodSchema     │  │
│  │  description(input)    │  │                              │  │
│  │    → Promise<string>   │  │  inputsEquivalent?(a, b)     │  │
│  │                        │  │    → boolean                 │  │
│  │  prompt(options)       │  │                              │  │
│  │    → Promise<string>   │  │  validateInput?(input, ctx)  │  │
│  │                        │  │    → ValidationResult        │  │
│  │  mapToolResultTo...    │  │                              │  │
│  │    → ToolResultBlock   │  └──────────────────────────────┘  │
│  └────────────────────────┘                                     │
│                                                                 │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │    安全 & 并发控制       │  │        UI 渲染                 │  │
│  │                        │  │                              │  │
│  │  checkPermissions()    │  │  renderToolUseMessage()     │  │
│  │    → PermissionResult  │  │  renderToolResultMessage()  │  │
│  │                        │  │  renderToolUseProgress()    │  │
│  │  isConcurrencySafe()   │  │  renderToolUseRejected()    │  │
│  │    → boolean           │  │  renderToolUseError()       │  │
│  │                        │  │  renderGroupedToolUse()     │  │
│  │  isReadOnly()          │  │  renderToolUseTag()         │  │
│  │    → boolean           │  │  renderToolUseQueued()      │  │
│  │                        │  │                              │  │
│  │  isDestructive?()      │  │  userFacingName()           │  │
│  │    → boolean           │  │  getActivityDescription()   │  │
│  │                        │  │  getToolUseSummary()        │  │
│  │  interruptBehavior?()  │  │  extractSearchText()        │  │
│  │    → 'cancel'|'block'  │  │                              │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
│                                                                 │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │       元数据            │  │       行为标记                 │  │
│  │                        │  │                              │  │
│  │  name: string          │  │  isMcp?: boolean            │  │
│  │  aliases?: string[]    │  │  isLsp?: boolean            │  │
│  │  searchHint?: string   │  │  shouldDefer?: boolean      │  │
│  │  maxResultSizeChars    │  │  alwaysLoad?: boolean       │  │
│  │  strict?: boolean      │  │  isOpenWorld?()             │  │
│  │                        │  │  requiresUserInteraction?() │  │
│  │  mcpInfo?              │  │  isTransparentWrapper?()   │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心类型详解

### 3.1 ToolResult — 工具执行结果

```typescript
// src/Tool.ts:321
export type ToolResult<T> = {
  data: T                                          // 工具返回的实际数据
  newMessages?: (UserMessage | AssistantMessage     // 可选：产生的新消息
    | AttachmentMessage | SystemMessage)[]
  contextModifier?: (                              // 可选：修改执行上下文
    context: ToolUseContext
  ) => ToolUseContext
  mcpMeta?: {                                      // MCP 协议透传
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

**关键设计**: `contextModifier` 允许工具执行后修改后续工具的执行上下文。例如，MCP 工具连接后可以将新的工具注册到上下文中。

### 3.2 ToolUseContext — 工具执行上下文

这是传递给每个工具的"环境"，包含了工具执行所需的一切：

```
┌──────────────────────────────────────────────────────────────┐
│                    ToolUseContext                              │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  options: {                                            │  │
│  │    commands: Command[]         // 所有可用命令           │  │
│  │    tools: Tools               // 所有可用工具           │  │
│  │    mainLoopModel: string      // 当前使用的模型          │  │
│  │    mcpClients: MCPServer[]    // MCP 服务器连接         │  │
│  │    agentDefinitions: ...      // Agent 定义列表         │  │
│  │    maxBudgetUsd?: number      // 预算上限               │  │
│  │    customSystemPrompt?: ...   // 自定义系统提示          │  │
│  │    refreshTools?: () => Tools // 刷新工具列表           │  │
│  │  }                                                     │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  abortController: AbortController  // 中断控制器         │  │
│  │  readFileState: FileStateCache      // 文件读取缓存      │  │
│  │  getAppState(): AppState            // 获取全局状态      │  │
│  │  setAppState(f): void              // 更新全局状态      │  │
│  │  messages: Message[]               // 当前对话消息      │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  agentId?: AgentId               // 子 Agent ID        │  │
│  │  agentType?: string               // 子 Agent 类型      │  │
│  │  queryTracking?: {                // 查询链追踪         │  │
│  │    chainId: string                                     │  │
│  │    depth: number                                      │  │
│  │  }                                                     │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  setInProgressToolUseIDs(f)    // 设置正在执行的工具     │  │
│  │  setStreamMode(mode)           // 设置流式模式          │  │
│  │  addNotification(notif)        // 添加通知             │  │
│  │  sendOSNotification(opts)      // 发送系统通知          │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 3.3 ToolPermissionContext — 权限上下文

```typescript
// src/Tool.ts:123
export type ToolPermissionContext = {
  mode: PermissionMode           // 'default' | 'plan' | 'bypassPermissions' | 'auto'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource   // 自动放行规则
  alwaysDenyRules: ToolPermissionRulesBySource    // 自动拒绝规则
  alwaysAskRules: ToolPermissionRulesBySource     // 总是询问规则
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  shouldAvoidPermissionPrompts?: boolean  // 后台 Agent 自动拒绝
  awaitAutomatedChecksBeforeDialog?: boolean  // 等待自动检查
  prePlanMode?: PermissionMode  // plan mode 之前的权限模式
}
```

---

## 4. buildTool 工厂函数 — 安全默认值模式

这是整个工具系统最优雅的设计之一：

```typescript
// src/Tool.ts:757
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,   // ← 默认不安全！
  isReadOnly: (_input?) => false,           // ← 默认可写！
  isDestructive: (_input?) => false,        // ← 默认非破坏性
  checkPermissions: (input, _ctx?) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?) => '',   // ← 默认跳过分类器
  userFacingName: (_input?) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,  // 覆盖默认值
    ...def,                          // 开发者定义覆盖一切
  } as BuiltTool<D>
}
```

### 设计哲学：Fail-Closed（安全优先）

```
┌──────────────────────────────────────────────────────────┐
│              buildTool 默认值策略                           │
│                                                          │
│  属性                默认值         设计理由              │
│  ─────────────────────────────────────────────────────── │
│  isConcurrencySafe   false        不声明 = 不安全          │
│  isReadOnly          false        不声明 = 可写            │
│  isDestructive       false        不声明 = 非破坏性        │
│  isEnabled           true         不声明 = 启用            │
│  checkPermissions    allow        交给通用权限系统处理      │
│  toAutoClassifier    ''           不声明 = 跳过安全分类    │
│                                                          │
│  核心原则: 宁可保守，不可激进                               │
│  • 新工具默认不能并行（避免竞态条件）                       │
│  • 新工具默认可写（避免误判为只读导致安全问题）              │
│  • 安全相关的工具必须显式声明安全属性                       │
└──────────────────────────────────────────────────────────┘
```

---

## 5. 工具查找机制

```typescript
// src/Tool.ts:348
export function toolMatchesName(
  tool: { name: string; aliases?: string[] },
  name: string,
): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}

export function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}
```

工具支持别名（aliases），用于向后兼容。例如工具改名后，旧名字仍然可以通过别名找到。

---

## 6. ToolDef vs Tool — 简化工具定义

`ToolDef` 是 `Tool` 的简化版，7 个常用方法变为可选：

```typescript
// src/Tool.ts:707
type DefaultableToolKeys =
  | 'isEnabled'
  | 'isConcurrencySafe'
  | 'isReadOnly'
  | 'isDestructive'
  | 'checkPermissions'
  | 'toAutoClassifierInput'
  | 'userFacingName'

// ToolDef = Tool 减去 DefaultableToolKeys（这些变成可选）
export type ToolDef<Input, Output, P> =
  Omit<Tool<Input, Output, P>, DefaultableToolKeys> &
  Partial<Pick<Tool<Input, Output, P>, DefaultableToolKeys>>
```

这意味着定义一个新工具时，你只需要实现核心方法，其余的由 `buildTool()` 自动填充：

```
┌──────────────────────────────────────────────────────────┐
│  定义工具的两种方式                                       │
│                                                          │
│  方式 1: 直接实现 Tool (需要 30+ 个方法)                   │
│  ├── 繁琐，容易遗漏                                       │
│  └── 不推荐                                              │
│                                                          │
│  方式 2: 使用 buildTool() + ToolDef (只需核心方法)         │
│  ├── 只需实现: call, description, prompt,                 │
│  │           inputSchema, renderToolUseMessage,           │
│  │           mapToolResultToToolResultBlockParam          │
│  ├── 7 个安全方法自动填充默认值                            │
│  └── ✅ 推荐，所有 60+ 工具都使用这种方式                  │
└──────────────────────────────────────────────────────────┘
```

---

## 7. 进度报告系统

工具可以实时报告执行进度：

```typescript
// src/Tool.ts:338
export type ToolCallProgress<P extends ToolProgressData> = (
  progress: ToolProgress<P>,
) => void

// 进度数据类型（定义在 types/tools.ts）
type ToolProgressData =
  | BashProgress          // Bash 命令执行进度
  | MCPProgress           // MCP 工具调用进度
  | AgentToolProgress     // 子 Agent 执行进度
  | SkillToolProgress     // 技能执行进度
  | TaskOutputProgress    // 任务输出进度
  | WebSearchProgress     // 网页搜索进度
  | REPLToolProgress      // REPL 执行进度
```

进度通过 `onProgress` 回调传递，最终渲染为终端 UI 中的 spinner 和状态文本。

---

## 8. 延迟加载工具 (Deferred Tools)

```typescript
// src/Tool.ts:442
readonly shouldDefer?: boolean    // 延迟加载
readonly alwaysLoad?: boolean     // 始终加载
```

```
┌──────────────────────────────────────────────────────────┐
│  工具加载策略                                              │
│                                                          │
│  shouldDefer: true                                       │
│  ├── 工具 schema 不在初始 prompt 中                       │
│  ├── 模型需要先调用 ToolSearch 才能发现                    │
│  └── 用途: 减少初始 prompt 的 token 消耗                   │
│                                                          │
│  alwaysLoad: true                                        │
│  ├── 工具 schema 始终在初始 prompt 中                     │
│  └── 用途: 模型必须在第一轮就看到的工具                     │
│                                                          │
│  默认 (两者都不设置)                                       │
│  ├── 由 ToolSearch 功能开关决定                           │
│  └── 大多数工具使用默认行为                                │
└──────────────────────────────────────────────────────────┘
```

---

## 9. 实战：如何定义一个新工具

```typescript
import { buildTool } from '../Tool.js'
import { z } from 'zod/v4'

export const MyTool = buildTool({
  name: 'MyTool',
  searchHint: 'custom operation for X',

  // 1. 输入 Schema (Zod 验证)
  inputSchema: z.object({
    query: z.string().describe('搜索查询'),
    maxResults: z.number().optional().default(10),
  }),

  // 2. 核心执行逻辑
  async call(args, context, canUseTool, parentMessage, onProgress) {
    onProgress?.({ toolUseID: 'xxx', data: { type: 'progress', percent: 50 } })
    const results = await doSearch(args.query, args.maxResults)
    return { data: results }
  },

  // 3. 给模型的描述（动态，可基于输入调整）
  async description(input, options) {
    return `搜索 "${input.query}"，最多返回 ${input.maxResults} 个结果`
  },

  // 4. 系统提示中的工具说明
  async prompt(options) {
    return '使用 MyTool 来搜索自定义数据源...'
  },

  // 5. 结果序列化
  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return {
      type: 'tool_result',
      tool_use_id: toolUseID,
      content: JSON.stringify(content),
    }
  },

  // 6. UI 渲染
  renderToolUseMessage(input, options) {
    return <Text>搜索: {input.query}</Text>
  },

  renderToolResultMessage(content, progressMessages, options) {
    return <Text>找到 {content.length} 个结果</Text>
  },

  // 7. 安全声明（覆盖默认值）
  isConcurrencySafe: () => true,    // 只读，可并行
  isReadOnly: () => true,            // 不修改文件系统
  maxResultSizeChars: 100_000,       // 结果最大 100K 字符
})
```

---

## 10. 关键设计模式总结

| 模式 | 代码位置 | 说明 |
|------|---------|------|
| **泛型三层抽象** | `Tool<Input, Output, P>` | 输入、输出、进度类型完全参数化 |
| **安全默认值** | `TOOL_DEFAULTS` | Fail-closed，必须显式声明安全属性 |
| **工厂函数** | `buildTool()` | 自动填充默认值，简化工具定义 |
| **别名系统** | `aliases` + `toolMatchesName()` | 工具改名后保持向后兼容 |
| **延迟加载** | `shouldDefer` / `alwaysLoad` | 减少 prompt token 消耗 |
| **上下文修改** | `contextModifier` | 工具可以修改后续工具的执行环境 |
| **进度回调** | `ToolCallProgress<P>` | 流式报告执行进度 |
| **React 渲染** | 6 个 render 方法 | 每个工具自定义 UI 展示 |

---

## 11. 面试要点

1. **为什么 isConcurrencySafe 默认 false？** — 安全优先。如果默认 true，一个有副作用的工具被并行执行可能导致竞态条件和数据损坏。开发者必须显式声明工具是并发安全的。

2. **ToolUseContext 为什么这么大？** — 它是工具执行的"万能环境"，包含了状态管理、权限、MCP 连接、Agent 定义等所有运行时信息。这避免了全局变量的使用，使工具可以安全地在子 Agent 中复用。

3. **buildTool 的类型系统如何工作？** — `BuiltTool<D>` 使用 TypeScript 的条件类型，对于每个 DefaultableKey：如果 D 提供了就用 D 的，否则用默认值。运行时是简单的 `{ ...DEFAULTS, ...def }` 展开。

4. **maxResultSizeChars 的作用？** — 防止工具返回超大结果撑爆上下文窗口。超过限制的结果会被持久化到磁盘，模型只看到预览和文件路径。Read 工具设为 Infinity 因为它有自己的内部分页机制。
