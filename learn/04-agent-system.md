# 04 - 多 Agent 设计 (AgentTool)

> 源文件: `src/tools/AgentTool/prompt.ts`, `src/tools/AgentTool/UI.tsx`
> Claude Code 的多 Agent 系统，支持子 Agent 生成、Fork 模式和 Git Worktree 隔离。

---

## 1. AgentTool 概述

AgentTool 允许 Claude 生成子 Agent 来处理复杂的多步骤任务。这是 Claude Code 实现"Agent 编排 Agent"的核心机制。

```
┌──────────────────────────────────────────────────────────────────┐
│  Claude Code 的 Agent 层级                                       │
│                                                                  │
│  主 Agent (用户直接交互)                                          │
│  ├── AgentTool → 子 Agent A (代码审查)                           │
│  │              └── AgentTool → 孙 Agent (安全检查)              │
│  ├── AgentTool → 子 Agent B (测试运行)                           │
│  ├── AgentTool → Fork Agent C (代码搜索，共享上下文)              │
│  └── TeamCreate → 团队 Agent D, E, F (并行工作)                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 两种子 Agent 模式

### 2.1 Fresh Agent (指定 subagent_type)

```
┌──────────────────────────────────────────────────────────────┐
│  Fresh Agent                                                  │
│                                                              │
│  父 Agent                    子 Agent                         │
│  ┌──────────────┐            ┌──────────────┐               │
│  │ 完整对话上下文 │            │ 零上下文      │               │
│  │ 所有工具      │──spawn────►│ 需要完整 prompt│               │
│  │ prompt cache │            │ 独立 cache   │               │
│  └──────────────┘            └──────────────┘               │
│                                                              │
│  适用场景:                                                    │
│  • 需要独立视角的任务 (如: "给这段代码做独立 review")          │
│  • 不同工具集的 Agent (如: 只读审查 Agent)                    │
│  • 需要隔离执行环境的任务                                     │
│                                                              │
│  特点:                                                       │
│  • 子 Agent 看不到父 Agent 的对话历史                         │
│  • prompt 必须包含完整背景信息                                │
│  • 不共享 prompt cache (成本更高)                             │
│  • 可以指定不同的 model                                      │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Fork Agent (省略 subagent_type)

```
┌──────────────────────────────────────────────────────────────┐
│  Fork Agent                                                  │
│                                                              │
│  父 Agent                    Fork Agent                      │
│  ┌──────────────┐            ┌──────────────┐               │
│  │ 完整对话上下文 │            │ 继承父上下文  │               │
│  │ 所有工具      │──fork─────►│ 共享 cache   │               │
│  │ prompt cache │◄──共享─────│ 指令式 prompt │               │
│  └──────────────┘            └──────────────┘               │
│         │                         │                          │
│         └───── prompt cache 共享 ──┘                         │
│                                                              │
│  适用场景:                                                    │
│  • 中间输出不需要保留的研究任务                               │
│  • 可以分解为独立问题的调查                                   │
│  • 需要低成本执行的子任务                                     │
│                                                              │
│  特点:                                                       │
│  • 继承父 Agent 的完整对话上下文                              │
│  • prompt 是"指令"而非"背景说明"                             │
│  • 共享 prompt cache (成本极低)                              │
│  • 不要给 fork 设不同 model (会破坏 cache)                    │
│  • 结果通过 output_file 返回，不要窥探中间输出                │
└──────────────────────────────────────────────────────────────┘
```

### 2.3 Fork 的三条铁律

```
┌──────────────────────────────────────────────────────────┐
│  Fork 铁律 (来自 prompt.ts 的指令)                         │
│                                                          │
│  1. 不要窥探 (Don't peek)                                │
│  ├── 工具结果包含 output_file 路径                        │
│  ├── 不要 Read 或 tail 这个文件                            │
│  ├── 你会在完成时收到通知，信任它                           │
│  └── 窥探会把 fork 的工具噪声拉入你的上下文                │
│                                                          │
│  2. 不要预测 (Don't race)                                │
│  ├── fork 启动后你不知道它的结果                           │
│  ├── 永远不要编造或预测 fork 的结果                        │
│  ├── 通知以 user-role 消息到达，不是你写的                 │
│  └── 用户提前询问时说"还在运行"，给状态不给猜测             │
│                                                          │
│  3. 指令式 prompt (Directive prompt)                     │
│  ├── fork 继承上下文，prompt 是"做什么"                   │
│  ├── 不需要重述背景                                       │
│  ├── 要明确范围: 什么在、什么不在                          │
│  └── 简短命名 (1-2 个词) 以便用户在面板中识别              │
└──────────────────────────────────────────────────────────┘
```

---

## 3. 隔离机制

```
┌──────────────────────────────────────────────────────────────┐
│  isolation 参数                                              │
│                                                              │
│  无隔离 (默认)                                               │
│  ├── Agent 直接操作当前工作目录                               │
│  ├── 适合: 只读研究任务                                      │
│  └── 风险: 可能修改用户正在编辑的文件                         │
│                                                              │
│  "worktree"                                                  │
│  ├── 创建临时 git worktree                                   │
│  ├── Agent 在隔离的仓库副本中操作                             │
│  ├── 无变更 → 自动清理                                       │
│  ├── 有变更 → 返回 worktree 路径和分支名                     │
│  └── 适合: 并行修改代码 (如 /batch 技能)                     │
│                                                              │
│  "remote" (仅内部使用)                                       │
│  ├── 在远程 CCR (Claude Code Remote) 环境中运行              │
│  ├── 始终是后台任务                                          │
│  └── 适合: 长时间运行的任务                                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Agent 定义与工具限制

```typescript
// src/tools/AgentTool/prompt.ts:15
function getToolsDescription(agent: AgentDefinition): string {
  const { tools, disallowedTools } = agent
  const hasAllowlist = tools && tools.length > 0
  const hasDenylist = disallowedTools && disallowedTools.length > 0

  if (hasAllowlist && hasDenylist) {
    // 两者都有: 从 allowlist 中过滤 denylist
    const denySet = new Set(disallowedTools)
    return tools.filter(t => !denySet.has(t)).join(', ')
  } else if (hasAllowlist) {
    return tools.join(', ')  // 只有白名单
  } else if (hasDenylist) {
    return `All tools except ${disallowedTools.join(', ')}`  // 只有黑名单
  }
  return 'All tools'  // 无限制
}
```

```
┌──────────────────────────────────────────────────────────┐
│  工具限制策略                                              │
│                                                          │
│  tools + disallowedTools:                                │
│  ├── tools: ['Read', 'Grep', 'Glob']                    │
│  ├── disallowedTools: ['Glob']                          │
│  └── 有效工具: ['Read', 'Grep']  (交集)                  │
│                                                          │
│  仅 tools:                                               │
│  └── 只能使用列出的工具                                   │
│                                                          │
│  仅 disallowedTools:                                     │
│  └── 可以使用除列出之外的所有工具                         │
│                                                          │
│  都不设置:                                                │
│  └── 可以使用所有工具                                     │
└──────────────────────────────────────────────────────────┘
```

---

## 5. AgentTool 的 prompt 设计

AgentTool 的 prompt 是一个精心设计的指令文档，告诉 LLM 何时以及如何使用子 Agent：

```
┌──────────────────────────────────────────────────────────────┐
│  AgentTool Prompt 结构                                        │
│                                                              │
│  1. 核心说明                                                  │
│  ├── AgentTool 启动专门的子 Agent                            │
│  ├── 每个 Agent 类型有特定能力和工具                           │
│  └── 可用的 Agent 类型列表                                    │
│                                                              │
│  2. 何时 NOT 使用 (非 Coordinator 模式)                       │
│  ├── 读特定文件 → 用 Read/Glob                               │
│  ├── 搜索类定义 → 用 Grep/Glob                               │
│  ├── 搜索 2-3 个文件 → 用 Read                               │
│  └── 与 Agent 描述无关的任务                                  │
│                                                              │
│  3. 使用说明                                                  │
│  ├── 包含简短描述 (3-5 词)                                   │
│  ├── 前台 vs 后台选择                                        │
│  ├── 并行启动 (单消息多 tool_use)                            │
│  ├── 用 SendMessage 继续之前的 Agent                         │
│  └── 明确告诉 Agent 是写代码还是做研究                        │
│                                                              │
│  4. Fork 使用指南 (feature: FORK_SUBAGENT)                   │
│  ├── 何时 fork (研究、实现)                                   │
│  ├── Fork 铁律 (不窥探、不预测、指令式 prompt)                │
│  └── Fork 示例                                               │
│                                                              │
│  5. Prompt 写作指南                                           │
│  ├── 像对聪明的同事说话                                      │
│  ├── 解释目标和原因                                          │
│  ├── 描述已知的发现和排除的方案                               │
│  └── 不要委托理解 ("based on your findings, fix it")         │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. 后台执行与通知

```
┌──────────────────────────────────────────────────────────────┐
│  前台 vs 后台 Agent                                           │
│                                                              │
│  前台 (默认):                                                 │
│  ├── 主 Agent 等待子 Agent 完成                               │
│  ├── 子 Agent 结果直接返回给主 Agent                          │
│  └── 适合: 需要子 Agent 结果才能继续的任务                     │
│                                                              │
│  后台 (run_in_background: true):                              │
│  ├── 主 Agent 立即继续，不等待                                │
│  ├── 子 Agent 完成后以 user-role 消息通知                     │
│  ├── 主 Agent 收到通知后可以处理结果                          │
│  └── 适合: 独立任务，可以和其他工作并行                        │
│                                                              │
│  通知流程:                                                    │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐             │
│  │ 子 Agent  │     │ 完成通知  │     │ 主 Agent  │             │
│  │ 执行中... │────►│ user-role│────►│ 处理结果  │             │
│  └──────────┘     │ 消息     │     └──────────┘             │
│                   └──────────┘                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 7. Agent 列表注入优化

```typescript
// src/tools/AgentTool/prompt.ts:59
export function shouldInjectAgentListInMessages(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES)) return true
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_agent_list_attach', false)
}
```

```
┌──────────────────────────────────────────────────────────┐
│  Agent 列表注入优化                                       │
│                                                          │
│  问题: Agent 列表占工具描述的 ~10.2% cache tokens       │
│  ├── MCP 异步连接会改变列表                               │
│  ├── /reload-plugins 会改变列表                           │
│  └── 权限模式变更会改变列表                               │
│  → 每次变更都会 bust 整个工具 schema 的 prompt cache     │
│                                                          │
│  解决方案:                                               │
│  ├── tengu_agent_list_attach 功能开关                     │
│  ├── Agent 列表从工具描述移到对话消息 (attachment)         │
│  ├── 工具描述保持静态 → cache 不会被 bust                │
│  └── 动态列表通过 system-reminder 消息注入                │
└──────────────────────────────────────────────────────────┘
```

---

## 8. 面试要点

1. **Fork 和 Fresh Agent 的核心区别？** — Fork 继承父 Agent 的完整上下文和 prompt cache（低成本），Fresh Agent 从零开始（高隔离）。Fork 适合研究任务，Fresh 适合需要独立视角的任务。

2. **为什么 Fork 不能设不同 model？** — 不同 model 有不同的 prompt cache 格式，设不同 model 会破坏 cache 共享，失去 Fork 的成本优势。

3. **Git Worktree 隔离如何工作？** — 每个 Agent 在独立的 git worktree 中操作，拥有自己的工作目录和分支。无变更时自动清理，有变更时返回路径供用户审查。

4. **Agent 列表为什么从工具描述移到消息中？** — 因为 Agent 列表是动态的（MCP 连接、插件加载等会改变它），放在工具描述中会导致每次变更都 bust 整个 prompt cache，代价是 ~10% 的 cache tokens。
