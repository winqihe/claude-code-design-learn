# 03 - 并发执行策略 (toolOrchestration.ts)

> 源文件: `src/services/tools/toolOrchestration.ts` (~188 行)
> 负责决定多个工具调用是并行还是串行执行，是 Agent 性能优化的关键。

---

## 1. 核心问题

当 LLM 一次返回多个 tool_use 块时，如何高效执行它们？

```
LLM 返回:
  tool_use_1: Grep("TODO")         ← 只读
  tool_use_2: Glob("*.ts")         ← 只读
  tool_use_3: Read("foo.ts")       ← 只读
  tool_use_4: Edit("bar.ts")       ← 写入
  tool_use_5: Bash("npm test")     ← 写入

问题: 这 5 个工具应该怎么执行？
```

---

## 2. 读写分离策略

### 2.1 partitionToolCalls — 分区算法

```typescript
// src/services/tools/toolOrchestration.ts:91
function partitionToolCalls(
  toolUseMessages: ToolUseBlock[],
  toolUseContext: ToolUseContext,
): Batch[] {
  return toolUseMessages.reduce((acc: Batch[], toolUse) => {
    const tool = findToolByName(toolUseContext.options.tools, toolUse.name)
    const parsedInput = tool?.inputSchema.safeParse(toolUse.input)
    const isConcurrencySafe = parsedInput?.success
      ? (() => {
          try {
            return Boolean(tool?.isConcurrencySafe(parsedInput.data))
          } catch {
            return false  // 解析失败 → 保守处理 → 不并行
          }
        })()
      : false

    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      // 追加到上一个只读批次
      acc[acc.length - 1]!.blocks.push(toolUse)
    } else {
      // 创建新批次
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

### 2.2 分区过程可视化

```
输入: [Grep, Glob, Read, Edit, Bash]

Step 1: Grep → 只读 → 新批次 [{safe, [Grep]}]
Step 2: Glob → 只读 → 追加 [{safe, [Grep, Glob]}]
Step 3: Read → 只读 → 追加 [{safe, [Grep, Glob, Read]}]
Step 4: Edit → 写入 → 新批次 [{safe, [Grep, Glob, Read]}, {unsafe, [Edit]}]
Step 5: Bash → 写入 → 新批次 [{safe, [Grep, Glob, Read]}, {unsafe, [Edit]}, {unsafe, [Bash]}]

结果: 3 个批次
  Batch 1: [Grep, Glob, Read]   → 并行执行
  Batch 2: [Edit]               → 串行执行
  Batch 3: [Bash]               → 串行执行
```

### 2.3 关键规则

```
┌──────────────────────────────────────────────────────────┐
│  分区规则                                                  │
│                                                          │
│  1. 连续的只读工具合并为一个批次                           │
│  2. 每个写入工具独占一个批次                               │
│  3. 写入工具打断只读批次的连续性                           │
│                                                          │
│  为什么写入工具不能合并？                                   │
│  ├── 写入工具可能修改文件系统                              │
│  ├── 后续工具可能依赖前一个工具的修改结果                   │
│  └── 例如: Edit 修改文件后，Bash 测试需要看到修改          │
│                                                          │
│  为什么只读工具可以合并？                                   │
│  ├── 只读工具不修改任何状态                                │
│  ├── 执行顺序不影响结果                                    │
│  └── 并行执行大幅减少总延迟                                │
│                                                          │
│  保守原则:                                                │
│  ├── Schema 解析失败 → 不并行                              │
│  ├── isConcurrencySafe 抛异常 → 不并行                     │
│  └── 宁可串行也不冒竞态条件的风险                          │
└──────────────────────────────────────────────────────────┘
```

---

## 3. 执行引擎

### 3.1 runTools — 顶层调度

```typescript
// src/services/tools/toolOrchestration.ts:19
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate, void> {
  let currentContext = toolUseContext

  for (const { isConcurrencySafe, blocks } of partitionToolCalls(
    toolUseMessages, currentContext,
  )) {
    if (isConcurrencySafe) {
      // 并行执行只读批次
      for await (const update of runToolsConcurrently(
        blocks, assistantMessages, canUseTool, currentContext,
      )) {
        // 收集 contextModifier
        yield { message: update.message, newContext: currentContext }
      }
      // 批次完成后，应用所有 contextModifier
      for (const block of blocks) {
        const modifiers = queuedContextModifiers[block.id]
        for (const modifier of modifiers) {
          currentContext = modifier(currentContext)
        }
      }
    } else {
      // 串行执行写入批次
      for await (const update of runToolsSerially(
        blocks, assistantMessages, canUseTool, currentContext,
      )) {
        if (update.newContext) {
          currentContext = update.newContext
        }
        yield { message: update.message, newContext: currentContext }
      }
    }
  }
}
```

### 3.2 runToolsConcurrently — 并行执行

```typescript
// src/services/tools/toolOrchestration.ts:152
async function* runToolsConcurrently(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void> {
  yield* all(
    toolUseMessages.map(async function* (toolUse) {
      toolUseContext.setInProgressToolUseIDs(prev =>
        new Set(prev).add(toolUse.id),
      )
      yield* runToolUse(toolUse, ..., canUseTool, toolUseContext)
      markToolUseAsComplete(toolUseContext, toolUse.id)
    }),
    getMaxToolUseConcurrency(),  // 最大并发数: 10
  )
}
```

### 3.3 runToolsSerially — 串行执行

```typescript
// src/services/tools/toolOrchestration.ts:118
async function* runToolsSerially(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate, void> {
  let currentContext = toolUseContext

  for (const toolUse of toolUseMessages) {
    toolUseContext.setInProgressToolUseIDs(prev =>
      new Set(prev).add(toolUse.id),
    )
    for await (const update of runToolUse(toolUse, ..., canUseTool, currentContext)) {
      if (update.contextModifier) {
        currentContext = update.contextModifier.modifyContext(currentContext)
      }
      yield { message: update.message, newContext: currentContext }
    }
    markToolUseAsComplete(toolUseContext, toolUse.id)
  }
}
```

---

## 4. 执行时序对比

```
┌──────────────────────────────────────────────────────────────────┐
│  场景: LLM 返回 5 个 tool_use (3 只读 + 2 写入)                   │
│                                                                  │
│  全串行执行 (无优化):                                             │
│  ├── Grep (~50ms) → Glob (~30ms) → Read (~20ms) → Edit (~100ms) │
│  └── → Bash (~2000ms)                                          │
│  总耗时: ~2200ms                                                 │
│                                                                  │
│  读写分离执行 (Claude Code):                                      │
│  ├── Phase 1 (并行): Grep + Glob + Read = max(50,30,20) = 50ms  │
│  ├── Phase 2 (串行): Edit = 100ms                               │
│  └── Phase 3 (串行): Bash = 2000ms                              │
│  总耗时: ~2150ms (节省 50ms)                                     │
│                                                                  │
│  如果有更多只读工具 (如 10 个):                                    │
│  ├── 全串行: 10 × 50ms = 500ms + 2100ms = 2600ms               │
│  └── 并行: max(10 × 50ms) + 2100ms = 2150ms (节省 450ms!)      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. ContextModifier 的处理差异

```
┌──────────────────────────────────────────────────────────┐
│  并行 vs 串行的 ContextModifier 处理                       │
│                                                          │
│  串行执行:                                                │
│  ├── 每个 tool 执行后立即应用 contextModifier             │
│  ├── 下一个 tool 看到修改后的 context                     │
│  └── 适合: 写入工具需要修改后续工具的环境                   │
│                                                          │
│  并行执行:                                                │
│  ├── 收集所有 contextModifier，不立即应用                  │
│  ├── 批次全部完成后，按顺序应用所有 modifier               │
│  └── 原因: 并行执行时无法确定 modifier 的应用顺序          │
│                                                          │
│  代码:                                                   │
│  // 并行: 先收集                                          │
│  const queuedContextModifiers = {}                        │
│  for await (update of runToolsConcurrently(...)) {        │
│    if (update.contextModifier) {                         │
│      queuedContextModifiers[toolUseID].push(modifier)    │
│    }                                                     │
│  }                                                       │
│  // 批次完成后: 按顺序应用                                 │
│  for (const block of blocks) {                           │
│    for (const modifier of queuedContextModifiers[block]) {│
│      currentContext = modifier(currentContext)            │
│    }                                                     │
│  }                                                       │
└──────────────────────────────────────────────────────────┘
```

---

## 6. 进度追踪

```typescript
// src/services/tools/toolOrchestration.ts:179
function markToolUseAsComplete(
  toolUseContext: ToolUseContext,
  toolUseID: string,
) {
  toolUseContext.setInProgressToolUseIDs(prev => {
    const next = new Set(prev)
    next.delete(toolUseID)
    return next
  })
}
```

每个工具执行前后都会更新 `inProgressToolUseIDs`，UI 可以实时显示哪些工具正在执行。

---

## 7. 最大并发控制

```typescript
// src/services/tools/toolOrchestration.ts:8
function getMaxToolUseConcurrency(): number {
  return (
    parseInt(process.env.CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY || '', 10) || 10
  )
}
```

默认最大并发数为 10，可通过环境变量 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 调整。

---

## 8. 面试要点

1. **为什么写入工具不能并行？** — 写入工具可能修改文件系统或外部状态，后续工具可能依赖前一个工具的修改。并行执行会导致竞态条件。

2. **ContextModifier 为什么在并行模式下延迟应用？** — 并行执行时，多个工具同时返回 contextModifier，无法确定它们的依赖顺序。延迟到批次完成后按原始顺序应用，保证一致性。

3. **如何判断一个工具是否并发安全？** — 通过 `tool.isConcurrencySafe(input)` 方法。每个工具根据自己的输入动态判断。例如 Bash 工具会解析命令，判断是否是只读命令（如 `git status`）。

4. **为什么 Schema 解析失败时不并行？** — 保守原则。如果无法解析输入，就无法判断工具是否安全，宁可串行也不冒风险。
