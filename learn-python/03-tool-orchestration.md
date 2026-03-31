# 03 - 并发执行策略 (tool_orchestration.py)

> 对应源文件: `src/services/tools/toolOrchestration.ts` (~188 行)
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

### 2.1 partition_tool_calls — 分区算法

```python
from dataclasses import dataclass

@dataclass
class Batch:
    is_concurrency_safe: bool
    blocks: list[ToolUseBlock]

def partition_tool_calls(
    tool_use_messages: list[ToolUseBlock],
    tool_use_context: ToolUseContext,
) -> list[Batch]:
    batches: list[Batch] = []

    for tool_use in tool_use_messages:
        tool = find_tool_by_name(tool_use_context.tools, tool_use.name)

        is_concurrency_safe = False
        if tool and tool.input_schema:
            try:
                parsed_input = tool.input_schema.model_validate(tool_use.input)
                is_concurrency_safe = bool(tool.is_concurrency_safe(parsed_input))
            except Exception:
                is_concurrency_safe = False

        if is_concurrency_safe and batches and batches[-1].is_concurrency_safe:
            batches[-1].blocks.append(tool_use)
        else:
            batches.append(Batch(
                is_concurrency_safe=is_concurrency_safe,
                blocks=[tool_use],
            ))

    return batches
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
│  ├── is_concurrency_safe 抛异常 → 不并行                   │
│  └── 宁可串行也不冒竞态条件的风险                          │
└──────────────────────────────────────────────────────────┘
```

---

## 3. 执行引擎

### 3.1 run_tools — 顶层调度

```python
import asyncio
from collections.abc import AsyncGenerator

async def run_tools(
    tool_use_messages: list[ToolUseBlock],
    assistant_messages: list[Message],
    can_use_tool: CanUseToolFn,
    tool_use_context: ToolUseContext,
) -> AsyncGenerator[MessageUpdate, None]:
    current_context = tool_use_context

    for batch in partition_tool_calls(tool_use_messages, current_context):
        if batch.is_concurrency_safe:
            queued_modifiers: dict[str, list[Callable]] = {}
            async for update in run_tools_concurrently(
                batch.blocks, assistant_messages, can_use_tool, current_context,
            ):
                if update.context_modifier:
                    tool_use_id = update.tool_use_id
                    queued_modifiers.setdefault(tool_use_id, []).append(
                        update.context_modifier
                    )
                yield MessageUpdate(message=update.message, new_context=current_context)

            for block in batch.blocks:
                for modifier in queued_modifiers.get(block.id, []):
                    current_context = modifier(current_context)
        else:
            async for update in run_tools_serially(
                batch.blocks, assistant_messages, can_use_tool, current_context,
            ):
                if update.new_context:
                    current_context = update.new_context
                yield update
```

### 3.2 run_tools_concurrently — 并行执行

```python
MAX_TOOL_USE_CONCURRENCY = int(
    os.environ.get("CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY", "10")
)

async def run_tools_concurrently(
    tool_use_messages: list[ToolUseBlock],
    assistant_messages: list[Message],
    can_use_tool: CanUseToolFn,
    tool_use_context: ToolUseContext,
) -> AsyncGenerator[MessageUpdateLazy, None]:
    semaphore = asyncio.Semaphore(MAX_TOOL_USE_CONCURRENCY)

    async def execute_with_limit(tool_use: ToolUseBlock):
        async with semaphore:
            tool_use_context.set_in_progress_tool_use_ids(
                lambda prev: prev | {tool_use.id}
            )
            async for update in run_tool_use(
                tool_use, assistant_messages, can_use_tool, tool_use_context
            ):
                yield update
            mark_tool_use_as_complete(tool_use_context, tool_use.id)

    generators = [
        execute_with_limit(tu) for tu in tool_use_messages
    ]
    async for update in _merge_async_generators(generators):
        yield update
```

### 3.3 run_tools_serially — 串行执行

```python
async def run_tools_serially(
    tool_use_messages: list[ToolUseBlock],
    assistant_messages: list[Message],
    can_use_tool: CanUseToolFn,
    tool_use_context: ToolUseContext,
) -> AsyncGenerator[MessageUpdate, None]:
    current_context = tool_use_context

    for tool_use in tool_use_messages:
        tool_use_context.set_in_progress_tool_use_ids(
            lambda prev: prev | {tool_use.id}
        )
        async for update in run_tool_use(
            tool_use, assistant_messages, can_use_tool, current_context,
        ):
            if update.context_modifier:
                current_context = update.context_modifier(current_context)
            yield MessageUpdate(message=update.message, new_context=current_context)

        mark_tool_use_as_complete(tool_use_context, tool_use.id)
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
│  ├── 每个 tool 执行后立即应用 context_modifier             │
│  ├── 下一个 tool 看到修改后的 context                     │
│  └── 适合: 写入工具需要修改后续工具的环境                   │
│                                                          │
│  并行执行:                                                │
│  ├── 收集所有 context_modifier，不立即应用                  │
│  ├── 批次全部完成后，按顺序应用所有 modifier               │
│  └── 原因: 并行执行时无法确定 modifier 的应用顺序          │
│                                                          │
│  代码:                                                   │
│  # 并行: 先收集                                          │
│  queued_modifiers: dict[str, list[Callable]] = {}        │
│  async for update in run_tools_concurrently(...):        │
│      if update.context_modifier:                         │
│          queued_modifiers[tool_use_id].append(modifier)  │
│                                                          │
│  # 批次完成后: 按顺序应用                                 │
│  for block in batch.blocks:                              │
│      for modifier in queued_modifiers.get(block.id, []): │
│          current_context = modifier(current_context)      │
└──────────────────────────────────────────────────────────┘
```

---

## 6. 进度追踪

```python
def mark_tool_use_as_complete(
    tool_use_context: ToolUseContext,
    tool_use_id: str,
) -> None:
    tool_use_context.set_in_progress_tool_use_ids(
        lambda prev: prev - {tool_use_id}
    )
```

每个工具执行前后都会更新 `in_progress_tool_use_ids`，UI 可以实时显示哪些工具正在执行。

---

## 7. 最大并发控制

```python
import os

MAX_TOOL_USE_CONCURRENCY = int(
    os.environ.get("CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY", "10")
)
```

默认最大并发数为 10，可通过环境变量 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 调整。Python 中使用 `asyncio.Semaphore` 实现并发限制。

---

## 8. 合并异步生成器

```python
import asyncio
from collections import deque

async def _merge_async_generators(
    generators: list[AsyncGenerator],
) -> AsyncGenerator:
    queue: asyncio.Queue = asyncio.Queue()
    active_count = len(generators)

    async def drain(gen: AsyncGenerator):
        nonlocal active_count
        try:
            async for item in gen:
                await queue.put(item)
        finally:
            active_count -= 1
            if active_count == 0:
                await queue.put(None)

    tasks = [asyncio.create_task(drain(g)) for g in generators]
    while active_count > 0:
        item = await queue.get()
        if item is None:
            break
        yield item

    await asyncio.gather(*tasks, return_exceptions=True)
```

---

## 9. 面试要点

1. **为什么写入工具不能并行？** — 写入工具可能修改文件系统或外部状态，后续工具可能依赖前一个工具的修改。并行执行会导致竞态条件。

2. **ContextModifier 为什么在并行模式下延迟应用？** — 并行执行时，多个工具同时返回 context_modifier，无法确定它们的依赖顺序。延迟到批次完成后按原始顺序应用，保证一致性。

3. **如何判断一个工具是否并发安全？** — 通过 `tool.is_concurrency_safe(input)` 方法。每个工具根据自己的输入动态判断。例如 Bash 工具会解析命令，判断是否是只读命令（如 `git status`）。

4. **为什么 Schema 解析失败时不并行？** — 保守原则。如果无法解析输入，就无法判断工具是否安全，宁可串行也不冒风险。Python 中 `model_validate` 抛出 `ValidationError` 时走 `except Exception` 分支。

5. **asyncio.Semaphore vs asyncio.TaskGroup？** — Semaphore 控制并发上限（最多 10 个），TaskGroup 管理任务生命周期。这里用 Semaphore 是因为需要限制并发数而非等待所有任务完成。
