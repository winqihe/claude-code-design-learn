# 06 - 技能系统 (bundledSkills)

> 源文件: `src/skills/bundledSkills.ts`, `src/skills/bundled/`, `src/skills/loadSkillsDir.ts`
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

```typescript
// src/skills/bundledSkills.ts:15
export type BundledSkillDefinition = {
  name: string                    // 技能名称 (如 "batch")
  description: string             // 描述
  aliases?: string[]              // 别名
  whenToUse?: string              // 何时使用 (给 LLM 的指导)
  argumentHint?: string           // 参数提示 (如 "<instruction>")
  allowedTools?: string[]         // 可用工具限制
  model?: string                  // 指定使用的模型
  disableModelInvocation?: boolean // 禁止模型自动调用
  userInvocable?: boolean         // 用户是否可以用 /skill 调用
  isEnabled?: () => boolean       // 是否启用 (动态判断)
  hooks?: HooksSettings           // 关联的 hooks
  context?: 'inline' | 'fork'     // 执行上下文模式
  agent?: string                  // 关联的 Agent 类型
  files?: Record<string, string>  // 附带参考文件
  getPromptForCommand: (          // 核心: 生成 prompt
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}
```

### 2.2 registerBundledSkill — 注册函数

```typescript
// src/skills/bundledSkills.ts:53
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const { files } = definition
  let skillRoot: string | undefined
  let getPromptForCommand = definition.getPromptForCommand

  // 如果有附带文件，提取到磁盘
  if (files && Object.keys(files).length > 0) {
    skillRoot = getBundledSkillExtractDir(definition.name)
    let extractionPromise: Promise<string | null> | undefined
    const inner = definition.getPromptForCommand
    getPromptForCommand = async (args, ctx) => {
      extractionPromise ??= extractBundledSkillFiles(definition.name, files)
      const extractedDir = await extractionPromise
      const blocks = await inner(args, ctx)
      if (extractedDir === null) return blocks
      return prependBaseDir(blocks, extractedDir)
    }
  }

  // 注册为 Command
  const command: Command = {
    type: 'prompt',
    name: definition.name,
    description: definition.description,
    aliases: definition.aliases,
    allowedTools: definition.allowedTools ?? [],
    argumentHint: definition.argumentHint,
    whenToUse: definition.whenToUse,
    model: definition.model,
    disableModelInvocation: definition.disableModelInvocation ?? false,
    userInvocable: definition.userInvocable ?? true,
    source: 'bundled',
    loadedFrom: 'bundled',
    hooks: definition.hooks,
    skillRoot,
    context: definition.context,
    agent: definition.agent,
    isEnabled: definition.isEnabled,
    getPromptForCommand,
  }
  bundledSkills.push(command)
}
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
│  ├── 每个 Worker: isolation="worktree", run_in_background=true│
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

```typescript
// 用法
/loop 5m check the deploy
/loop 30m /babysit-prs
/loop 1h /standup 1
```

```
┌──────────────────────────────────────────────────────────────┐
│  /loop 技能流程                                               │
│                                                              │
│  输入解析 (优先级):                                           │
│  1. 前导 token: "5m /foo" → interval=5m, prompt="/foo"      │
│  2. 尾部 "every": "check every 20m" → interval=20m            │
│  3. 默认: "check deploy" → interval=10m                      │
│                                                              │
│  Interval → Cron 转换:                                       │
│  ├── Nm (N≤59): */N * * * *                                 │
│  ├── Nm (N≥60): 0 */H * * * (H=N/60)                       │
│  ├── Nh (N≤23): 0 */N * * *                                 │
│  └── Nd: 0 0 */N * *                                        │
│                                                              │
│  执行:                                                       │
│  1. 调用 CronCreateTool 创建定时任务                          │
│  2. 立即执行一次 (不等第一次 cron 触发)                       │
│  3. 自动过期: DEFAULT_MAX_AGE_DAYS 天后                      │
│  4. 可用 CronDeleteTool 取消                                 │
└──────────────────────────────────────────────────────────────┘
```

### 3.3 /verify — 验证代码变更

```typescript
// 仅限内部使用 (process.env.USER_TYPE === 'ant')
registerBundledSkill({
  name: 'verify',
  description: 'Verify a code change does what it should by running the app.',
  userInvocable: true,
  files: SKILL_FILES,  // 附带参考文件
  async getPromptForCommand(args) {
    const parts = [SKILL_BODY.trimStart()]
    if (args) parts.push(`## User Request\n\n${args}`)
    return [{ type: 'text', text: parts.join('\n\n') }]
  },
})
```

### 3.4 /debug — 调试辅助

帮助 Claude 在卡住时恢复，提供调试策略。

### 3.5 /stuck — 卡住恢复

当 Claude 陷入循环或无法继续时，提供恢复策略。

---

## 4. 附带文件系统

```typescript
// src/skills/bundledSkills.ts:131
async function extractBundledSkillFiles(
  skillName: string,
  files: Record<string, string>,
): Promise<string | null> {
  const dir = getBundledSkillExtractDir(skillName)
  try {
    await writeSkillFiles(dir, files)
    return dir
  } catch (e) {
    return null  // 失败不影响技能运行
  }
}
```

```
┌──────────────────────────────────────────────────────────┐
│  附带文件系统                                              │
│                                                          │
│  目的: 技能可能需要参考文件 (如配置模板、检查清单)        │
│                                                          │
│  流程:                                                    │
│  1. 技能定义时声明 files: { "template.yaml": "..." }    │
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
│  ├── 旧方案: 在 getAttachmentMessages 中同步查找              │
│  └── 阻塞了整个查询循环                                      │
│                                                              │
│  新方案: 异步预取 (skillPrefetch)                             │
│  ├── 在 LLM 推理和工具执行期间并行查找                       │
│  ├── 使用 findWritePivot 守卫 (非写操作轮次提前返回)          │
│  ├── 查找结果在工具执行后注入为 attachment                   │
│  └── Turn 0 用户输入发现仍然阻塞 (唯一信号)                  │
│                                                              │
│  代码位置:                                                   │
│  ├── query.ts:331 — 启动预取                                │
│  ├── query.ts:1620 — 收集预取结果                            │
│  └── services/skillSearch/prefetch.js — 预取实现             │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. 用户自定义技能

```typescript
// src/skills/loadSkillsDir.ts
// 从指定目录加载用户自定义技能
```

```
┌──────────────────────────────────────────────────────────┐
│  技能加载来源                                              │
│                                                          │
│  1. 内置技能 (bundled)                                    │
│     ├── 编译到 CLI 二进制中                               │
│     ├── registerBundledSkill() 注册                      │
│     └── 所有用户可用                                     │
│                                                          │
│  2. 用户自定义技能 (directory)                             │
│     ├── 从配置目录加载                                    │
│     ├── loadSkillsDir() 扫描                             │
│     └── 用户可以用 /skills 管理                           │
│                                                          │
│  3. 动态发现 (discovery)                                  │
│     ├── 基于当前上下文发现相关技能                         │
│     ├── skillPrefetch 异步预取                            │
│     └── 注入为 attachment 消息                             │
└──────────────────────────────────────────────────────────┘
```

---

## 7. 面试要点

1. **技能和工具的本质区别？** — 工具是原子操作（读文件、执行命令），技能是工作流模板（编排多个工具和 Agent 的使用方式）。技能本质上是一段精心设计的 prompt，告诉 Claude 如何完成特定类型的任务。

2. **为什么 /batch 需要 git worktree？** — /batch 启动 5-30 个并行 Agent，每个 Agent 需要独立修改代码。Git worktree 提供了隔离的工作目录，避免并行修改冲突。

3. **附带文件系统的安全设计？** — 使用进程唯一 nonce 目录防止预创建符号链接攻击，O_NOFOLLOW|O_EXCL 防止竞争条件，路径遍历检查防止目录逃逸。提取失败时优雅降级，不影响技能运行。

4. **技能发现为什么从同步改为异步？** — 因为 97% 的发现调用什么都没找到，同步调用白白阻塞了查询循环。异步预取利用 LLM 推理和工具执行的等待时间，零额外延迟。
