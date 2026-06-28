# TodoWrite 问答

> 本文档整理自学习 `s05_todo_write` 过程中的 TodoWrite / Task tools 相关问题，基于本仓库 `s05_todo_write` 章节、前置的 `s04_hooks` 机制，以及 Claude Code 官方文档说明。与 [Hooks 问答](hooks-qa.md)、[Permission 问答](permission-qa.md)、[Validation 问答](validation-qa.md) 互补：TodoWrite 管「如何把计划变成可见状态」。
>
> **创建日期**：2026-06-28 · **最后更新**：2026-06-28

---

## 目录

- [Q1：TodoWrite 到底新增了什么能力？](#q1todowrite-到底新增了什么能力)
  - [一句话概括](#一句话概括)
  - [它不是执行器，而是外部工作记忆](#它不是执行器而是外部工作记忆)
  - [s04 到 s05 的变化](#s04-到-s05-的变化)
- [Q2：如何确保 todo_write 会执行？计划如何不被海量上下文淹没？](#q2如何确保-todo_write-会执行计划如何不被海量上下文淹没)
  - [教学版是软约束，不是硬门禁](#教学版是软约束不是硬门禁)
  - [为什么 reminder 有用](#为什么-reminder-有用)
  - [如果要更强保证，需要 runtime 做硬约束](#如果要更强保证需要-runtime-做硬约束)
- [Q3：真实 Claude Code 是把 TodoWrite 做成 hook 吗？](#q3真实-claude-code-是把-todowrite-做成-hook-吗)
  - [不是 hook，而是内置工具 / 任务系统](#不是-hook而是内置工具--任务系统)
  - [TodoWrite V1 与 Task tools](#todowrite-v1-与-task-tools)
  - [Hooks 可以围绕任务系统做集成](#hooks-可以围绕任务系统做集成)
- [Q4：内置工具是否比 hook 更核心？](#q4内置工具是否比-hook-更核心)
  - [工具是 agent 行动能力，hook 是生命周期扩展点](#工具是-agent-行动能力hook-是生命周期扩展点)
  - [TaskCreate 与 TaskCreated 的区别](#taskcreate-与-taskcreated-的区别)
- [Q5：一个好 plan 是模型能力还是 harness 能力？](#q5一个好-plan-是模型能力还是-harness-能力)
  - [计划质量主要来自模型](#计划质量主要来自模型)
  - [可靠执行主要来自 harness](#可靠执行主要来自-harness)
  - [从 todo list 到 workflow engine](#从-todo-list-到-workflow-engine)
- [参考链接](#参考链接)

---

## Q1：TodoWrite 到底新增了什么能力？

### 一句话概括

**TodoWrite 给 agent 增加的是规划状态，不是执行能力。**

`todo_write` 不会读文件、不会写代码、不会跑命令。它只是让模型把「我要做什么、正在做什么、做完了什么」写成结构化状态，放到 agent runtime 可以展示、保存、检查、提醒的位置。

### 它不是执行器，而是外部工作记忆

没有 TodoWrite 时，模型也可能在脑子里想：

```text
先看文件 → 再改代码 → 再跑测试 → 再总结
```

但这个计划是隐性的，容易被后续工具结果、错误输出和长上下文冲掉。

有 TodoWrite 后，模型需要把计划显式写出来：

```json
[
  { "content": "Inspect current parser behavior", "status": "in_progress" },
  { "content": "Refactor parsing branch", "status": "pending" },
  { "content": "Add regression test", "status": "pending" },
  { "content": "Run focused tests", "status": "pending" }
]
```

这就把模型内部的临时想法，变成了外部、结构化、可追踪的状态。

### s04 到 s05 的变化

`s05_todo_write/code.py` 继承了 `s04` 的 hook 结构，真正新增的是三件事：

| 变化 | 代码位置 | 作用 |
|------|----------|------|
| `SYSTEM` 加入 planning guidance | `s05_todo_write/code.py` | 告诉模型多步骤任务前先计划 |
| 新增 `todo_write` 工具 | `TOOLS` / `TOOL_HANDLERS` | 让模型能主动写入任务列表 |
| 新增 `rounds_since_todo` reminder | `agent_loop()` | 太久没更新 todo 时提醒模型 |

所以 `s05` 的重点不是改工具分发机制，而是：**同一套 dispatch 机制里多了一个规划工具**。

---

## Q2：如何确保 todo_write 会执行？计划如何不被海量上下文淹没？

### 教学版是软约束，不是硬门禁

严格说，`s05` 教学版不能 100% 确保 `todo_write` 一定会执行。它靠三层软约束增加发生概率：

| 机制 | 作用 |
|------|------|
| System prompt | 告诉模型「多步骤任务前使用 `todo_write`」 |
| Tool schema | 把 `todo_write` 暴露给模型，模型才能选择调用 |
| Nag reminder | 连续 3 轮没有更新 todo，就把提醒插入最新上下文 |

关键在第三点。规划要求如果只写在最早的 system prompt 里，长任务中可能被大量 tool results 稀释；reminder 会在后续轮次重新插入一条新消息：

```text
<reminder>Update your todos.</reminder>
```

这不是让模型「永久记住」计划，而是在它可能忘的时候，把 planning pressure 重新放回上下文前沿。

### 为什么 reminder 有用

上下文里的信息不是等权的。越新的消息，通常越容易影响下一次模型行为。`s05` 的 reminder 机制就是利用这一点：

```text
早期 system prompt：你应该计划
  ↓
工具结果越来越多，规划要求变弱
  ↓
runtime 注入最新 reminder
  ↓
模型重新看到：该更新 todo 了
```

所以它解决的不是「模型不会忘」，而是「模型忘了以后，有机制把它拉回来」。

### 如果要更强保证，需要 runtime 做硬约束

如果目标是**确保**，仅靠 prompt 不够，需要 harness / runtime 做硬约束。例如：

| 强约束 | 含义 |
|--------|------|
| 没有 todo 时禁止执行普通工具 | 多步骤任务必须先 `todo_write` |
| 执行工具前必须存在 `in_progress` 项 | 每次行动都绑定当前任务 |
| 每轮 LLM 前自动注入当前 todo 摘要 | 让计划持续可见 |
| `todo_write` 太空泛时拒绝 | 防止写出 `Do task` 这类无效计划 |
| 完成所有任务前必须有验证项 | 防止跳过测试 / 检查 |

这时 `todo_write` 就不只是一个工具，而开始接近 workflow engine：runtime 不只展示计划，还会根据计划控制执行。

---

## Q3：真实 Claude Code 是把 TodoWrite 做成 hook 吗？

### 不是 hook，而是内置工具 / 任务系统

真实 Claude Code 中，TodoWrite / Task tools 不是作为 hook 实现的。更准确的分层是：

```text
TodoWrite / Task tools = 模型可主动调用的内置规划工具
Hooks = runtime 在生命周期事件上自动触发的扩展机制
```

也就是说，不是：

```text
hook 实现 todo
```

而是：

```text
todo/task 是工具；
task 变化可以触发 hook 或被外部代码监听。
```

原因很直接：todo/task 是模型需要主动创建、更新、读取的状态；hook 是事件发生时被 runtime 自动调用的扩展逻辑。规划状态应该作为模型可见的工具存在，而不是藏在 hook 里。

### TodoWrite V1 与 Task tools

按照 Claude Code 官方 Agent SDK 文档，截至 Claude Code `v2.1.142` / TypeScript Agent SDK `0.3.142`：

- `TodoWrite` 默认禁用
- 默认使用结构化 Task tools：`TaskCreate`、`TaskUpdate`、`TaskGet`、`TaskList`
- 可以设置 `CLAUDE_CODE_ENABLE_TASKS=0` 回退到 `TodoWrite`

迁移关系可以这样理解：

| TodoWrite V1 | Task tools |
|--------------|------------|
| 一个工具调用重写整个 `todos` 数组 | `TaskCreate` 创建单个任务，`TaskUpdate` 修改单个任务 |
| 每个 item 没有稳定 task ID | 每个任务有 ID |
| 通常是内存中的 flat list | 可支持更结构化的状态 |
| 适合简单进度展示 | 更适合复杂任务和多轮状态管理 |

官方文档也说明，Task tools 中 `TaskList` / `TaskGet` 可让模型读回当前任务状态；这比「只在内存里替换一个 todos 数组」更接近真实任务系统。

### Hooks 可以围绕任务系统做集成

虽然 Todo / Task 不是 hook，但 hooks 可以围绕它做监督、集成和副作用：

| Hook 用法 | 例子 |
|-----------|------|
| `TaskCreated` | 新任务创建后同步到 Jira / GitHub Issue |
| `TaskCompleted` | 任务完成后发通知或写审计日志 |
| `PreToolUse` | 复杂任务未建 task 时，阻止普通工具执行 |
| `PostToolUse` | 工具执行后提醒模型更新任务状态 |

所以正确关系是：

> Task tools 是任务系统本体；Hooks 是围绕任务系统和工具生命周期的扩展点。

---

## Q4：内置工具是否比 hook 更核心？

### 工具是 agent 行动能力，hook 是生命周期扩展点

可以这么理解：内置工具是框架 / runtime 的一等能力，通常比普通 hook 更核心、更稳定，也更受系统约束。

但「优先级更高」要拆开看：

| 维度 | 内置工具 | Hook |
|------|----------|------|
| 定位 | agent 能主动调用的能力 | runtime 在事件点自动触发的扩展逻辑 |
| 谁触发 | 模型根据任务选择调用 | 框架在固定生命周期点调用 |
| 是否出现在工具列表 | 是，模型能看到 schema | 通常不作为模型工具暴露 |
| 适合做什么 | 读写文件、执行命令、维护任务、创建子任务 | 日志、审计、拦截、同步外部系统、注入上下文 |
| 对 agent 行为的影响 | 直接影响 agent 能做什么 | 影响事件发生时如何处理 |

所以 `TodoWrite` / `TaskCreate` 这类规划能力如果是内置工具，就说明它不是外挂小功能，而是框架希望模型显式使用的一种核心工作方式。

### TaskCreate 与 TaskCreated 的区别

这个对比最容易说明工具和 hook 的边界：

```text
TaskCreate 工具：
  模型主动调用，用来创建一个任务。

TaskCreated hook event：
  任务已经创建后，runtime 通知外部系统这个事件发生了。
```

类比 Spring：

```text
内置工具 ≈ 框架提供的一等 API / service
Hooks ≈ AOP advice / interceptor / listener
```

所以它们不是互相替代，而是职责不同：

```text
工具 = agent 行动能力
hook = 行动过程中的扩展点
```

---

## Q5：一个好 plan 是模型能力还是 harness 能力？

### 计划质量主要来自模型

`todo_write` 不会自动帮模型想出好计划。它只规定输出结构：

```json
{
  "content": "Run focused tests",
  "status": "pending"
}
```

至于模型会不会把任务拆成：

```text
1. Inspect current behavior
2. Locate relevant files
3. Make minimal change
4. Run focused tests
5. Summarize result
```

还是拆成：

```text
1. Do task
2. Finish task
```

主要取决于模型能力、系统提示、工具描述、上下文示例和任务本身是否清楚。

### 可靠执行主要来自 harness

如果说「写出好计划」主要是模型能力，那么「让计划持续存在、持续可见、持续被更新」就是 harness / runtime 能力。

| 问题 | 主要靠谁 |
|------|----------|
| 计划拆得好不好 | 模型能力 |
| 是否先写计划 | system prompt + harness |
| 计划格式是否结构化 | tool schema / harness |
| 是否持续更新计划 | reminder / runtime 约束 |
| 是否能从计划回到下一步 | 模型能力 + 当前上下文可见性 |
| 是否防止计划丢失 | harness 注入、持久化、任务工具 |

因此 TodoWrite 的价值不是让模型突然变聪明，而是让模型的计划变成可观测、可提醒、可恢复的外部状态。

### 从 todo list 到 workflow engine

如果想让计划质量更稳定，harness 可以继续加规则，例如：

| 规则 | 目的 |
|------|------|
| 多步骤任务至少 3 个 todo | 避免过粗计划 |
| 必须包含验证 / 测试步骤 | 防止做到一半就宣称完成 |
| 同一时间只能有一个 `in_progress` | 保持焦点 |
| 执行工具前必须有当前 `in_progress` | 每个动作绑定任务 |
| 所有任务 completed 前必须跑验证 | 提升完成可信度 |
| 计划太空泛时拒绝 `todo_write` | 提升计划质量 |

但规则越强，系统就越从「todo 工具」走向「workflow engine」。这会提高可靠性，也会降低灵活性。

更合理的心智模型是：

```text
模型负责生成计划；
harness 负责结构化、保存、展示、提醒、必要时约束计划。
```

---

## 参考链接

| 主题 | 路径 / 链接 |
|------|-------------|
| 教学版 TodoWrite 说明 | [`s05_todo_write/README.md`](../../s05_todo_write/README.md) |
| 教学版 TodoWrite 代码 | [`s05_todo_write/code.py`](../../s05_todo_write/code.py) |
| Hooks 问答 | [`hooks-qa.md`](hooks-qa.md) |
| Permission 问答 | [`permission-qa.md`](permission-qa.md) |
| Validation 问答 | [`validation-qa.md`](validation-qa.md) |
| Claude Code Todo Lists 官方文档 | <https://code.claude.com/docs/en/agent-sdk/todo-tracking> |
| Claude Code Python SDK 工具参考 | <https://code.claude.com/docs/en/agent-sdk/python> |
| Claude Code Hooks 官方参考 | <https://docs.anthropic.com/en/docs/claude-code/hooks> |

---

*文档版本：2026-06-28*
