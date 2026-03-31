# 02 - Agent 运行循环 (query.py)

> 对应源文件: `src/query.ts` (~1729 行)
> 这是 Claude Code 的心脏 — 一个 `while True` 无限循环，驱动着 LLM 推理和工具执行的交替进行。

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
│  │                    while True:                              │  │
│  │                                                            │  │
│  │   ① 上下文管理                                              │  │
│  │   ├── snip (历史裁剪)                                      │  │
│  │   ├── microcompact (微压缩)                                │  │
│  │   ├── context_collapse (上下文折叠)                        │  │
│  │   └── autocompact (自动压缩)                               │  │
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
│  │   ④ 执行工具 (run_tools)                                  │  │
│  │   ├── 只读工具 → 并行                                      │  │
│  │   └── 写工具 → 串行                                        │  │
│  │                                                            │  │
│  │   ⑤ 收集附件 (attachments)                                 │  │
│  │   ├── 文件变更通知                                         │  │
│  │   ├── 记忆文件                                             │  │
│  │   └── 技能发现                                             │  │
│  │                                                            │  │
│  │   ⑥ 安全检查                                              │  │
│  │   ├── max_turns 检查                                       │  │
│  │   └── abort 检查                                          │  │
│  │                                                            │  │
│  │   ⑦ 更新状态 → 回到 ①                                    │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 入口函数与状态管理

### 2.1 query() — 公开入口

```python
from collections.abc import AsyncGenerator
from dataclasses import dataclass, field
from typing import Any

@dataclass
class Terminal:
    reason: str
    turn_count: int | None = None
    error: Any = None

async def query(params: QueryParams) -> AsyncGenerator[StreamEvent | Message | TombstoneMessage | ToolUseSummaryMessage, Terminal]:
    consumed_command_uuids: list[str] = []

    async for item in query_loop(params, consumed_command_uuids):
        if isinstance(item, Terminal):
            for uuid in consumed_command_uuids:
                notify_command_lifecycle(uuid, "completed")
            return item
        yield item

    return Terminal(reason="completed")
```

**设计要点**:
- 使用 `AsyncGenerator` 实现流式输出，调用者可以实时接收消息
- `async for` 委托给内部 `query_loop`，保持生成器链
- 返回 `Terminal` 类型，描述循环结束的原因

### 2.2 State — 跨迭代状态

```python
from dataclasses import dataclass, field

@dataclass
class QueryTracking:
    chain_id: str
    depth: int

@dataclass
class AutoCompactTrackingState:
    compacted_token_count: int = 0
    last_compact_turn: int = 0

@dataclass
class State:
    messages: list[Message] = field(default_factory=list)
    tool_use_context: ToolUseContext = field(default_factory=ToolUseContext)
    auto_compact_tracking: AutoCompactTrackingState = field(default_factory=AutoCompactTrackingState)
    max_output_tokens_recovery_count: int = 0
    has_attempted_reactive_compact: bool = False
    max_output_tokens_override: int | None = None
    pending_tool_use_summary: Any = None
    stop_hook_active: bool | None = None
    turn_count: int = 0
    transition: Continue | None = None
```

**设计要点**: 使用单个 `state` dataclass，在 continue 点一次性更新所有字段，避免 9 个独立赋值：

```python
state = State(
    messages=[*messages_for_query, *assistant_messages, *tool_results],
    tool_use_context=tool_use_context_with_query_tracking,
    turn_count=next_turn_count,
    max_output_tokens_recovery_count=0,
    has_attempted_reactive_compact=False,
    transition=Continue(reason="next_turn"),
)
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
│  Snip: 裁剪过期的历史消息                                     │
│  ├── 删除不再需要的中间消息                                    │
│  └── 保留关键上下文                                           │
│                                                              │
│  Microcompact: 微型压缩 (cached microcompact)                  │
│  ├── 删除已缓存的工具结果                                      │
│  └── 减少 token 消耗但不丢失信息                               │
│                                                              │
│  Context Collapse: 上下文折叠                                  │
│  ├── 将相似消息折叠为摘要                                      │
│  └── 保留细粒度上下文而非单一摘要                               │
│                                                              │
│  Autocompact: 自动压缩                                        │
│  ├── 当 token 接近上限时触发                                   │
│  ├── 用 LLM 生成对话摘要                                      │
│  └── 替换旧消息为摘要消息                                      │
│                                                              │
│  Token Budget: Token 预算                                     │
│  ├── 跟踪 token 消耗                                          │
│  └── 超预算时注入提示消息要求精简                               │
└──────────────────────────────────────────────────────────────┘
```

### 阶段 ②: 调用 LLM API

```python
async def call_model(
    messages: list[Message],
    system_prompt: str,
    tools: list[Tool],
    model: str,
    fallback_model: str | None = None,
    signal: AbortSignal | None = None,
) -> AsyncGenerator[Message, None]:
    try:
        async for message in _stream_api_call(
            messages=messages,
            system_prompt=system_prompt,
            tools=tools,
            model=model,
            signal=signal,
        ):
            yield message
    except OverloadedError:
        if fallback_model and not attempt_with_fallback:
            yield await _handle_model_fallback(
                messages, system_prompt, tools, fallback_model
            )
        else:
            raise
```

**关键特性 — 模型降级 (Fallback)**:

```
┌──────────────────────────────────────────────────────────┐
│  模型降级流程                                              │
│                                                          │
│  调用 Claude Opus ──► 高负载错误 ──► 切换到 Sonnet       │
│                                                          │
│  降级时:                                                 │
│  1. 清空已收集的 assistant_messages                       │
│  2. 为已发出的 tool_use 生成空的 tool_result              │
│  3. 清除 streaming_tool_executor 的待处理结果             │
│  4. 剥离 thinking 签名块 (不同模型签名不兼容)             │
│  5. 用新模型重试整个请求                                  │
│                                                          │
│  降级最多重试 1 次 (attempt_with_fallback 标志)           │
└──────────────────────────────────────────────────────────┘
```

### 阶段 ③: 判断 stop_reason

```python
class StopReason(Enum):
    END_TURN = "end_turn"
    TOOL_USE = "tool_use"
    MAX_TOKENS = "max_tokens"
    PROMPT_TOO_LONG = "prompt_too_long"
    ABORTED = "aborted"

async def handle_stop_reason(
    stop_reason: StopReason,
    state: State,
    assistant_messages: list[Message],
) -> Terminal | None:
    match stop_reason:
        case StopReason.END_TURN:
            await execute_stop_hooks(state)
            return Terminal(reason="completed")

        case StopReason.TOOL_USE:
            return None  # 继续循环，执行工具

        case StopReason.MAX_TOKENS:
            if state.max_output_tokens_recovery_count < MAX_RECOVERY_LIMIT:
                return await _recover_max_tokens(state)
            return Terminal(reason="blocking_limit")

        case StopReason.PROMPT_TOO_LONG:
            if await _try_context_collapse_drain(state):
                return None
            if await _try_reactive_compact(state):
                return None
            return Terminal(reason="prompt_too_long")

        case StopReason.ABORTED:
            await _cleanup_mcp_resources(state)
            return Terminal(reason="aborted_tools")
```

### 阶段 ④: 执行工具

```python
async def run_phase_tool_execution(
    tool_use_blocks: list[ToolUseBlock],
    assistant_messages: list[Message],
    can_use_tool: CanUseToolFn,
    tool_use_context: ToolUseContext,
) -> AsyncGenerator[MessageUpdate, None]:
    if streaming_tool_executor:
        async for update in streaming_tool_executor.get_remaining_results():
            yield update
    else:
        async for update in run_tools(
            tool_use_blocks, assistant_messages, can_use_tool, tool_use_context
        ):
            yield update
```

### 阶段 ⑤: 收集附件

```python
async def get_attachment_messages(
    tool_use_context: ToolUseContext,
    queued_commands: list[Command],
    messages: list[Message],
    query_source: str,
) -> AsyncGenerator[AttachmentMessage, None]:
    async for attachment in _collect_attachments(
        tool_use_context, queued_commands, messages, query_source
    ):
        yield attachment
```

### 阶段 ⑥: 安全检查

```python
MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

if max_turns and next_turn_count > max_turns:
    yield AttachmentMessage(
        type="max_turns_reached",
        max_turns=max_turns,
        turn_count=next_turn_count,
    )
    return Terminal(reason="max_turns", turn_count=next_turn_count)
```

---

## 4. QueryChain — 查询链追踪

```python
def build_query_tracking(
    tool_use_context: ToolUseContext,
    uuid_fn: Callable[[], str],
) -> QueryTracking:
    if tool_use_context.query_tracking:
        return QueryTracking(
            chain_id=tool_use_context.query_tracking.chain_id,
            depth=tool_use_context.query_tracking.depth + 1,
        )
    return QueryTracking(chain_id=uuid_fn(), depth=0)
```

```
┌──────────────────────────────────────────────────────────┐
│  Query Chain 追踪                                          │
│                                                          │
│  主 Agent (depth=0, chain_id=abc)                         │
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

```python
class StreamingToolExecutor:
    def __init__(self, tools: list[Tool], can_use_tool: CanUseToolFn, ctx: ToolUseContext):
        self._tools = tools
        self._can_use_tool = can_use_tool
        self._ctx = ctx
        self._pending: asyncio.Queue[MessageUpdate] = asyncio.Queue()

    async def submit_tool_use(self, tool_use: ToolUseBlock) -> None:
        asyncio.create_task(self._execute_tool(tool_use))

    async def get_remaining_results(self) -> AsyncGenerator[MessageUpdate, None]:
        while not self._pending.empty():
            yield await self._pending.get()
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

## 6. Terminal 返回值 — 循环结束原因

```python
class TerminalReason(Enum):
    COMPLETED = "completed"
    ABORTED_STREAMING = "aborted_streaming"
    ABORTED_TOOLS = "aborted_tools"
    MAX_TURNS = "max_turns"
    PROMPT_TOO_LONG = "prompt_too_long"
    IMAGE_ERROR = "image_error"
    MODEL_ERROR = "model_error"
    STOP_HOOK_PREVENTED = "stop_hook_prevented"
    HOOK_STOPPED = "hook_stopped"
    BLOCKING_LIMIT = "blocking_limit"
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

## 7. 关键设计模式总结

| 模式 | Python 实现方式 | 说明 |
|------|----------------|------|
| **AsyncGenerator** | `async def ... -> AsyncGenerator` | 整个循环是异步生成器，支持实时流式输出和中断 |
| **State dataclass** | `@dataclass class State` | 跨迭代状态集中管理，continue 时一次性更新 |
| **多层压缩** | 四级管道: Snip → Micro → Collapse → Auto | 按成本从低到高排列 |
| **模型降级** | try/except + fallback_model | Opus → Sonnet 自动降级，保证可用性 |
| **max_tokens 恢复** | 计数器 + 升级到 64k | 先升级，再注入恢复消息，最多 3 次 |
| **流式工具执行** | `asyncio.Queue` + `create_task` | LLM 输出和工具执行并行 |
| **查询链追踪** | `QueryTracking(chain_id, depth)` | chainId + depth 追踪 Agent 调用链路 |
| **match/case** | Python 3.10+ 结构化模式匹配 | 优雅处理 stop_reason 分支 |

---

## 8. 面试要点

1. **为什么用 while True 而不是递归？** — 递归在深度 Agent 调用时可能栈溢出。`while True` + `yield` 是扁平化的递归，没有栈深度限制。Python 的 `sys.setrecursionlimit()` 虽然可以调，但 AsyncGenerator 天然避免了这个问题。

2. **上下文压缩为什么有四层？** — 每层解决不同问题：Snip 删除过期消息，Microcompact 删除缓存结果，Collapse 折叠相似消息，Autocompact 用 LLM 生成摘要。它们按成本从低到高排列，先尝试便宜的方案。

3. **max_tokens 恢复机制如何避免无限循环？** — `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`，最多恢复 3 次。第一次尝试升级到 64k tokens，后续注入恢复消息。超限后 surface 错误退出。

4. **流式工具执行如何处理依赖？** — `StreamingToolExecutor` 在 LLM 输出 tool_use 块时立即用 `asyncio.create_task` 开始执行，但如果工具之间有依赖（写工具），实际执行仍然串行。只有只读工具真正并行。
