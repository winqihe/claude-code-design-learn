# 06 - 技能系统 (bundled_skills.py)

> 对应源文件: `src/skills/bundledSkills.ts`, `src/skills/bundled/`, `src/skills/loadSkillsDir.ts`
> Claude Code 的可扩展工作流系统，支持内置技能和用户自定义技能。

---

## 1. 技能系统概述

技能 (Skill) 是预定义的工作流模板，告诉 Claude 如何完成特定类型的任务。与普通工具不同，技能是**更高层次的工作流编排**。

```
┌──────────────────────────────────────────────────────────────┐
│  工具 vs 技能 vs Agent                                        │
│                                                              │
│  工具 (Tool): 原子操作                                       │
│  ├── Read, Write, Edit, Bash, Grep, Glob...                 │
│  └── 单一功能，由 LLM 直接调用                               │
│                                                              │
│  技能 (Skill): 工作流模板                                     │
│  ├── /batch, /loop, /verify, /debug, /stuck                │
│  └── 编排多个工具和 Agent 的使用方式                         │
│                                                              │
│  Agent: 自主执行者                                           │
│  ├── 子 Agent，有完整推理能力                                │
│  └── 可以自主决策使用哪些工具                                │
│                                                              │
│  关系: 技能指导 Agent 如何使用工具                            │
│  ┌────────────────────────────────────────────────────┐      │
│  │  /batch 技能                                       │      │
│  │  ├── Phase 1: 用 AgentTool 启动研究 Agent           │      │
│  │  ├── Phase 2: 用 AgentTool 启动 5-30 个 Worker     │      │
│  │  └── Phase 3: 渲染进度表格                          │      │
│  └────────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 技能注册机制

### 2.1 BundledSkillDefinition 类型

```python
from dataclasses import dataclass, field
from typing import Any, Callable, Awaitable

@dataclass
class BundledSkillDefinition:
    name: str
    description: str
    aliases: list[str] = field(default_factory=list)
    when_to_use: str | None = None
    argument_hint: str | None = None
    allowed_tools: list[str] = field(default_factory=list)
    model: str | None = None
    disable_model_invocation: bool = False
    user_invocable: bool = True
    is_enabled: Callable[[], bool] | None = None
    hooks: dict[str, Any] | None = None
    context: str | None = None  # "inline" | "fork"
    agent: str | None = None
    files: dict[str, str] | None = None
    get_prompt_for_command: Callable[
        [str, "ToolUseContext"], Awaitable[list[dict]]
    ] | None = None
```

### 2.2 register_bundled_skill — 注册函数

```python
bundled_skills: list[Command] = []

def register_bundled_skill(definition: BundledSkillDefinition) -> None:
    files = definition.files
    skill_root: str | None = None
    get_prompt_for_command = definition.get_prompt_for_command

    if files and len(files) > 0:
        skill_root = get_bundled_skill_extract_dir(definition.name)
        extraction_promise: asyncio.Task | None = None
        original_get_prompt = definition.get_prompt_for_command

        async def wrapped_get_prompt(args: str, ctx: ToolUseContext) -> list[dict]:
            nonlocal extraction_promise
            if extraction_promise is None:
                extraction_promise = asyncio.create_task(
                    extract_bundled_skill_files(definition.name, files)
                )
            extracted_dir = await extraction_promise
            blocks = await original_get_prompt(args, ctx)
            if extracted_dir is None:
                return blocks
            return prepend_base_dir(blocks, extracted_dir)

        get_prompt_for_command = wrapped_get_prompt

    command = Command(
        type="prompt",
        name=definition.name,
        description=definition.description,
        aliases=definition.aliases,
        allowed_tools=definition.allowed_tools,
        argument_hint=definition.argument_hint,
        when_to_use=definition.when_to_use,
        model=definition.model,
        disable_model_invocation=definition.disable_model_invocation,
        user_invocable=definition.user_invocable,
        source="bundled",
        loaded_from="bundled",
        hooks=definition.hooks,
        skill_root=skill_root,
        context=definition.context,
        agent=definition.agent,
        is_enabled=definition.is_enabled,
        get_prompt_for_command=get_prompt_for_command,
    )
    bundled_skills.append(command)
```

---

## 3. 内置技能详解

### 3.1 /batch — 大规模并行变更

```
┌──────────────────────────────────────────────────────────────┐
│  /batch 技能流程                                              │
│                                                              │
│  用户: /batch migrate from react to vue                       │
│                    │                                         │
│                    ▼                                         │
│  Phase 1: 研究和规划 (Plan Mode)                              │
│  ├── EnterPlanMode                                           │
│  ├── 启动子 Agent 深入研究代码库                              │
│  ├── 分解为 5-30 个独立工作单元                               │
│  ├── 确定端到端测试方案                                       │
│  ├── 编写计划文件                                             │
│  └── ExitPlanMode → 用户审批                                  │
│                    │                                         │
│                    ▼                                         │
│  Phase 2: 启动 Worker (全部并行)                              │
│  ├── 每个 Worker: isolation="worktree", run_in_background=True│
│  ├── Worker 内部流程:                                         │
│  │   1. 实现变更                                             │
│  │   2. /simplify (简化代码)                                 │
│  │   3. 运行单元测试                                         │
│  │   4. 端到端测试                                           │
│  │   5. git commit + push + gh pr create                    │
│  │   6. 报告: "PR: <url>"                                   │
│  └── 所有 Worker 在单条消息中启动 (并行)                      │
│                    │                                         │
│                    ▼                                         │
│  Phase 3: 跟踪进度                                            │
│  ├── 渲染状态表格: # | Unit | Status | PR                   │
│  ├── 收到完成通知后更新表格                                   │
│  └── 最终汇总: "22/24 units landed as PRs"                   │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 /loop — 定时循环任务

```python
import re

def parse_loop_interval(args: str) -> tuple[int, str, str]:
    """
    解析 /loop 的时间间隔参数

    优先级:
    1. 前导 token: "5m /foo" → interval=5m, prompt="/foo"
    2. 尾部 "every": "check every 20m" → interval=20m
    3. 默认: "check deploy" → interval=10m
    """
    match = re.match(r"^(\d+)([mhd])\s+(.+)$", args)
    if match:
        amount = int(match.group(1))
        unit = match.group(2)
        prompt = match.group(3)
        return amount, unit, prompt

    match = re.search(r"every\s+(\d+)([mhd])", args)
    if match:
        amount = int(match.group(1))
        unit = match.group(2)
        prompt = args
        return amount, unit, prompt

    return 10, "m", args

def interval_to_cron(amount: int, unit: str) -> str:
    """将时间间隔转换为 cron 表达式"""
    match unit:
        case "m":
            if amount <= 59:
                return f"*/{amount} * * * *"
            hours = amount // 60
            return f"0 */{hours} * * *"
        case "h":
            return f"0 */{amount} * * *"
        case "d":
            return f"0 0 */{amount} * *"
    return "*/10 * * * *"
```

### 3.3 /verify — 验证代码变更

```python
def register_verify_skill() -> None:
    register_bundled_skill(BundledSkillDefinition(
        name="verify",
        description="Verify a code change does what it should by running the app.",
        user_invocable=True,
        files=SKILL_FILES,
        async def get_prompt_for_command(args: str, ctx: ToolUseContext):
            parts = [SKILL_BODY.strip()]
            if args:
                parts.append(f"## User Request\n\n{args}")
            return [{"type": "text", "text": "\n\n".join(parts)}]
    ))
```

### 3.4 /debug — 调试辅助

帮助 Claude 在卡住时恢复，提供调试策略。

### 3.5 /stuck — 卡住恢复

当 Claude 陷入循环或无法继续时，提供恢复策略。

---

## 4. 附带文件系统

```python
import os
import tempfile
from pathlib import Path

def get_bundled_skill_extract_dir(skill_name: str) -> str:
    nonce = os.urandom(8).hex()
    return os.path.expanduser(f"~/.claude/skills/{skill_name}-{nonce}")

async def extract_bundled_skill_files(
    skill_name: str,
    files: dict[str, str],
) -> str | None:
    dir_path = get_bundled_skill_extract_dir(skill_name)
    try:
        os.makedirs(dir_path, mode=0o700, exist_ok=True)
        await write_skill_files(dir_path, files)
        return dir_path
    except Exception:
        return None

async def write_skill_files(dir_path: str, files: dict[str, str]) -> None:
    for filename, content in files.items():
        filepath = Path(dir_path) / filename

        if ".." in Path(filename).parts or Path(filename).is_absolute():
            raise ValueError(f"Invalid file path: {filename}")

        filepath.parent.mkdir(parents=True, exist_ok=True)
        flags = os.O_WRONLY | os.O_CREAT | os.O_EXCL | os.O_NOFOLLOW
        fd = os.open(str(filepath), flags, 0o600)
        try:
            os.write(fd, content.encode("utf-8"))
        finally:
            os.close(fd)
```

```
┌──────────────────────────────────────────────────────────┐
│  附带文件系统                                              │
│                                                          │
│  目的: 技能可能需要参考文件 (如配置模板、检查清单)        │
│                                                          │
│  流程:                                                    │
│  1. 技能定义时声明 files: {"template.yaml": "..."}      │
│  2. 首次调用时提取到 ~/.claude/skills/<name>/            │
│  3. Prompt 前缀: "Base directory for this skill: <dir>" │
│  4. Claude 可以用 Read/Grep 读取这些文件                  │
│                                                          │
│  安全措施:                                                │
│  ├── 每进程唯一 nonce 目录 (防止符号链接攻击)             │
│  ├── 0o700 权限 (仅 owner 可访问)                        │
│  ├── O_NOFOLLOW | O_EXCL (防止符号链接竞争)              │
│  ├── 路径遍历检查 (拒绝 .. 和绝对路径)                    │
│  └── 提取失败不影响技能运行 (优雅降级)                    │
└──────────────────────────────────────────────────────────┘
```

---

## 5. 技能发现 (Skill Discovery)

```
┌──────────────────────────────────────────────────────────────┐
│  技能发现机制                                                │
│                                                              │
│  问题: 97% 的 skill discovery 调用什么都没找到 (prod 数据)    │
│  ├── 旧方案: 在 get_attachment_messages 中同步查找              │
│  └── 阻塞了整个查询循环                                      │
│                                                              │
│  新方案: 异步预取 (skill_prefetch)                            │
│  ├── 在 LLM 推理和工具执行期间并行查找                       │
│  ├── 使用 find_write_pivot 守卫 (非写操作轮次提前返回)          │
│  ├── 查找结果在工具执行后注入为 attachment                   │
│  └── Turn 0 用户输入发现仍然阻塞 (唯一信号)                  │
│                                                              │
│  Python 实现:                                               │
│  ├── query.py:331 — 启动预取 (asyncio.create_task)          │
│  ├── query.py:1620 — 收集预取结果 (await task)               │
│  └── services/skill_search/prefetch.py — 预取实现            │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. 用户自定义技能

```python
from pathlib import Path

def load_skills_dir(directory: str) -> list[Command]:
    """从指定目录加载用户自定义技能"""
    skills: list[Command] = []
    dir_path = Path(directory)

    if not dir_path.exists():
        return skills

    for skill_file in sorted(dir_path.glob("*.py")):
        if skill_file.name.startswith("_"):
            continue
        try:
            skill = load_skill_from_file(skill_file)
            if skill:
                skills.append(skill)
        except Exception as e:
            logger.warning(f"Failed to load skill {skill_file}: {e}")

    return skills
```

```
┌──────────────────────────────────────────────────────────┐
│  技能加载来源                                              │
│                                                          │
│  1. 内置技能 (bundled)                                    │
│     ├── 编译到 CLI 二进制中                               │
│     ├── register_bundled_skill() 注册                    │
│     └── 所有用户可用                                     │
│                                                          │
│  2. 用户自定义技能 (directory)                             │
│     ├── 从配置目录加载                                    │
│     ├── load_skills_dir() 扫描                           │
│     └── 用户可以用 /skills 管理                           │
│                                                          │
│  3. 动态发现 (discovery)                                  │
│     ├── 基于当前上下文发现相关技能                         │
│     ├── skill_prefetch 异步预取                           │
│     └── 注入为 attachment 消息                             │
└──────────────────────────────────────────────────────────┘
```

---

## 7. 面试要点

1. **技能和工具的本质区别？** — 工具是原子操作（读文件、执行命令），技能是工作流模板（编排多个工具和 Agent 的使用方式）。技能本质上是一段精心设计的 prompt，告诉 Claude 如何完成特定类型的任务。

2. **为什么 /batch 需要 git worktree？** — /batch 启动 5-30 个并行 Agent，每个 Agent 需要独立修改代码。Git worktree 提供了隔离的工作目录，避免并行修改冲突。

3. **附带文件系统的安全设计？** — 使用进程唯一 nonce 目录防止预创建符号链接攻击，`O_NOFOLLOW | O_EXCL` 防止竞争条件，路径遍历检查防止目录逃逸。提取失败时优雅降级，不影响技能运行。

4. **技能发现为什么从同步改为异步？** — 因为 97% 的发现调用什么都没找到，同步调用白白阻塞了查询循环。异步预取利用 LLM 推理和工具执行的等待时间，零额外延迟。Python 中用 `asyncio.create_task` 实现并行预取。

5. **Python 中如何实现 interval_to_cron？** — 使用 `match/case` 结构化模式匹配（Python 3.10+），将用户友好的时间间隔（如 `5m`、`2h`、`1d`）转换为标准 cron 表达式。
