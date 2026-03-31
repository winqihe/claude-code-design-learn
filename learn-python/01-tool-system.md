# 01 - 工具抽象核心 (tool.py)

> 对应源文件: `src/Tool.ts` (~792 行)
> 这是 Claude Code 整个工具系统的基石，定义了 Tool 接口、ToolUseContext、build_tool 工厂函数等核心类型。

---

## 1. 为什么 tool.py 是最重要的文件？

Claude Code 有 40+ 个内置工具（Bash、Read、Write、Edit、Grep、Glob、WebFetch、AgentTool...），每个工具都要处理：

- 输入验证（Pydantic Schema）
- 权限检查（allow/deny/ask）
- 并发安全性（能否并行执行）
- 进度报告（流式 UI 更新）
- 结果渲染（模板渲染组件）
- 错误处理（自定义错误 UI）

`tool.py` 用一个统一的泛型 Protocol 解决了所有这些问题。

---

## 2. Tool 接口全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Tool[InputT, OutputT, ProgressT]            │
│                                                                 │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │      核心执行方法       │  │        Schema 定义             │  │
│  │                        │  │                              │  │
│  │  call(args, ctx, ...)  │  │  input_schema: type[BaseModel]│  │
│  │    -> ToolResult       │  │  output_schema?: type[...]   │  │
│  │                        │  │                              │  │
│  │  description(input)    │  │  inputs_equivalent?(a, b)    │  │
│  │    -> str              │  │    -> bool                   │  │
│  │                        │  │                              │  │
│  │  prompt(options)       │  │  validate_input?(input, ctx) │  │
│  │    -> str              │  │    -> ValidationResult       │  │
│  │                        │  │                              │  │
│  │  map_result(...)       │  │                              │  │
│  │    -> ToolResultBlock  │  │                              │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
│                                                                 │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │    安全 & 并发控制       │  │        UI 渲染                 │  │
│  │                        │  │                              │  │
│  │  check_permissions()   │  │  render_tool_use_message()  │  │
│  │    -> PermissionResult │  │  render_tool_result()       │  │
│  │                        │  │  render_tool_use_progress() │  │
│  │  is_concurrency_safe() │  │  render_tool_use_rejected() │  │
│  │    -> bool             │  │  render_tool_use_error()    │  │
│  │                        │  │  render_grouped_tool_use()  │  │
│  │  is_read_only()        │  │  render_tool_use_tag()      │  │
│  │    -> bool             │  │  render_tool_use_queued()   │  │
│  │                        │  │                              │  │
│  │  is_destructive?()     │  │  user_facing_name()         │  │
│  │    -> bool             │  │  get_activity_description() │  │
│  │                        │  │  get_tool_use_summary()     │  │
│  │  interrupt_behavior?() │  │  extract_search_text()      │  │
│  │    -> 'cancel'|'block' │  │                              │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
│                                                                 │
│  ┌────────────────────────┐  ┌──────────────────────────────┐  │
│  │       元数据            │  │       行为标记                 │  │
│  │                        │  │                              │  │
│  │  name: str             │  │  is_mcp: bool               │  │
│  │  aliases: list[str]    │  │  is_lsp: bool               │  │
│  │  search_hint: str      │  │  should_defer: bool         │  │
│  │  max_result_size_chars │  │  always_load: bool          │  │
│  │  strict: bool          │  │  is_open_world?()           │  │
│  │                        │  │  requires_user_interaction? │  │
│  │  mcp_info: dict        │  │  is_transparent_wrapper?() │  │
│  └────────────────────────┘  └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心类型详解

### 3.1 ToolResult — 工具执行结果

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Any, Callable, TypeVar

T = TypeVar("T")

@dataclass
class ToolResult:
    data: T
    new_messages: list[Message] = field(default_factory=list)
    context_modifier: Callable[["ToolUseContext"], "ToolUseContext"] | None = None
    mcp_meta: dict[str, Any] | None = None
```

**关键设计**: `context_modifier` 允许工具执行后修改后续工具的执行上下文。例如，MCP 工具连接后可以将新的工具注册到上下文中。

### 3.2 ToolUseContext — 工具执行上下文

这是传递给每个工具的"环境"，包含了工具执行所需的一切：

```python
from dataclasses import dataclass, field
from typing import Any, Callable

@dataclass
class ToolUseContext:
    commands: list[Command]
    tools: list[Tool]
    main_loop_model: str
    mcp_clients: list[MCPServer]
    agent_definitions: list[AgentDefinition]
    max_budget_usd: float | None = None
    custom_system_prompt: str | None = None
    refresh_tools: Callable[[], list[Tool]] | None = None

    abort_controller: AbortController = field(default_factory=AbortController)
    read_file_state: FileStateCache = field(default_factory=FileStateCache)
    messages: list[Message] = field(default_factory=list)

    agent_id: str | None = None
    agent_type: str | None = None
    query_tracking: QueryTracking | None = None

    in_progress_tool_use_ids: set[str] = field(default_factory=set)

    def set_in_progress_tool_use_ids(
        self, updater: Callable[[set[str]], set[str]]
    ) -> None:
        self.in_progress_tool_use_ids = updater(self.in_progress_tool_use_ids)
```

### 3.3 ToolPermissionContext — 权限上下文

```python
from enum import Enum

class PermissionMode(Enum):
    DEFAULT = "default"
    PLAN = "plan"
    BYPASS_PERMISSIONS = "bypassPermissions"
    AUTO = "auto"

@dataclass
class ToolPermissionContext:
    mode: PermissionMode
    additional_working_dirs: dict[str, AdditionalWorkingDirectory]
    always_allow_rules: dict[str, list[PermissionRule]]
    always_deny_rules: dict[str, list[PermissionRule]]
    always_ask_rules: dict[str, list[PermissionRule]]
    is_bypass_permissions_mode_available: bool = False
    is_auto_mode_available: bool = False
    should_avoid_permission_prompts: bool = False
    await_automated_checks_before_dialog: bool = False
    pre_plan_mode: PermissionMode | None = None
```

---

## 4. build_tool 工厂函数 — 安全默认值模式

这是整个工具系统最优雅的设计之一：

```python
from functools import wraps
from typing import Protocol, runtime_checkable

TOOL_DEFAULTS: dict[str, Any] = {
    "is_enabled": lambda: True,
    "is_concurrency_safe": lambda input=None: False,
    "is_read_only": lambda input=None: False,
    "is_destructive": lambda input=None: False,
    "check_permissions": lambda input, ctx=None: PermissionResult(
        behavior="allow", updated_input=input
    ),
    "to_auto_classifier_input": lambda input=None: "",
    "user_facing_name": lambda input=None: "",
}

def build_tool(
    name: str,
    *,
    input_schema: type[BaseModel] | None = None,
    call: Callable | None = None,
    description: Callable | None = None,
    prompt: Callable | None = None,
    map_result: Callable | None = None,
    render_tool_use_message: Callable | None = None,
    render_tool_result_message: Callable | None = None,
    is_concurrency_safe: Callable | None = None,
    is_read_only: Callable | None = None,
    is_destructive: Callable | None = None,
    is_enabled: Callable[[], bool] | None = None,
    check_permissions: Callable | None = None,
    max_result_size_chars: int | None = None,
    aliases: list[str] | None = None,
    search_hint: str | None = None,
    is_mcp: bool = False,
    **kwargs,
) -> Tool:
    overrides = {
        k: v for k, v in locals().items()
        if v is not None and k not in ("name", "kwargs", "input_schema")
    }
    return Tool(
        name=name,
        input_schema=input_schema,
        user_facing_name=lambda input=None: name,
        **TOOL_DEFAULTS,
        **overrides,
        **kwargs,
    )
```

### 设计哲学：Fail-Closed（安全优先）

```
┌──────────────────────────────────────────────────────────┐
│              build_tool 默认值策略                           │
│                                                          │
│  属性                默认值         设计理由              │
│  ─────────────────────────────────────────────────────── │
│  is_concurrency_safe  False        不声明 = 不安全          │
│  is_read_only         False        不声明 = 可写            │
│  is_destructive       False        不声明 = 非破坏性        │
│  is_enabled           True         不声明 = 启用            │
│  check_permissions    allow        交给通用权限系统处理      │
│  to_auto_classifier    ''           不声明 = 跳过安全分类    │
│                                                          │
│  核心原则: 宁可保守，不可激进                               │
│  • 新工具默认不能并行（避免竞态条件）                       │
│  • 新工具默认可写（避免误判为只读导致安全问题）              │
│  • 安全相关的工具必须显式声明安全属性                       │
└──────────────────────────────────────────────────────────┘
```

---

## 5. 工具查找机制

```python
def tool_matches_name(tool: Tool, name: str) -> bool:
    return tool.name == name or (tool.aliases and name in tool.aliases)

def find_tool_by_name(tools: list[Tool], name: str) -> Tool | None:
    for tool in tools:
        if tool_matches_name(tool, name):
            return tool
    return None
```

工具支持别名（aliases），用于向后兼容。例如工具改名后，旧名字仍然可以通过别名找到。

---

## 6. ToolDef vs Tool — 简化工具定义

Python 中我们用 `build_tool()` 关键字参数来简化工具定义，不需要像 TypeScript 那样定义复杂的类型差集：

```
┌──────────────────────────────────────────────────────────┐
│  定义工具的两种方式                                       │
│                                                          │
│  方式 1: 直接实现 Tool Protocol (需要 30+ 个方法)         │
│  ├── 繁琐，容易遗漏                                       │
│  └── 不推荐                                              │
│                                                          │
│  方式 2: 使用 build_tool() (只需核心方法)                  │
│  ├── 只需实现: call, description, prompt,                 │
│  │           input_schema, render_tool_use_message,       │
│  │           map_result                                  │
│  ├── 7 个安全方法自动填充默认值                            │
│  └── ✅ 推荐，所有 60+ 工具都使用这种方式                  │
└──────────────────────────────────────────────────────────┘
```

---

## 7. 进度报告系统

工具可以实时报告执行进度：

```python
from typing import Protocol

class ToolCallProgress(Protocol):
    def __call__(self, progress: ToolProgress) -> None: ...

@dataclass
class BashProgress:
    tool_use_id: str
    data: dict[str, Any]

@dataclass
class MCPProgress:
    tool_use_id: str
    data: dict[str, Any]

@dataclass
class AgentToolProgress:
    tool_use_id: str
    data: dict[str, Any]

ToolProgress = BashProgress | MCPProgress | AgentToolProgress | ...
```

进度通过 `on_progress` 回调传递，最终渲染为终端 UI 中的 spinner 和状态文本。

---

## 8. 延迟加载工具 (Deferred Tools)

```python
@dataclass
class Tool:
    name: str
    should_defer: bool = False
    always_load: bool = False
    # ...
```

```
┌──────────────────────────────────────────────────────────┐
│  工具加载策略                                              │
│                                                          │
│  should_defer: True                                       │
│  ├── 工具 schema 不在初始 prompt 中                       │
│  ├── 模型需要先调用 ToolSearch 才能发现                    │
│  └── 用途: 减少初始 prompt 的 token 消耗                   │
│                                                          │
│  always_load: True                                        │
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

```python
from pydantic import BaseModel, Field

class MyToolInput(BaseModel):
    query: str = Field(description="搜索查询")
    max_results: int = Field(default=10, description="最大返回数")

class SearchResult(BaseModel):
    title: str
    url: str
    snippet: str

my_tool = build_tool(
    name="MyTool",
    search_hint="custom operation for X",

    input_schema=MyToolInput,

    async def call(args: MyToolInput, ctx: ToolUseContext, can_use_tool, parent_msg, on_progress):
        if on_progress:
            on_progress(BashProgress(tool_use_id="xxx", data={"type": "progress", "percent": 50}))
        results = await do_search(args.query, args.max_results)
        return ToolResult(data=results)

    async def description(input: MyToolInput, options) -> str:
        return f'搜索 "{input.query}"，最多返回 {input.max_results} 个结果'

    async def prompt(options) -> str:
        return "使用 MyTool 来搜索自定义数据源..."

    def map_result(content, tool_use_id) -> dict:
        return {
            "type": "tool_result",
            "tool_use_id": tool_use_id,
            "content": json.dumps(content),
        }

    def render_tool_use_message(input: MyToolInput, options) -> str:
        return f"搜索: {input.query}"

    def render_tool_result_message(content, progress_messages, options) -> str:
        return f"找到 {len(content)} 个结果"

    is_concurrency_safe=lambda input=None: True,
    is_read_only=lambda input=None: True,
    max_result_size_chars=100_000,
)
```

---

## 10. 关键设计模式总结

| 模式 | Python 实现方式 | 说明 |
|------|----------------|------|
| **泛型三层抽象** | `Tool[InputT, OutputT, ProgressT]` | 输入、输出、进度类型完全参数化 |
| **安全默认值** | `TOOL_DEFAULTS` dict | Fail-closed，必须显式声明安全属性 |
| **工厂函数** | `build_tool(**kwargs)` | 自动填充默认值，简化工具定义 |
| **别名系统** | `aliases` + `tool_matches_name()` | 工具改名后保持向后兼容 |
| **延迟加载** | `should_defer` / `always_load` | 减少 prompt token 消耗 |
| **上下文修改** | `context_modifier: Callable` | 工具可以修改后续工具的执行环境 |
| **进度回调** | `ToolCallProgress` Protocol | 流式报告执行进度 |
| **数据类** | `@dataclass` | 替代 TypeScript 的 type alias |

---

## 11. 面试要点

1. **为什么 is_concurrency_safe 默认 False？** — 安全优先。如果默认 True，一个有副作用的工具被并行执行可能导致竞态条件和数据损坏。开发者必须显式声明工具是并发安全的。

2. **ToolUseContext 为什么这么大？** — 它是工具执行的"万能环境"，包含了状态管理、权限、MCP 连接、Agent 定义等所有运行时信息。这避免了全局变量的使用，使工具可以安全地在子 Agent 中复用。

3. **build_tool 的类型系统如何工作？** — Python 使用 `**TOOL_DEFAULTS, **overrides` 的字典合并策略。后传入的字典覆盖先传入的同名键。运行时就是简单的字典展开合并。

4. **max_result_size_chars 的作用？** — 防止工具返回超大结果撑爆上下文窗口。超过限制的结果会被持久化到磁盘，模型只看到预览和文件路径。Read 工具设为 `float('inf')` 因为它有自己的内部分页机制。
