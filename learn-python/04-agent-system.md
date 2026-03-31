# 04 - 多 Agent 设计 (AgentTool)

> 对应源文件: `src/tools/AgentTool/prompt.ts`, `src/tools/AgentTool/UI.tsx`
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
│  Fork 铁律 (来自 prompt.py 的指令)                         │
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

```python
from enum import Enum

class IsolationMode(Enum):
    NONE = None
    WORKTREE = "worktree"
    REMOTE = "remote"
```

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

```python
from dataclasses import dataclass, field

@dataclass
class AgentDefinition:
    name: str
    description: str
    tools: list[str] = field(default_factory=list)
    disallowed_tools: list[str] = field(default_factory=list)
    model: str | None = None
    system_prompt: str | None = None

def get_tools_description(agent: AgentDefinition) -> str:
    has_allowlist = len(agent.tools) > 0
    has_denylist = len(agent.disallowed_tools) > 0

    if has_allowlist and has_denylist:
        deny_set = set(agent.disallowed_tools)
        return ", ".join(t for t in agent.tools if t not in deny_set)
    elif has_allowlist:
        return ", ".join(agent.tools)
    elif has_denylist:
        return f"All tools except {', '.join(agent.disallowed_tools)}"
    return "All tools"
```

```
┌──────────────────────────────────────────────────────────┐
│  工具限制策略                                              │
│                                                          │
│  tools + disallowed_tools:                                │
│  ├── tools: ['Read', 'Grep', 'Glob']                    │
│  ├── disallowed_tools: ['Glob']                          │
│  └── 有效工具: ['Read', 'Grep']  (交集)                  │
│                                                          │
│  仅 tools:                                               │
│  └── 只能使用列出的工具                                   │
│                                                          │
│  仅 disallowed_tools:                                     │
│  └── 可以使用除列出之外的所有工具                         │
│                                                          │
│  都不设置:                                                │
│  └── 可以使用所有工具                                     │
└──────────────────────────────────────────────────────────┘
```

---

## 5. AgentTool 的 prompt 设计

```python
AGENT_TOOL_PROMPT = """
## AgentTool

Launch a sub-agent and assign a task to it.

### When NOT to use (non-Coordinator mode)
- To read a specific file → use Read/Glob
- To search for a class definition → use Grep/Glob
- To search 2-3 files → use Read
- Tasks unrelated to any agent description

### How to use
1. Include a short description (3-5 words)
2. Choose foreground vs background
3. Launch multiple agents in a single message for parallelism
4. Use SendMessage to continue a previous agent
5. Tell the agent whether to write code or do research

### Fork usage (when FORK_SUBAGENT enabled)
- Fork for research or implementation tasks
- Fork rules: don't peek, don't race, directive prompt
- Fork shares prompt cache with parent (low cost)

### Prompt writing guide
- Talk to the agent like a smart colleague
- Explain the goal and the reason
- Describe known findings and excluded approaches
- Don't delegate understanding ("based on your findings, fix it")
"""
```

---

## 6. 后台执行与通知

```python
from dataclasses import dataclass

@dataclass
class AgentToolInput(BaseModel):
    name: str = Field(description="Agent name (1-2 words)")
    description: str = Field(description="Task description (3-5 words)")
    query: str = Field(description="Detailed task instructions")
    subagent_type: str | None = Field(default=None, description="Agent type")
    isolation: str | None = Field(default=None, description="Isolation mode")
    run_in_background: bool = Field(default=False, description="Run in background")
```

```
┌──────────────────────────────────────────────────────────────┐
│  前台 vs 后台 Agent                                           │
│                                                              │
│  前台 (默认):                                                 │
│  ├── 主 Agent 等待子 Agent 完成                               │
│  ├── 子 Agent 结果直接返回给主 Agent                          │
│  └── 适合: 需要子 Agent 结果才能继续的任务                     │
│                                                              │
│  后台 (run_in_background=True):                               │
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

```python
def should_inject_agent_list_in_messages() -> bool:
    if os.environ.get("CLAUDE_CODE_AGENT_LIST_IN_MESSAGES", "").lower() == "true":
        return True
    return get_feature_value_cached("tengu_agent_list_attach", False)
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

5. **Python 中如何实现 Fork 的 prompt cache 共享？** — 将父 Agent 的 messages 列表直接传递给子 Agent，子 Agent 在此基础上追加自己的消息。由于共享相同的 messages 前缀，LLM API 的 prompt cache 可以命中。
