# 08 - 大规模并行编排最佳实践 (batch.ts)

> 源文件: `src/skills/bundled/batch.ts`
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

```typescript
// src/skills/bundled/batch.ts:30
const WORKER_INSTRUCTIONS = `After you finish implementing the change:
1. Simplify — Invoke the SkillTool with skill: "simplify"
2. Run unit tests
3. Test end-to-end (follow the e2e test recipe)
4. Commit and push — Create a PR with gh pr create
5. Report — End with: PR: <url> or PR: none — <reason>`
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

```typescript
// 关键: 所有 Agent 在单条消息中启动
// Claude 必须在一个 message 中包含多个 AgentTool 调用
AgentTool({
  name: "unit-1",
  description: "Migrate src/components/",
  subagent_type: "general-purpose",
  isolation: "worktree",
  run_in_background: true,
  prompt: "..."
})
AgentTool({
  name: "unit-2",
  description: "Migrate src/hooks/",
  subagent_type: "general-purpose",
  isolation: "worktree",
  run_in_background: true,
  prompt: "..."
})
// ... 更多 Agent
```

```
┌──────────────────────────────────────────────────────────────┐
│  Worker 启动策略                                              │
│                                                              │
│  单消息多 tool_use:                                          │
│  ├── Claude 在一条消息中发出所有 AgentTool 调用               │
│  ├── query.ts 的 runToolsConcurrently 并行执行                │
│  └── 所有 Worker 同时启动                                     │
│                                                              │
│  isolation: "worktree":                                      │
│  ├── 每个 Worker 在独立的 git worktree 中工作                │
│  ├── 互不干扰，可以安全并行修改代码                           │
│  ├── 无变更 → 自动清理                                       │
│  └── 有变更 → 返回 worktree 路径和分支名                     │
│                                                              │
│  run_in_background: true:                                    │
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
│  "4/5 units landed as PRs. 1 failed (test failure in src/utils/)"│
└──────────────────────────────────────────────────────────────┘
```

---

## 5. 技能注册代码

```typescript
// src/skills/bundled/batch.ts:100
export function registerBatchSkill(): void {
  registerBundledSkill({
    name: 'batch',
    description:
      'Research and plan a large-scale change, then execute it in parallel across 5–30 isolated worktree agents that each open a PR.',
    whenToUse:
      'Use when the user wants to make a sweeping, mechanical change across many files (migrations, refactors, bulk renames) that can be decomposed into independent parallel units.',
    argumentHint: '<instruction>',
    userInvocable: true,
    disableModelInvocation: true,  // 模型不能自动触发 /batch
    async getPromptForCommand(args) {
      const instruction = args.trim()
      if (!instruction) {
        return [{ type: 'text', text: MISSING_INSTRUCTION_MESSAGE }]
      }
      const isGit = await getIsGit()
      if (!isGit) {
        return [{ type: 'text', text: NOT_A_GIT_REPO_MESSAGE }]
      }
      return [{ type: 'text', text: buildPrompt(instruction) }]
    },
  })
}
```

**关键设计决策**:
- `disableModelInvocation: true` — `/batch` 只能由用户显式调用，模型不能自动触发（太危险）
- `userInvocable: true` — 用户可以通过 `/batch` 命令调用
- 前置检查: 必须是 git 仓库，必须有指令参数

---

## 6. Worker 指令模板

```typescript
const WORKER_INSTRUCTIONS = `After you finish implementing the change:
1. **Simplify** — Invoke the SkillTool with skill: "simplify" to review and clean up your changes.
2. **Run unit tests** — Run the project's test suite. If tests fail, fix them.
3. **Test end-to-end** — Follow the e2e test recipe from the coordinator's prompt.
4. **Commit and push** — Commit all changes, push, and create a PR with gh pr create.
5. **Report** — End with a single line: PR: <url> so the coordinator can track it.`
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
│     ├── disableModelInvocation: true                         │
│     ├── 前置检查 (git 仓库、非空指令)                        │
│     └── 用户审批环节                                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. 面试要点

1. **为什么 /batch 需要 Plan Mode？** — 大规模变更一旦开始执行就很难回滚。Plan Mode 让用户在执行前审查和修改计划，避免大规模错误。

2. **为什么 Worker 必须在 worktree 中执行？** — 如果所有 Worker 在同一个目录中并行修改文件，会产生严重的冲突和竞态条件。Worktree 提供了完全隔离的工作环境。

3. **"PR: \<url\>" 格式为什么重要？** — Coordinator 需要从 Worker 的结果中提取 PR 链接来更新进度表格。结构化的输出格式使解析变得简单可靠。

4. **如何处理 Worker 失败？** — Worker 在报告时使用 "PR: none — \<reason\>" 格式说明失败原因。Coordinator 在最终汇总中列出失败单元及其原因，用户可以手动修复。

5. **这个模式可以推广到其他场景吗？** — 可以。Plan-then-Execute + 隔离执行 + 并行编排 + 标准化 Worker 的模式适用于任何大规模并行任务，不仅限于代码变更。
