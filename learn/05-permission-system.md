# 05 - 权限系统 (useCanUseTool)

> 源文件: `src/hooks/useCanUseTool.tsx`
> Claude Code 的三层权限防线，控制每个工具调用的安全性。

---

## 1. 权限系统概述

Claude Code 的权限系统确保 LLM 不能未经用户同意执行危险操作。它是一个三层防线架构：

```
┌──────────────────────────────────────────────────────────────────┐
│                      权限决策流程                                  │
│                                                                  │
│  LLM 想调用: Bash("rm -rf /tmp/foo")                            │
│                    │                                             │
│                    ▼                                             │
│  ┌──────────────────────────────────────┐                        │
│  │  Layer 1: 规则匹配                    │                        │
│  │  hasPermissionsToUseTool()           │                        │
│  │                                      │                        │
│  │  alwaysAllowRules:                   │                        │
│  │    "Bash(git *)" → allow              │                        │
│  │    "Read(*.ts)"  → allow              │                        │
│  │                                      │                        │
│  │  alwaysDenyRules:                    │                        │
│  │    "Bash(rm -rf *)" → deny           │                        │
│  │                                      │                        │
│  │  alwaysAskRules:                     │                        │
│  │    "Bash(npm publish)" → ask         │                        │
│  └──────────────┬───────────────────────┘                        │
│                 │                                                 │
│         ┌───────┼───────┐                                        │
│         ▼       ▼       ▼                                        │
│      allow    deny     ask                                       │
│         │       │       │                                        │
│         │       │       ▼                                        │
│         │       │  ┌──────────────────────────────┐             │
│         │       │  │  Layer 2: 安全分类器           │             │
│         │       │  │  (TRANSCRIPT_CLASSIFIER)      │             │
│         │       │  │                              │             │
│         │       │  │  Bash 命令 → 安全/危险?       │             │
│         │       │  │  文件编辑 → 安全/危险?         │             │
│         │       │  └──────────────┬───────────────┘             │
│         │       │                 │                              │
│         │       │         ┌───────┼───────┐                      │
│         │       │         ▼       ▼       ▼                      │
│         │       │      safe   unsafe  unsure                   │
│         │       │         │       │       │                      │
│         │       │         ▼       ▼       ▼                      │
│         │       │  ┌──────────────────────────────┐             │
│         │       │  │  Layer 3: 用户确认 (ask)       │             │
│         │       │  │                              │             │
│         │       │  │  ┌────────────────────────┐  │             │
│         │       │  │  │ Allow │ Deny │ Always   │  │             │
│         │       │  │  │ Once  │      │ Allow    │  │             │
│         │       │  │  └────────────────────────┘  │             │
│         │       │  └──────────────────────────────┘             │
│         ▼       ▼                                               │
│      执行     拒绝                                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. CanUseToolFn — 权限检查函数签名

```typescript
// src/hooks/useCanUseTool.tsx:27
export type CanUseToolFn<Input extends Record<string, unknown> = Record<string, unknown>> = (
  tool: ToolType,
  input: Input,
  toolUseContext: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: PermissionDecision<Input>,
) => Promise<PermissionDecision<Input>>
```

**设计要点**:
- 返回 `Promise` — 权限检查可能是异步的（等待用户输入、调用分类器）
- `forceDecision` 参数 — 允许跳过权限检查（用于 plan mode 等场景）
- `PermissionDecision` 包含 `behavior` (allow/deny/ask) 和 `updatedInput`

---

## 3. useCanUseTool Hook 实现

```typescript
// src/hooks/useCanUseTool.tsx:28
function useCanUseTool(setToolUseConfirmQueue, setToolPermissionContext) {
  const t0 = async (tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision) =>
    new Promise(resolve => {
      // 1. 创建权限上下文
      const ctx = createPermissionContext(tool, input, toolUseContext, ...)

      // 2. 检查是否已中断
      if (ctx.resolveIfAborted(resolve)) return

      // 3. 执行权限检查 (或使用强制决策)
      const decisionPromise = forceDecision !== undefined
        ? Promise.resolve(forceDecision)
        : hasPermissionsToUseTool(tool, input, toolUseContext, ...)

      return decisionPromise.then(async result => {
        // 4. 处理 allow
        if (result.behavior === "allow") {
          resolve(ctx.buildAllow(result.updatedInput ?? input))
          return
        }

        // 5. 获取工具描述
        const description = await tool.description(input, ...)

        // 6. 处理 deny
        if (result.behavior === "deny") {
          resolve(result)
          return
        }

        // 7. 处理 ask → 进入用户确认流程
        switch (result.behavior) {
          case "ask":
            // 7a. Coordinator 权限处理
            const coordinatorDecision = await handleCoordinatorPermission(...)
            if (coordinatorDecision) { resolve(coordinatorDecision); return }

            // 7b. Swarm Worker 权限处理
            const swarmDecision = await handleSwarmWorkerPermission(...)
            if (swarmDecision) { resolve(swarmDecision); return }

            // 7c. 交互式用户确认
            const interactiveDecision = await handleInteractivePermission(...)
            resolve(interactiveDecision)
        }
      })
    })
}
```

---

## 4. 权限模式 (PermissionMode)

```
┌──────────────────────────────────────────────────────────────┐
│  权限模式                                                     │
│                                                              │
│  default (默认)                                              │
│  ├── 未匹配规则的工具需要用户确认                              │
│  ├── 匹配 alwaysAllow 的自动放行                              │
│  ├── 匹配 alwaysDeny 的自动拒绝                               │
│  └── 适合: 日常交互使用                                       │
│                                                              │
│  plan (计划模式)                                              │
│  ├── 只允许只读操作                                          │
│  ├── 写操作需要用户确认                                       │
│  └── 适合: 规划阶段，防止意外修改                             │
│                                                              │
│  bypassPermissions (绕过权限)                                 │
│  ├── 跳过所有权限检查                                        │
│  └── 适合: CI/CD、自动化脚本                                  │
│                                                              │
│  auto (自动模式)                                              │
│  ├── 安全分类器自动决策                                       │
│  ├── 安全操作自动放行，危险操作自动拒绝                        │
│  └── 适合: 自动化任务，无人值守                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. 权限规则匹配

```typescript
// 权限规则格式
type ToolPermissionRulesBySource = {
  config?: PermissionRule[]      // 来自配置文件
  hooks?: PermissionRule[]       // 来自 hooks
  classifier?: PermissionRule[] // 来自分类器
}

type PermissionRule = {
  tool: string           // 工具名 (支持通配符)
  pattern?: string       // 参数模式 (如 "git *")
  behavior: 'allow' | 'deny' | 'ask'
}
```

```
┌──────────────────────────────────────────────────────────┐
│  规则匹配示例                                            │
│                                                          │
│  规则: "Bash(git *)"                                    │
│  ├── 匹配: Bash("git status")     ✅                    │
│  ├── 匹配: Bash("git diff")       ✅                    │
│  ├── 匹配: Bash("git commit -m")  ✅                    │
│  └── 不匹配: Bash("npm test")     ❌                    │
│                                                          │
│  规则: "Read(*.ts)"                                      │
│  ├── 匹配: Read("src/foo.ts")    ✅                    │
│  └── 不匹配: Read("src/foo.js")  ❌                    │
│                                                          │
│  规则优先级:                                              │
│  1. alwaysDeny (最高优先级)                              │
│  2. alwaysAsk                                             │
│  3. alwaysAllow (最低优先级)                              │
└──────────────────────────────────────────────────────────┘
```

---

## 6. 三种权限处理器

### 6.1 Coordinator Handler

```typescript
// 用于 Coordinator 模式下的 Worker Agent
const coordinatorDecision = await handleCoordinatorPermission({
  ctx,
  pendingClassifierCheck: result.pendingClassifierCheck,
  updatedInput: result.updatedInput,
  suggestions: result.suggestions,
  permissionMode: appState.toolPermissionContext.mode,
})
```

### 6.2 Swarm Worker Handler

```typescript
// 用于 Swarm 模式下的 Worker Agent
const swarmDecision = await handleSwarmWorkerPermission({
  ctx,
  description,
  pendingClassifierCheck: result.pendingClassifierCheck,
  updatedInput: result.updatedInput,
  suggestions: result.suggestions,
})
```

### 6.3 Interactive Handler

```typescript
// 用于交互式 REPL 模式，弹出用户确认对话框
const interactiveDecision = await handleInteractivePermission({
  ctx,
  description,
  updatedInput: result.updatedInput,
  suggestions: result.suggestions,
})
```

---

## 7. 权限拒绝追踪

```typescript
// src/types/permissions.ts
type DenialTrackingState = {
  count: number
  lastDenialTime: number
}

// 在 ToolUseContext 中
localDenialTracking?: DenialTrackingState
```

```
┌──────────────────────────────────────────────────────────┐
│  拒绝追踪机制                                              │
│                                                          │
│  问题: 异步子 Agent 的 setAppState 是 no-op              │
│  ├── 拒绝计数器永远不会累加                               │
│  └── 回退到提示用户的阈值永远不会达到                       │
│                                                          │
│  解决: localDenialTracking                               │
│  ├── 每个 Agent 本地维护拒绝计数                          │
│  ├── 可变对象，权限代码原地更新                           │
│  └── 达到阈值后回退到提示用户                              │
└──────────────────────────────────────────────────────────┘
```

---

## 8. 安全分类器 (TRANSCRIPT_CLASSIFIER)

```typescript
// src/hooks/useCanUseTool.tsx:43
if (feature("TRANSCRIPT_CLASSIFIER") &&
    result.decisionReason?.type === "classifier" &&
    result.decisionReason.classifier === "auto-mode") {
  setYoloClassifierApproval(toolUseID, result.decisionReason.reason)
}
```

```
┌──────────────────────────────────────────────────────────┐
│  安全分类器工作流程                                        │
│                                                          │
│  1. 提取工具调用的紧凑表示                                 │
│     toAutoClassifierInput(input)                         │
│     ├── Bash: "ls -la"                                   │
│     ├── Edit: "/tmp/x: new content"                      │
│     └── 安全无关的工具: '' (跳过)                         │
│                                                          │
│  2. 分类器判断                                            │
│     ├── safe → 自动放行                                   │
│     ├── unsafe → 自动拒绝                                 │
│     └── unsure → 询问用户                                 │
│                                                          │
│  3. 记录分类结果                                          │
│     ├── setYoloClassifierApproval (放行时)                │
│     └── recordAutoModeDenial (拒绝时)                     │
└──────────────────────────────────────────────────────────┘
```

---

## 9. 面试要点

1. **为什么需要三层权限？** — 规则匹配处理明确的策略（如"git 命令总是允许"），分类器处理模糊的情况（如"这个 bash 命令是否安全"），用户确认是最终兜底。三层互补，缺一不可。

2. **为什么异步 Agent 需要本地拒绝追踪？** — 异步 Agent 的 `setAppState` 是 no-op（防止状态泄漏），但权限系统依赖拒绝计数来决定是否回退到提示用户。本地追踪解决了这个问题。

3. **bypassPermissions 模式为什么存在？** — CI/CD 和自动化脚本中无法交互式确认，需要完全跳过权限检查。这是一个必要的"逃生舱口"。

4. **preparePermissionMatcher 的作用？** — 为 hook 的 `if` 条件准备匹配器。例如规则 "Bash(git *)" 需要解析 bash 命令来判断是否匹配。这个方法在 hook-input 对级别调用一次，返回一个闭包用于每个 hook pattern 的匹配。
