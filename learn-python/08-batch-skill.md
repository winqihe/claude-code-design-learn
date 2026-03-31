# 08 - 大规模并行编排最佳实践 (batch.py)

> 对应源文件: `src/skills/bundled/batch.ts`
> Claude Code 最复杂的内置技能，展示了如何编排 5-30 个并行 Agent 完成大规模代码变更。

---

## 1. /batch 技能概述

`/batch` 是 Claude Code 中最令人印象深刻的技能。它将一个大规模代码变更任务分解为多个独立的工作单元，然后在隔离的 Git Worktree 中并行执行，每个 Agent 独立完成自己的工作并创建 PR。

```
┌──────────────────────────────────────────────────────────────────┐
│  /batch 技能: 大规模并行代码变更                                  │
│                                                                  │
│  用户: /batch migrate from react to vue                           │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Coordinator (主 Agent)                                    │  │
│  │                                                            │  │
│  │  Phase 1: 研究和规划                                       │  │
│  │  ├── 启动研究 Agent 深入分析代码库                          │  │
│  │  ├── 分解为 5-30 个独立工作单元                            │  │
│  │  ├── 编写计划 → 用户审批                                   │  │
│  │                                                            │  │
│  │  Phase 2: 并行执行 (5-30 个 Worker Agent)                  │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐               │  │
│  │  │ W1  │ │ W2  │ │ W3  │ │ W4  │ │ ... │  ← 全部并行    │  │
│  │  │pr:12│ │pr:34│ │pr:56│ │pr:78│ │     │               │  │
│  │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘               │  │
│  │     │       │       │       │       │                    │  │
│  │  Phase 3: 收集结果                                       │  │
│  │  ├── # │ Unit | Status | PR                              │  │
│  │  ├── 1 | src/  | done   | github.com/.../pull/12         │  │
│  │  ├── 2 | comp/ | done   | github.com/.../pull/34         │  │
│  │  ├── 3 | util/ | failed | —                              │  │
│  │  └── ...                                                 │  │
│  │                                                            │  │
│  │  最终: "22/24 units landed as PRs"                        │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Phase 1: 研究和规划

```python
WORKER_INSTRUCTIONS = """After you finish implementing the change:
1. **Simplify** — Invoke the SkillTool with skill: "simplify" to review and clean up your changes.
2. **Run unit tests** — Run the project's test suite. If tests fail, fix them.
3. **Test end-to-end** — Follow the e2e test recipe from the coordinator's prompt.
4. **Commit and push** — Commit all changes, push, and create a PR with gh pr create.
5. **Report** — End with a single line: PR: <url> so the coordinator can track it."""
```

### 规划阶段的关键决策

```
┌──────────────────────────────────────────────────────────────┐
│  规划阶段的四个关键决策                                        │
│                                                              │
│  1. 工作分解 (5-30 个单元)                                   │
│     ├── 每个单元必须独立实现 (不依赖其他单元)                  │
│     ├── 每个单元可以在隔离的 worktree 中完成                  │
│     ├── 每个单元的 PR 可以独立合并                            │
│     ├── 单元大小应均匀 (大的拆分，小的合并)                   │
│     └── 按目录或模块切片 (而非随机文件列表)                   │
│                                                              │
│  2. 端到端测试方案                                           │
│     ├── 查找 claude-in-chrome 技能 (UI 变更)                 │
│     ├── 查找 tmux 技能 (CLI 变更)                            │
│     ├── 查找 dev-server + curl 模式 (API 变更)               │
│     ├── 查找现有 e2e 测试套件                                │
│     └── 找不到就问用户 (提供 2-3 个具体选项)                  │
│                                                              │
│  3. Worker 指令模板                                          │
│     ├── 包含整体目标                                         │
│     ├── 包含本单元的具体任务                                 │
│     ├── 包含代码库约定                                       │
│     ├── 包含 e2e 测试方案                                    │
│     └── 包含 WORKER_INSTRUCTIONS                             │
│                                                              │
│  4. Agent 类型选择                                           │
│     └── 默认 "general-purpose"，可指定更具体的类型           │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Phase 2: 并行 Worker 启动

```python
from pydantic import BaseModel, Field

class AgentToolInput(BaseModel):
    name: str = Field(description="Agent name (1-2 words)")
    description: str = Field(description="Task description (3-5 words)")
    query: str = Field(description="Detailed task instructions")
    subagent_type: str | None = Field(default=None)
    isolation: str | None = Field(default=None)
    run_in_background: bool = Field(default=False)

def build_worker_calls(units: list[WorkUnit]) -> list[dict]:
    """构建所有 Worker 的 AgentTool 调用参数"""
    calls = []
    for unit in units:
        calls.append({
            "name": f"unit-{unit.id}",
            "description": unit.description,
            "subagent_type": "general-purpose",
            "isolation": "worktree",
            "run_in_background": True,
            "query": f"""
## Overall Goal
{unit.overall_goal}

## Your Unit
{unit.description}

## Scope
Files: {unit.files}

## Codebase Conventions
{unit.conventions}

## E2E Test Recipe
{unit.e2e_recipe}

{WORKER_INSTRUCTIONS}
""".strip(),
        })
    return calls
```

```
┌──────────────────────────────────────────────────────────────┐
│  Worker 启动策略                                              │
│                                                              │
│  单消息多 tool_use:                                          │
│  ├── Claude 在一条消息中发出所有 AgentTool 调用               │
│  ├── query.py 的 run_tools_concurrently 并行执行               │
│  └── 所有 Worker 同时启动                                     │
│                                                              │
│  isolation: "worktree":                                      │
│  ├── 每个 Worker 在独立的 git worktree 中工作                │
│  ├── 互不干扰，可以安全并行修改代码                           │
│  ├── 无变更 → 自动清理                                       │
│  └── 有变更 → 返回 worktree 路径和分支名                     │
│                                                              │
│  run_in_background: True:                                    │
│  ├── 主 Agent 立即继续，不等待                               │
│  ├── Worker 完成后以 user-role 消息通知                       │
│  └── 主 Agent 收到通知后更新进度表格                          │
│                                                              │
│  Worker 内部流程:                                            │
│  1. 实现代码变更                                            │
│  2. /simplify — 简化和清理代码                               │
│  3. 运行单元测试                                            │
│  4. 执行 e2e 测试                                           │
│  5. git commit + push + gh pr create                        │
│  6. 报告: "PR: <url>"                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Phase 3: 进度跟踪

```python
from dataclasses import dataclass, field
from enum import Enum

class UnitStatus(Enum):
    RUNNING = "running"
    DONE = "done"
    FAILED = "failed"

@dataclass
class WorkUnit:
    id: int
    description: str
    files: list[str]
    status: UnitStatus = UnitStatus.RUNNING
    pr_url: str | None = None
    error: str | None = None

@dataclass
class BatchTracker:
    units: list[WorkUnit] = field(default_factory=list)

    def render_table(self) -> str:
        """渲染进度表格"""
        lines = ["| # | Unit | Status | PR |"]
        lines.append("|---|------|--------|-----|")
        for unit in self.units:
            pr = unit.pr_url or "—"
            if unit.error:
                pr = f"— ({unit.error})"
            lines.append(
                f"| {unit.id} | {unit.description} | {unit.status.value} | {pr} |"
            )
        return "\n".join(lines)

    def update_from_notification(self, notification: dict) -> None:
        """从 Worker 通知更新状态"""
        unit_id = notification.get("unit_id")
        report = notification.get("report", "")

        for unit in self.units:
            if unit.id == unit_id:
                if report.startswith("PR: ") and "none" not in report:
                    unit.status = UnitStatus.DONE
                    unit.pr_url = report.replace("PR: ", "").strip()
                elif "none" in report:
                    unit.status = UnitStatus.FAILED
                    reason = report.split("—")[-1].strip() if "—" in report else "unknown"
                    unit.error = reason
                break

    def summary(self) -> str:
        """生成最终汇总"""
        done = sum(1 for u in self.units if u.status == UnitStatus.DONE)
        total = len(self.units)
        failed = [u for u in self.units if u.status == UnitStatus.FAILED]
        msg = f"{done}/{total} units landed as PRs"
        if failed:
            failures = ", ".join(f"{u.description} ({u.error})" for u in failed)
            msg += f". {len(failed)} failed: {failures}"
        return msg
```

```
┌──────────────────────────────────────────────────────────────┐
│  进度跟踪机制                                                │
│                                                              │
│  初始表格:                                                   │
│  ┌───┬──────────────┬────────┬─────┐                       │
│  │ # │ Unit         │ Status │ PR  │                       │
│  ├───┼──────────────┼────────┼─────┤                       │
│  │ 1 │ src/comp/    │ running│ —   │                       │
│  │ 2 │ src/hooks/   │ running│ —   │                       │
│  │ 3 │ src/utils/   │ running│ —   │                       │
│  │ 4 │ tests/       │ running│ —   │                       │
│  │ 5 │ config/      │ running│ —   │                       │
│  └───┴──────────────┴────────┴─────┘                       │
│                                                              │
│  收到通知后更新:                                             │
│  ┌───┬──────────────┬────────┬─────────────────────┐       │
│  │ # │ Unit         │ Status │ PR                    │       │
│  ├───┼──────────────┼────────┼─────────────────────┤       │
│  │ 1 │ src/comp/    │ done   │ github.com/.../12    │       │
│  │ 2 │ src/hooks/   │ done   │ github.com/.../34    │       │
│  │ 3 │ src/utils/   │ failed │ — (test failure)     │       │
│  │ 4 │ tests/       │ running│ —                     │       │
│  │ 5 │ config/      │ running│ —                     │       │
│  └───┴──────────────┴────────┴─────────────────────┘       │
│                                                              │
│  最终汇总:                                                   │
│  "4/5 units landed as PRs. 1 failed (test failure)"         │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. 技能注册代码

```python
MISSING_INSTRUCTION_MESSAGE = "Error: /batch requires an instruction. Usage: /batch <instruction>"
NOT_A_GIT_REPO_MESSAGE = "Error: /batch requires a git repository."

async def is_git_repo() -> bool:
    """检查当前目录是否是 git 仓库"""
    proc = await asyncio.create_subprocess_exec(
        "git", "rev-parse", "--is-inside-work-tree",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    await proc.wait()
    return proc.returncode == 0

def register_batch_skill() -> None:
    register_bundled_skill(BundledSkillDefinition(
        name="batch",
        description=(
            "Research and plan a large-scale change, then execute it in parallel "
            "across 5-30 isolated worktree agents that each open a PR."
        ),
        when_to_use=(
            "Use when the user wants to make a sweeping, mechanical change across "
            "many files (migrations, refactors, bulk renames) that can be decomposed "
            "into independent parallel units."
        ),
        argument_hint="<instruction>",
        user_invocable=True,
        disable_model_invocation=True,
        async def get_prompt_for_command(args: str, ctx: ToolUseContext):
            instruction = args.strip()
            if not instruction:
                return [{"type": "text", "text": MISSING_INSTRUCTION_MESSAGE}]
            git_ok = await is_git_repo()
            if not git_ok:
                return [{"type": "text", "text": NOT_A_GIT_REPO_MESSAGE}]
            return [{"type": "text", "text": build_batch_prompt(instruction)}],
    ))
```

**关键设计决策**:
- `disable_model_invocation=True` — `/batch` 只能由用户显式调用，模型不能自动触发（太危险）
- `user_invocable=True` — 用户可以通过 `/batch` 命令调用
- 前置检查: 必须是 git 仓库，必须有指令参数

---

## 6. Worker 指令模板

```python
def build_worker_prompt(
    overall_goal: str,
    unit: WorkUnit,
    conventions: str,
    e2e_recipe: str,
) -> str:
    """构建 Worker 的完整 prompt"""
    return f"""
## Overall Goal
{overall_goal}

## Your Unit: {unit.description}

### Scope
Files to modify:
{chr(10).join(f"- {f}" for f in unit.files)}

### Codebase Conventions
{conventions}

### E2E Test Recipe
{e2e_recipe}

{WORKER_INSTRUCTIONS}

### Important
- Work ONLY within your assigned scope
- Do NOT modify files outside your unit
- If you encounter issues, report them clearly
- Your final message MUST start with "PR: " followed by the URL
""".strip()
```

```
┌──────────────────────────────────────────────────────────┐
│  Worker 指令设计原则                                      │
│                                                          │
│  1. 自包含 (Self-contained)                               │
│     ├── 每个 Worker 的 prompt 必须完整                    │
│     ├── 不依赖外部上下文                                  │
│     └── 包含目标、范围、约定、测试方案                     │
│                                                          │
│  2. 标准化流程 (Standardized)                             │
│     ├── 所有 Worker 遵循相同的 5 步流程                    │
│     ├── 简化 → 测试 → e2e → 提交 → 报告                  │
│     └── 便于 Coordinator 统一跟踪                          │
│                                                          │
│  3. 结果格式化 (Structured output)                        │
│     ├── "PR: <url>" — 成功                                │
│     └── "PR: none — <reason>" — 失败                      │
│                                                          │
│  4. 质量保证 (Quality gates)                              │
│     ├── /simplify 清理代码                                 │
│     ├── 单元测试必须通过                                   │
│     └── e2e 测试必须通过                                  │
└──────────────────────────────────────────────────────────┘
```

---

## 7. 从 /batch 学到的设计模式

```
┌──────────────────────────────────────────────────────────────┐
│  /batch 展示的核心设计模式                                    │
│                                                              │
│  1. Plan-then-Execute                                       │
│     ├── 先规划，获得用户审批后再执行                          │
│     ├── 避免大规模错误执行                                    │
│     └── 使用 EnterPlanMode / ExitPlanMode 工具               │
│                                                              │
│  2. 隔离执行 (Isolation)                                     │
│     ├── Git Worktree 提供文件系统隔离                        │
│     ├── 每个 Agent 独立工作，互不影响                        │
│     └── 无变更自动清理，减少资源浪费                          │
│                                                              │
│  3. 并行编排 (Parallel Orchestration)                        │
│     ├── 单消息多 tool_use 实现并行启动                       │
│     ├── 后台执行 + 通知机制                                  │
│     └── 进度表格实时更新                                     │
│                                                              │
│  4. 标准化 Worker (Standardized Workers)                     │
│     ├── 统一的 5 步流程                                      │
│     ├── 结构化输出格式                                       │
│     └── 内置质量门控                                         │
│                                                              │
│  5. 渐进式分解 (Progressive Decomposition)                   │
│     ├── 先研究再分解 (不盲目拆分)                            │
│     ├── 按目录/模块切片 (而非随机文件列表)                    │
│     └── 单元大小均匀化                                       │
│                                                              │
│  6. 安全门控 (Safety Gates)                                 │
│     ├── disable_model_invocation=True                        │
│     ├── 前置检查 (git 仓库、非空指令)                        │
│     └── 用户审批环节                                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Python 实现的额外考量

### 8.1 Git Worktree 管理

```python
import subprocess
import tempfile
import shutil

async def create_worktree(
    repo_root: str,
    branch_name: str,
) -> str:
    """创建临时 git worktree"""
    worktree_path = tempfile.mkdtemp(prefix="claude-batch-")
    proc = await asyncio.create_subprocess_exec(
        "git", "worktree", "add", worktree_path,
        "-b", branch_name,
        cwd=repo_root,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    _, stderr = await proc.communicate()
    if proc.returncode != 0:
        shutil.rmtree(worktree_path, ignore_errors=True)
        raise RuntimeError(f"Failed to create worktree: {stderr.decode()}")
    return worktree_path

async def cleanup_worktree(repo_root: str, worktree_path: str, has_changes: bool) -> None:
    """清理 git worktree"""
    if not has_changes:
        proc = await asyncio.create_subprocess_exec(
            "git", "worktree", "remove", worktree_path,
            "--force",
            cwd=repo_root,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        await proc.wait()
    else:
        proc = await asyncio.create_subprocess_exec(
            "git", "worktree", "remove", worktree_path,
            cwd=repo_root,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        await proc.wait()
```

### 8.2 PR 创建

```python
async def create_pr(
    worktree_path: str,
    title: str,
    body: str,
) -> str | None:
    """在 worktree 中创建 PR"""
    proc = await asyncio.create_subprocess_exec(
        "gh", "pr", "create",
        "--title", title,
        "--body", body,
        cwd=worktree_path,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    stdout, stderr = await proc.communicate()
    if proc.returncode != 0:
        return None
    return stdout.decode().strip()
```

---

## 9. 面试要点

1. **为什么 /batch 需要 Plan Mode？** — 大规模变更一旦开始执行就很难回滚。Plan Mode 让用户在执行前审查和修改计划，避免大规模错误。

2. **为什么 Worker 必须在 worktree 中执行？** — 如果所有 Worker 在同一个目录中并行修改文件，会产生严重的冲突和竞态条件。Worktree 提供了完全隔离的工作环境。

3. **"PR: \<url\>" 格式为什么重要？** — Coordinator 需要从 Worker 的结果中提取 PR 链接来更新进度表格。结构化的输出格式使解析变得简单可靠。Python 中用 `str.startswith("PR: ")` 即可判断。

4. **如何处理 Worker 失败？** — Worker 在报告时使用 "PR: none — \<reason\>" 格式说明失败原因。Coordinator 在最终汇总中列出失败单元及其原因，用户可以手动修复。

5. **这个模式可以推广到其他场景吗？** — 可以。Plan-then-Execute + 隔离执行 + 并行编排 + 标准化 Worker 的模式适用于任何大规模并行任务，不仅限于代码变更。Python 的 `asyncio` 天然适合这种并发编排。

6. **Python 中如何管理 30 个并行 Agent 的生命周期？** — 使用 `asyncio.TaskGroup`（Python 3.11+）或 `asyncio.gather()` 管理所有 Worker 协程。每个 Worker 是一个独立的 `asyncio.Task`，通过 `asyncio.Queue` 接收通知。`BatchTracker` 作为共享状态，用 `threading.Lock` 或 `asyncio.Lock` 保护并发访问。
