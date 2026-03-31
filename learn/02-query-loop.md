# 02 - Agent 运行循环 (query.ts)

> 源文件: `src/query.ts` (~1729 行)
> 这是 Claude Code 的心脏 — 一个 `while(true)` 无限循环，驱动着 LLM 推理和工具执行的交替进行。

---

## 1. 什么是 Query Loop？

Query Loop 是所有 AI Agent 系统的核心模式：**LLM 思考 → 调用工具 → 获取结果 → 继续思考**。Claude Code 的实现是业界最复杂的之一，包含了上下文压缩、Token 预算、模型降级、流式工具执行等高级特性。

```
┌──────────────────────────────────────────────────────────────────┐
│                     Query Loop 整体流程                            │
│                                                                  │
│  用户输入                                                         │
│    │                                                             │
│    ▼                                                             │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    while (true) {                          │  │
│  │                                                            │  │
│  │   ① 上下文管理                                              │  │
│  │   ├── Snip (历史裁剪)                                      │  │
│  │   ├── Microcompact (微压缩)                                │  │
│  │   ├── Context Collapse (上下文折叠)                        │  │
│  │   └── Autocompact (自动压缩)                               │  │
│  │                                                            │  │
│  │   ② 调用 LLM API (流式)                                   │  │
│  │   ├── 构造请求 (messages + system + tools)                 │  │
│  │   ├── 流式接收响应                                         │  │
│  │   └── 收集 tool_use 块                                     │  │
│  │                                                            │  │
│  │   ③ 判断 stop_reason                                      │  │
│  │   ├── end_turn → 退出循环 ✅                               │  │
│  │   ├── tool_use → 执行工具 → 继续循环 🔄                    │  │
│  │   └── max_tokens → 恢复机制 → 继续循环 🔄                  │  │
│  │                                                            │  │
│  │   ④ 执行工具 (runTools)                                   │  │
│  │   ├── 只读工具 → 并行                                      │  │
│  │   └── 写工具 → 串行                                        │  │
│  │                                                            │  │
│  │   ⑤ 收集附件 (attachments)                                 │  │
│  │   ├── 文件变更通知                                         │  │
│  │   ├── 记忆文件                                             │  │
│  │   └── 技能发现                                             │  │
│  │                                                            │  │
│  │   ⑥ 安全检查                                              │  │
│  │   ├── maxTurns 检查                                       │  │
│  │   └── abort 检查                                          │  │
│  │                                                            │  │
│  │   ⑦ 更新状态 → 回到 ①                                    │  │
│  │                                                            │  │
│  │  }                                                         │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 入口函数与状态管理

### 2.1 query() — 公开入口

```typescript
// src/query.ts:219
export async function* query(
  params: QueryParams,
): AsyncGenerator<StreamEvent | Message | TombstoneMessage | ToolUseSummaryMessage, Terminal> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)

  // 循环正常结束后，通知所有消费的命令已完成
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

**设计要点**:
- 使用 `AsyncGenerator` 实现流式输出，调用者可以实时接收消息
- `yield*` 委托给内部 `queryLoop`，保持生成器链
- 返回 `Terminal` 类型，描述循环结束的原因

### 2.2 State — 跨迭代状态

```typescript
// src/query.ts:204
type State = {
  messages: Message[]                              // 对话消息列表
  toolUseContext: ToolUseContext                    // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState    // 自动压缩追踪
  maxOutputTokensRecoveryCount: number             // max_tokens 恢复次数
  hasAttemptedReactiveCompact: boolean             // 是否尝试过响应式压缩
  maxOutputTokensOverride: number | undefined      // max_tokens 覆盖值
  pendingToolUseSummary: Promise<...> | undefined  // 待处理的工具摘要
  stopHookActive: boolean | undefined              // stop hook 是否活跃
  turnCount: number                                // 当前轮次
  transition: Continue | undefined                 // 上次继续的原因
}
```

**设计要点**: 使用单个 `state` 对象，在 continue 点一次性更新所有字段，避免 9 个独立赋值：

```typescript
// 继续循环时
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,  // 重置
  hasAttemptedReactiveCompact: false,  // 重置
  transition: { reason: 'next_turn' },
}
```

---

## 3. 循环的六大阶段详解

### 阶段 ①: 上下文管理

```
┌──────────────────────────────────────────────────────────────┐
│  上下文管理管道 (按顺序执行)                                    │
│                                                              │
│  messages ──► Snip ──► Microcompact ──► Collapse ──► Autocompact │
│                                                              │
│  Snip: 裁剪过期的历史消息 (feature: HISTORY_SNIP)              │
│  ├── 删除不再需要的中间消息                                    │
│  └── 保留关键上下文                                           │
│                                                              │
│  Microcompact: 微型压缩 (cached microcompact)                  │
│  ├── 删除已缓存的工具结果                                      │
│  └── 减少 token 消耗但不丢失信息                               │
│                                                              │
│  Context Collapse: 上下文折叠 (feature: CONTEXT_COLLAPSE)      │
│  ├── 将相似消息折叠为摘要                                      │
│  └── 保留细粒度上下文而非单一摘要                               │
│                                                              │
│  Autocompact: 自动压缩                                        │
│  ├── 当 token 接近上限时触发                                   │
│  ├── 用 LLM 生成对话摘要                                      │
│  └── 替换旧消息为摘要消息                                      │
│                                                              │
│  Token Budget: Token 预算 (feature: TOKEN_BUDGET)             │
│  ├── 跟踪 token 消耗                                          │
│  └── 超预算时注入提示消息要求精简                               │
└──────────────────────────────────────────────────────────────┘
```

### 阶段 ②: 调用 LLM API

```typescript
// src/query.ts:659
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fallbackModel,
    querySource,
    // ... 更多选项
  },
})) {
  // 处理流式消息
  if (message.type === 'assistant') {
    assistantMessages.push(message)
    // 收集 tool_use 块
    const toolUseBlocks = message.message.content
      .filter(c => c.type === 'tool_use')
    if (toolUseBlocks.length > 0) {
      needsFollowUp = true  // 需要继续循环
    }
  }
}
```

**关键特性 — 模型降级 (Fallback)**:

```
┌──────────────────────────────────────────────────────────┐
│  模型降级流程                                              │
│                                                          │
│  调用 Claude Opus ──► 高负载错误 ──► 切换到 Sonnet       │
│                                                          │
│  降级时:                                                 │
│  1. 清空已收集的 assistantMessages                       │
│  2. 为已发出的 tool_use 生成空的 tool_result              │
│  3. 清除 streamingToolExecutor 的待处理结果               │
│  4. 剥离 thinking 签名块 (不同模型签名不兼容)             │
│  5. 用新模型重试整个请求                                  │
│                                                          │
│  降级最多重试 1 次 (attemptWithFallback 标志)             │
└──────────────────────────────────────────────────────────┘
```

### 阶段 ③: 判断 stop_reason

```
┌──────────────────────────────────────────────────────────────┐
│  stop_reason 处理策略                                         │
│                                                              │
│  end_turn (正常结束)                                         │
│  ├── 执行 Stop Hooks (后处理钩子)                            │
│  ├── 检查 Token Budget                                      │
│  └── return { reason: 'completed' }                         │
│                                                              │
│  tool_use (需要执行工具)                                      │
│  ├── needsFollowUp = true                                   │
│  └── 继续到阶段 ④                                           │
│                                                              │
│  max_tokens (输出被截断)                                      │
│  ├── 第一次: 升级到 64k tokens 重试                          │
│  ├── 后续: 注入恢复消息 "从中断处继续"                       │
│  ├── 最多恢复 3 次 (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT)        │
│  └── 超限: surface 错误并退出                               │
│                                                              │
│  prompt_too_long (上下文过长)                                 │
│  ├── 先尝试 Context Collapse drain (释放折叠的消息)          │
│  ├── 再尝试 Reactive Compact (响应式压缩)                    │
│  └── 都失败: surface 错误并退出                              │
│                                                              │
│  aborted (用户中断)                                          │
│  ├── 为未完成的 tool_use 生成错误 tool_result                │
│  ├── 清理 MCP 资源                                          │
│  └── return { reason: 'aborted' }                           │
└──────────────────────────────────────────────────────────────┘
```

### 阶段 ④: 执行工具

```typescript
// src/query.ts:1380
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message
    toolResults.push(...)
  }
  if (update.newContext) {
    updatedToolUseContext = update.newContext
  }
}
```

工具执行的详细策略见 [03-tool-orchestration.md](./03-tool-orchestration.md)。

### 阶段 ⑤: 收集附件

```typescript
// src/query.ts:1580
for await (const attachment of getAttachmentMessages(
  null, updatedToolUseContext, null,
  queuedCommandsSnapshot,  // 排队的命令通知
  [...messagesForQuery, ...assistantMessages, ...toolResults],
  querySource,
)) {
  yield attachment
  toolResults.push(attachment)
}
```

附件包括：
- **文件变更通知**: 工具修改了哪些文件
- **记忆文件**: CLAUDE.md 等持久化记忆
- **技能发现**: 基于当前上下文发现的可用技能
- **任务通知**: 后台任务完成的通知

### 阶段 ⑥: 安全检查

```typescript
// src/query.ts:1705
if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({
    type: 'max_turns_reached',
    maxTurns,
    turnCount: nextTurnCount,
  })
  return { reason: 'max_turns', turnCount: nextTurnCount }
}
```

---

## 4. QueryChain — 查询链追踪

```typescript
// src/query.ts:347
const queryTracking = toolUseContext.queryTracking
  ? {
      chainId: toolUseContext.queryTracking.chainId,
      depth: toolUseContext.queryTracking.depth + 1,
    }
  : {
      chainId: deps.uuid(),
      depth: 0,
    }
```

```
┌──────────────────────────────────────────────────────────┐
│  Query Chain 追踪                                          │
│                                                          │
│  主 Agent (depth=0, chainId=abc)                         │
│  ├── 调用 AgentTool → 子 Agent (depth=1, chainId=abc)    │
│  │   └── 调用 AgentTool → 孙 Agent (depth=2, chainId=abc)│
│  └── 调用 AgentTool → 子 Agent (depth=1, chainId=abc)    │
│                                                          │
│  用途:                                                   │
│  • 分析日志中追踪请求链路                                  │
│  • 区分主线程和子 Agent 的行为                            │
│  • headless 延迟追踪跳过子 Agent                          │
└──────────────────────────────────────────────────────────┘
```

---

## 5. 流式工具执行 (StreamingToolExecutor)

```typescript
// src/query.ts:561
let streamingToolExecutor = config.gates.streamingToolExecution
  ? new StreamingToolExecutor(tools, canUseTool, toolUseContext)
  : null
```

```
┌──────────────────────────────────────────────────────────────┐
│  传统模式 vs 流式工具执行                                      │
│                                                              │
│  传统模式 (非流式):                                           │
│  ├── LLM 流式输出所有 tool_use 块                            │
│  ├── LLM 输出完成后，才开始执行工具                            │
│  └── 总延迟 = LLM 推理 + 工具执行                            │
│                                                              │
│  流式模式 (StreamingToolExecutor):                            │
│  ├── LLM 流式输出 tool_use 块                                │
│  ├── 每收到一个 tool_use 块，立即开始执行                     │
│  ├── 工具执行和 LLM 推理并行                                 │
│  └── 总延迟 = max(LLM 推理, 工具执行)  ← 更快！              │
│                                                              │
│  ┌────────────────────────────────────────────────────┐      │
│  │  LLM:  ───tool_use_1───tool_use_2───tool_use_3─── │      │
│  │  Tool: ──exec_1────exec_2────exec_3────────────── │      │
│  │        ↑并行↑                                       │      │
│  └────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Feature Flag 条件编译

query.ts 大量使用 `feature()` 进行条件编译：

```typescript
const reactiveCompact = feature('REACTIVE_COMPACT')
  ? require('./services/compact/reactiveCompact.js')
  : null

const contextCollapse = feature('CONTEXT_COLLAPSE')
  ? require('./services/compact/contextCollapse/index.js')
  : null

const snipModule = feature('HISTORY_SNIP')
  ? require('./services/compact/snipCompact.js')
  : null
```

**效果**: 未启用的功能在构建时被完全移除，减小包体积。

---

## 7. Terminal 返回值 — 循环结束原因

```typescript
type Terminal = {
  reason: 'completed' | 'aborted_streaming' | 'aborted_tools'
    | 'max_turns' | 'prompt_too_long' | 'image_error'
    | 'model_error' | 'stop_hook_prevented' | 'hook_stopped'
    | 'blocking_limit'
  turnCount?: number
  error?: unknown
}
```

```
┌──────────────────────────────────────────────────────────┐
│  循环结束原因                                              │
│                                                          │
│  completed          正常完成 (end_turn)                   │
│  aborted_streaming  用户在 LLM 推理时中断                 │
│  aborted_tools      用户在工具执行时中断                   │
│  max_turns          达到最大轮次限制                       │
│  prompt_too_long    上下文过长且压缩失败                   │
│  image_error        图片大小/格式错误                     │
│  model_error        LLM API 调用失败                      │
│  stop_hook_prevented Stop hook 阻止继续                   │
│  hook_stopped       Hook 阻止继续                         │
│  blocking_limit     达到硬性 token 上限                   │
└──────────────────────────────────────────────────────────┘
```

---

## 8. 关键设计模式总结

| 模式 | 说明 |
|------|------|
| **AsyncGenerator** | 整个循环是异步生成器，支持实时流式输出和中断 |
| **State 对象** | 跨迭代状态集中管理，continue 时一次性更新 |
| **多层压缩** | Snip → Microcompact → Collapse → Autocompact 四级压缩管道 |
| **模型降级** | Opus → Sonnet 自动降级，保证可用性 |
| **max_tokens 恢复** | 先升级到 64k，再注入恢复消息，最多 3 次 |
| **流式工具执行** | LLM 输出和工具执行并行，减少总延迟 |
| **Feature Flag** | 条件编译，未启用功能构建时移除 |
| **查询链追踪** | chainId + depth 追踪 Agent 调用链路 |

---

## 9. 面试要点

1. **为什么用 while(true) 而不是递归？** — 递归在深度 Agent 调用时可能栈溢出。while(true) + yield 是扁平化的递归，没有栈深度限制。

2. **上下文压缩为什么有四层？** — 每层解决不同问题：Snip 删除过期消息，Microcompact 删除缓存结果，Collapse 折叠相似消息，Autocompact 用 LLM 生成摘要。它们按成本从低到高排列，先尝试便宜的方案。

3. **max_tokens 恢复机制如何避免无限循环？** — MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3，最多恢复 3 次。第一次尝试升级到 64k tokens，后续注入恢复消息。超限后 surface 错误退出。

4. **流式工具执行如何处理依赖？** — StreamingToolExecutor 在 LLM 输出 tool_use 块时立即开始执行，但如果工具之间有依赖（写工具），实际执行仍然串行。只有只读工具真正并行。
