# Subagent 问答

> 本文档整理自学习 `s06_subagent` 过程中的 Subagent 相关问题，基于本仓库 `s06_subagent`、后续 `s15_agent_teams` 章节，以及 Claude Code 源码附录说明。与 [TodoWrite 问答](todo-write-qa.md)、[Hooks 问答](hooks-qa.md) 互补：Subagent 管「如何把局部探索隔离到干净上下文里」。
>
> **创建日期**：2026-06-28 · **最后更新**：2026-06-28

---

## 目录

- [Q1：Subagent 和 multi-agent 有什么区别？](#q1subagent-和-multi-agent-有什么区别)
  - [一句话概括](#一句话概括)
  - [Subagent 是一次性 worker，不是团队成员](#subagent-是一次性-worker不是团队成员)
  - [为什么先用 Subagent，而不是直接 multi-agent](#为什么先用-subagent而不是直接-multi-agent)
- [Q2：Subagent 会携带父 agent 的上下文吗？](#q2subagent-会携带父-agent-的上下文吗)
  - [携带任务说明，不携带完整父上下文](#携带任务说明不携带完整父上下文)
  - [Subagent 如何知道自己是 subagent](#subagent-如何知道自己是-subagent)
  - [真实 CC 的边界更细](#真实-cc-的边界更细)
- [Q3：Subagent 完成任务后会被销毁吗？还能追问细节吗？](#q3subagent-完成任务后会被销毁吗还能追问细节吗)
  - [教学版是用完即丢，只回传 summary](#教学版是用完即丢只回传-summary)
  - [后续追问取决于 summary 或外部 artifact](#后续追问取决于-summary-或外部-artifact)
  - [好的 subagent prompt 要求返回证据](#好的-subagent-prompt-要求返回证据)
- [参考链接](#参考链接)

---

## Q1：Subagent 和 multi-agent 有什么区别？

### 一句话概括

**Subagent 是上下文隔离工具；multi-agent 是协作系统。**

Subagent 更像主 agent 调用一个临时 worker / 子过程，让它用干净上下文完成一个局部任务，然后只把结论返回。Multi-agent / Agent team 则是多个长期 agent 之间通过 inbox、消息协议、任务归属、权限冒泡等机制持续协作。

### Subagent 是一次性 worker，不是团队成员

在 `s06_subagent` 中，父 agent 通过 `task` 工具调用 `spawn_subagent()`。Subagent 会创建新的 `messages[]`，跑自己的 loop，最后只返回最终文本：

```python
messages = [{"role": "user", "content": description}]  # fresh context
...
return result  # only summary, entire message history discarded
```

可以这样对比：

| 维度 | Subagent | Multi-agent / Agent team |
|------|----------|--------------------------|
| 生命周期 | 一次性，用完即丢 | 长生命周期，可持续协作 |
| 关系 | Parent → child，像函数调用 | 多个 agent 互相通信，像团队 |
| 上下文 | 子 agent 有独立 `messages[]` | 每个 agent 有自己的上下文 / inbox |
| 通信方式 | 只返回最终 summary | 可多轮互发消息 |
| 主要目的 | 隔离上下文、压缩探索过程 | 并行协作、分工、长期任务 |
| 复杂度 | 低 | 高，接近分布式系统 |
| 课程位置 | `s06_subagent` | `s15_agent_teams` 及之后 |

### 为什么先用 Subagent，而不是直接 multi-agent

`s06` 要解决的核心问题不是「人手不够」，而是**上下文污染**。

例如主 agent 要修一个 bug，但定位问题需要读 30 个文件、跑很多 grep、追很长调用链。如果这些中间过程都塞进主 `messages[]`，主 agent 后面会越来越忘记原始目标。

Subagent 的做法是：

```text
主 agent：你去调查 parser 的调用链
  ↓
subagent：自己读文件、搜索、分析
  ↓
subagent：返回一个总结
  ↓
主 agent：只拿总结继续修 bug
```

中间 30 个工具结果被丢掉，只留下结论。这就是它的价值。

Multi-agent 解决的是另一类问题：多个长期工作流需要并行推进和互相通信。这会引入：

- inbox / message bus
- 消息协议
- 任务归属
- 权限冒泡
- 冲突处理
- shutdown 协议

这些能力很强，但协调成本也高。能用 Subagent 解决时，不要先上 multi-agent。

---

## Q2：Subagent 会携带父 agent 的上下文吗？

### 携带任务说明，不携带完整父上下文

需要区分两类上下文：

| 上下文类型 | 教学版 subagent 是否携带 |
|------------|--------------------------|
| 父 agent 完整聊天历史 / tool results | 不携带 |
| 父 agent 写给子 agent 的任务说明 | 携带 |
| 子 agent 自己的 system prompt | 携带 |
| 工具列表 | 携带，但受限 |
| 工作目录 / 文件系统状态 | 共享 |

教学版 `s06` 中，subagent 的第一条用户消息就是父 agent 传入的 `description`：

```python
messages = [{"role": "user", "content": description}]
```

所以它不是空上下文。它知道自己要做什么，是因为父 agent 调用 `task` 工具时会写清楚任务，例如：

```text
Explore the CLI parser and summarize the control flow.
```

但它不会自动拿到父 agent 的全部历史、全部中间推理、全部工具结果。这正是 Subagent 的设计目的：**带足够信息完成局部任务，但不带走父上下文噪声**。

### Subagent 如何知道自己是 subagent

它有独立的 `SUB_SYSTEM`：

```python
SUB_SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Complete the task you were given, then return a concise summary. "
    "Do not delegate further."
)
```

这相当于告诉它：

```text
你是一个 coding agent
你只完成给定任务
完成后返回简洁总结
不要继续委派
```

同时，教学版 subagent 的工具也受限制：它有 `bash`、`read_file`、`write_file`、`edit_file`、`glob`，但没有 `task` 工具，避免递归创建 sub-subagent。

### 真实 CC 的边界更细

真实 Claude Code 比教学版复杂。`s06_subagent` 附录里提到至少三种模式：

| 模式 | 特点 |
|------|------|
| Normal Subagent | 真正 fresh messages，只带任务 prompt |
| Fork Subagent | 构造 cache-friendly message prefix，利于 prompt cache |
| General-Purpose | 无指定类型时的通用路径 |

真实实现里有些状态可能会共享或克隆，例如文件读取状态、权限冒泡、abort controller 等。但核心目标仍然一样：

> 不是复制父 agent 的整个脑子，而是给子 agent 一个边界清楚的子任务上下文。

---

## Q3：Subagent 完成任务后会被销毁吗？还能追问细节吗？

### 教学版是用完即丢，只回传 summary

在教学版 `s06` 中，subagent 完成后会被销毁。更准确地说，`spawn_subagent()` 里的 `messages[]` 是局部变量，函数返回后父 agent 只拿到 `result`：

```python
return result  # only summary, entire message history discarded
```

这意味着：

```text
subagent 的完整 messages[]：丢弃
subagent 的中间 tool results：丢弃
subagent 的最终 summary：返回给父 agent
文件系统副作用：保留
```

这也是 context isolation 的核心取舍：

| 优点 | 缺点 |
|------|------|
| 主上下文干净，只保留结论 | 中间过程默认不可追问 |
| 大量探索不污染主 session | 如果 summary 太短，后续缺少证据 |
| 父 agent 继续聚焦原任务 | 需要重新调查或读取外部 artifact |

### 后续追问取决于 summary 或外部 artifact

如果你后续在 main session 里问：

```text
刚才 subagent 为什么这么判断？
它具体看了哪些文件？
```

教学版 main agent 默认只能根据 summary 回答。如果 summary 没写，它就不知道。

除非发生以下情况之一：

1. subagent 的 summary 主动包含文件、函数、证据和判断过程
2. subagent 对文件系统产生了可见副作用，例如写了报告文件
3. runtime 额外保存了 subagent transcript
4. main agent 重新派一个 subagent 或自己重新查

因此，subagent 不是知识永久缓存。它更像一次性调查任务：调查过程被压缩成结论，结论质量决定后续可追问性。

### 好的 subagent prompt 要求返回证据

为了后续可追问，父 agent 不应该只写：

```text
Find the bug.
```

更好的 prompt 是：

```text
Trace the parser bug. Return:
1. files inspected
2. relevant functions
3. suspected cause
4. evidence
5. recommended fix
```

这样 subagent 即使用完即丢，父 agent 也拿到了足够结构化的结果。

可以把 subagent 的返回设计成「结论 + 证据索引」：

| 返回字段 | 作用 |
|----------|------|
| Summary | 给父 agent 快速使用的结论 |
| Files inspected | 后续追问时知道查过哪里 |
| Key evidence | 防止结论变成无源判断 |
| Unknowns / caveats | 告诉父 agent 哪些还没确认 |
| Recommended next step | 让父 agent 接续执行 |

真实 Claude Code 可能通过 transcript、forked context、后台 agent、team inbox 等机制保留更多信息。但从设计原则看，subagent 默认应该返回压缩后的结果，而不是把完整过程塞回主上下文；否则就失去了上下文隔离的意义。

---

## 参考链接

| 主题 | 路径 |
|------|------|
| 教学版 Subagent 说明 | [`s06_subagent/README.md`](../../s06_subagent/README.md) |
| 教学版 Subagent 代码 | [`s06_subagent/code.py`](../../s06_subagent/code.py) |
| Agent Teams 对比 | [`s15_agent_teams/README.md`](../../s15_agent_teams/README.md) |
| TodoWrite 问答 | [`todo-write-qa.md`](todo-write-qa.md) |
| Hooks 问答 | [`hooks-qa.md`](hooks-qa.md) |

**CC 源码参考**（附录中引用，本仓库不含 CC 源码）：

- `AgentTool.tsx` — Agent / subagent 工具入口
- `runAgent.ts` — subagent 运行路径
- `forkSubagent.ts` — fork 模式与 prompt cache 相关逻辑
- `forkedAgent.ts` — subagent context 创建与共享边界
- `spawnMultiAgent.ts` — team / teammate 创建
- `teammateMailbox.ts` — multi-agent message types

---

*文档版本：2026-06-28*
