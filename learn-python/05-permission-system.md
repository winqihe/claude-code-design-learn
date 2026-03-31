# 05 - 权限系统 (use_can_use_tool.py)

> 对应源文件: `src/hooks/useCanUseTool.tsx`
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
│  │  has_permissions_to_use_tool()       │                        │
│  │                                      │                        │
│  │  always_allow_rules:                 │                        │
│  │    "Bash(git *)" → allow             │                        │
│  │    "Read(*.ts)"  → allow             │                        │
│  │                                      │                        │
│  │  always_deny_rules:                  │                        │
│  │    "Bash(rm -rf *)" → deny           │                        │
│  │                                      │                        │
│  │  always_ask_rules:                   │                        │
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

```python
from typing import Protocol, TypeVar, Any

InputT = TypeVar("InputT", bound=dict)

class CanUseToolFn(Protocol[InputT]):
    async def __call__(
        self,
        tool: Tool,
        input: InputT,
        tool_use_context: ToolUseContext,
        assistant_message: AssistantMessage,
        tool_use_id: str,
        force_decision: PermissionDecision | None = None,
    ) -> PermissionDecision: ...
```

**设计要点**:
- 返回 `Awaitable` — 权限检查可能是异步的（等待用户输入、调用分类器）
- `force_decision` 参数 — 允许跳过权限检查（用于 plan mode 等场景）
- `PermissionDecision` 包含 `behavior` (allow/deny/ask) 和 `updated_input`

---

## 3. use_can_use_tool 实现

```python
from dataclasses import dataclass

@dataclass
class PermissionDecision:
    behavior: str  # "allow" | "deny" | "ask"
    updated_input: Any = None
    decision_reason: dict | None = None

async def can_use_tool(
    tool: Tool,
    input: dict,
    tool_use_context: ToolUseContext,
    assistant_message: AssistantMessage,
    tool_use_id: str,
    force_decision: PermissionDecision | None = None,
    set_tool_use_confirm_queue=None,
    set_tool_permission_context=None,
) -> PermissionDecision:
    ctx = create_permission_context(
        tool, input, tool_use_context, assistant_message, tool_use_id
    )

    if ctx.resolve_if_aborted():
        return PermissionDecision(behavior="deny")

    if force_decision is not None:
        result = force_decision
    else:
        result = await has_permissions_to_use_tool(
            tool, input, tool_use_context
        )

    if result.behavior == "allow":
        return PermissionDecision(
            behavior="allow",
            updated_input=result.updated_input or input,
        )

    description = await tool.description(input, tool_use_context)

    if result.behavior == "deny":
        return result

    if result.behavior == "ask":
        coordinator_decision = await handle_coordinator_permission(
            ctx, result, tool_use_context
        )
        if coordinator_decision:
            return coordinator_decision

        swarm_decision = await handle_swarm_worker_permission(
            ctx, description, result
        )
        if swarm_decision:
            return swarm_decision

        interactive_decision = await handle_interactive_permission(
            ctx, description, result
        )
        return interactive_decision
```

---

## 4. 权限模式 (PermissionMode)

```python
from enum import Enum

class PermissionMode(Enum):
    DEFAULT = "default"
    PLAN = "plan"
    BYPASS_PERMISSIONS = "bypassPermissions"
    AUTO = "auto"
```

```
┌──────────────────────────────────────────────────────────────┐
│  权限模式                                                     │
│                                                              │
│  default (默认)                                              │
│  ├── 未匹配规则的工具需要用户确认                              │
│  ├── 匹配 always_allow 的自动放行                              │
│  ├── 匹配 always_deny 的自动拒绝                               │
│  └── 适合: 日常交互使用                                       │
│                                                              │
│  plan (计划模式)                                              │
│  ├── 只允许只读操作                                          │
│  ├── 写操作需要用户确认                                       │
│  └── 适合: 规划阶段，防止意外修改                             │
│                                                              │
│  bypass_permissions (绕过权限)                                │
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

```python
from dataclasses import dataclass
import fnmatch
import re

@dataclass
class PermissionRule:
    tool: str
    pattern: str | None = None
    behavior: str  # "allow" | "deny" | "ask"

def match_rule(rule: PermissionRule, tool_name: str, tool_input: dict) -> bool:
    if not fnmatch.fnmatch(tool_name, rule.tool):
        return False
    if rule.pattern:
        input_str = json.dumps(tool_input)
        return bool(re.match(rule.pattern, input_str))
    return True

def check_permission_rules(
    rules: list[PermissionRule],
    tool_name: str,
    tool_input: dict,
) -> str | None:
    for rule in rules:
        if match_rule(rule, tool_name, tool_input):
            return rule.behavior
    return None
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
│  1. always_deny (最高优先级)                             │
│  2. always_ask                                            │
│  3. always_allow (最低优先级)                            │
└──────────────────────────────────────────────────────────┘
```

---

## 6. 三种权限处理器

### 6.1 Coordinator Handler

```python
async def handle_coordinator_permission(
    ctx: PermissionContext,
    result: PermissionDecision,
    tool_use_context: ToolUseContext,
) -> PermissionDecision | None:
    if not tool_use_context.options.get("is_coordinator_mode"):
        return None

    if result.behavior == "ask":
        if ctx.is_dangerous():
            return PermissionDecision(behavior="deny")
        return PermissionDecision(
            behavior="allow",
            updated_input=result.updated_input,
        )
    return None
```

### 6.2 Swarm Worker Handler

```python
async def handle_swarm_worker_permission(
    ctx: PermissionContext,
    description: str,
    result: PermissionDecision,
) -> PermissionDecision | None:
    if not ctx.is_swarm_worker():
        return None

    if ctx.is_dangerous():
        return PermissionDecision(behavior="deny")
    return PermissionDecision(
        behavior="allow",
        updated_input=result.updated_input,
    )
```

### 6.3 Interactive Handler

```python
async def handle_interactive_permission(
    ctx: PermissionContext,
    description: str,
    result: PermissionDecision,
) -> PermissionDecision:
    event = asyncio.Event()
    set_tool_use_confirm_queue(
        lambda queue: queue.append({
            "tool_use_id": ctx.tool_use_id,
            "description": description,
            "updated_input": result.updated_input,
            "on_decision": lambda decision: (
                event.set(),
                setattr(ctx, "decision", decision),
            ),
        })
    )
    await event.wait()
    return ctx.decision
```

---

## 7. 权限拒绝追踪

```python
@dataclass
class DenialTrackingState:
    count: int = 0
    last_denial_time: float = 0.0

    def record_denial(self) -> None:
        self.count += 1
        self.last_denial_time = time.time()

    def should_fallback_to_prompt(self, threshold: int = 3) -> bool:
        return self.count >= threshold
```

```
┌──────────────────────────────────────────────────────────┐
│  拒绝追踪机制                                              │
│                                                          │
│  问题: 异步子 Agent 的 set_app_state 是 no-op            │
│  ├── 拒绝计数器永远不会累加                               │
│  └── 回退到提示用户的阈值永远不会达到                       │
│                                                          │
│  解决: local_denial_tracking                             │
│  ├── 每个 Agent 本地维护拒绝计数                          │
│  ├── 可变对象，权限代码原地更新                           │
│  └── 达到阈值后回退到提示用户                              │
└──────────────────────────────────────────────────────────┘
```

---

## 8. 安全分类器 (TRANSCRIPT_CLASSIFIER)

```python
async def run_safety_classifier(
    tool: Tool,
    input: dict,
) -> str:
    classifier_input = tool.to_auto_classifier_input(input)
    if not classifier_input:
        return "safe"

    response = await call_classifier_api(classifier_input)
    return response.verdict  # "safe" | "unsafe" | "unsure"
```

```
┌──────────────────────────────────────────────────────────┐
│  安全分类器工作流程                                        │
│                                                          │
│  1. 提取工具调用的紧凑表示                                 │
│     to_auto_classifier_input(input)                      │
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
│     ├── set_yolo_classifier_approval (放行时)             │
│     └── record_auto_mode_denial (拒绝时)                  │
└──────────────────────────────────────────────────────────┘
```

---

## 9. 面试要点

1. **为什么需要三层权限？** — 规则匹配处理明确的策略（如"git 命令总是允许"），分类器处理模糊的情况（如"这个 bash 命令是否安全"），用户确认是最终兜底。三层互补，缺一不可。

2. **为什么异步 Agent 需要本地拒绝追踪？** — 异步 Agent 的 `set_app_state` 是 no-op（防止状态泄漏），但权限系统依赖拒绝计数来决定是否回退到提示用户。本地追踪用可变 dataclass 解决了这个问题。

3. **bypass_permissions 模式为什么存在？** — CI/CD 和自动化脚本中无法交互式确认，需要完全跳过权限检查。这是一个必要的"逃生舱口"。

4. **Python 中如何实现 fnmatch 规则匹配？** — 使用 `fnmatch.fnmatch()` 进行工具名匹配，`re.match()` 进行参数模式匹配。规则按优先级排序：deny > ask > allow。

5. **asyncio.Event 在交互式确认中的作用？** — `asyncio.Event` 作为信号量，权限检查协程 await event.wait() 挂起，用户在 UI 中点击确认后调用 event.set() 唤醒协程。这是 Python 异步编程中经典的"事件通知"模式。
