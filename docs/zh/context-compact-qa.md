# Context Compact 问答

> 本文档整理自学习 `s08_context_compact` 过程中的 Context Compact 相关问题，基于本仓库 `s08_context_compact` 章节和前置的 agent loop / tool result / error recovery 机制。与 [Error Recovery 问答](error-recovery-qa.md) 互补：Error Recovery 管「调用失败后怎么恢复」，Context Compact 管「调用前怎么让上下文继续装得下」。
>
> **创建日期**：2026-06-28 · **最后更新**：2026-06-28

---

## 目录

- [Q1：Context Compact 是不是就是上下文满了以后总结一下，然后丢弃旧上下文新开对话？](#q1context-compact-是不是就是上下文满了以后总结一下然后丢弃旧上下文新开对话)
- [Q2：压缩是有损的，多次 compact 后是不是一定会出现目标漂移？](#q2压缩是有损的多次-compact-后是不是一定会出现目标漂移)
- [Q3：成熟 agent 是否应该把内容持久化到硬盘，尤其是中间步骤和大输出？](#q3成熟-agent-是否应该把内容持久化到硬盘尤其是中间步骤和大输出)
- [参考链接](#参考链接)

---

## Q1：Context Compact 是不是就是上下文满了以后总结一下，然后丢弃旧上下文新开对话？

这个理解抓住了最后一层，但不完整。

更准确地说：

> Context Compact 不是「满了才总结」，而是「每次 LLM call 前都管理上下文预算；先做便宜压缩，实在不够才用 LLM 总结」。

`s08` 的核心是四层压缩管线：

```text
每轮 LLM call 前
  -> tool_result_budget：大工具输出落盘，只留预览
  -> snip_compact：消息太多时裁掉中间旧对话
  -> micro_compact：旧 tool_result 替换成占位符
  -> 如果仍超阈值：compact_history，用 LLM 总结
  -> 如果 API 仍报 prompt_too_long：reactive_compact 应急压缩
```

代码里的执行顺序是：

```python
messages[:] = tool_result_budget(messages)
messages[:] = snip_compact(messages)
messages[:] = micro_compact(messages)

if estimate_size(messages) > CONTEXT_LIMIT:
    messages[:] = compact_history(messages)
```

所以它不是直接「总结旧对话」，而是先尝试三种 0 API cost 的结构性压缩：

| 层 | 做什么 | 为什么便宜 |
|------|------|------|
| `tool_result_budget` | 把大工具输出写到 `.task_outputs/tool-results/`，上下文里只留路径和 preview | 不调用 LLM |
| `snip_compact` | 消息太多时保留头尾，裁掉中间 | 不调用 LLM |
| `micro_compact` | 旧 `tool_result` 替换成占位符 | 不调用 LLM |
| `compact_history` | 让 LLM 总结历史，替换 `messages[]` | 需要 1 次 LLM call，有损且更贵 |

你说的「总结下然后丢弃之前上下文」对应的是最后一层 `compact_history()`。教学版确实会把旧 `messages[]` 替换成：

```text
[Compacted]

summary...
```

但这不是严格意义上的「新开对话」。更准确是：

```text
同一个 session
  -> harness 改写 messages[]
  -> 旧细节退出 active context
  -> summary 作为新的上下文继续
```

旧内容也不是完全销毁。教学版在总结前会先写 transcript：

```python
transcript_path = write_transcript(messages)
summary = summarize_history(messages)
return [{"role": "user", "content": f"[Compacted]\n\n{summary}"}]
```

所以：

```text
完整旧历史 -> .transcripts/
active context -> 只保留 summary
```

教学版没有提供 transcript retrieval tool，因此模型当前不能自动回查旧 transcript；但文件已经持久化在硬盘上。

这个章节真正重要的点是：

```text
Context Compact = agent 的上下文垃圾回收机制
```

它不是优雅地总结聊天记录，而是让 agent 在大项目里继续跑下去。

---

## Q2：压缩是有损的，多次 compact 后是不是一定会出现目标漂移？

是的，这个判断是对的。

> Context compact 本质上是有损压缩；多次 compact 一定有 drift 风险。

原始上下文可能包含：

```text
100 条消息
30 个 tool_result
10 个文件内容
若干用户约束
若干中间判断
若干「为什么不做 X」的理由
```

compact 后可能变成：

```text
一条 1000-3000 token 的 summary
```

这一定会丢信息，尤其容易丢：

```text
细小约束
用户语气中的偏好
曾经排除过的方案
某个工具输出里的边角线索
中间判断的依据
「为什么不做 X」的原因
```

多次 compact 后，可能出现：

```text
原始目标
  -> summary v1
  -> summary v2
  -> summary v3
  -> 越来越抽象
  -> 目标漂移 / 约束丢失 / 重复踩坑
```

但不 compact 的结果也很糟：

```text
不压缩：
  信息完整，但 context 爆掉，API 返回 prompt_too_long，任务中断

压缩：
  信息有损，但任务可以继续
```

所以它是一个 tradeoff，不是无损魔法。

降低 drift 的常见办法：

| 机制 | 作用 |
|------|------|
| 结构化 summary prompt | 强制保留当前目标、用户约束、已完成工作、已读/已改文件、剩余任务 |
| 外部化状态 | 把 todo、task、git diff、测试结果、报告文件放在 context 外 |
| compact 后恢复关键上下文 | 真实 CC 会重新附加部分最近文件、plan、agent/skill/tool context |
| 允许重新读取 | 旧 `tool_result` 被压缩后，需要时再 `read_file` 或重跑命令 |
| transcript 持久化 | 完整历史可追溯，尽管不一定在 active context |
| 熔断 / 限制 compact 次数 | 防止无意义地反复总结和漂移 |

所以可以更精确地说：

> 多次 context compact 会累积信息损失，因此长任务必须把关键状态外部化；compact 只能保留足够继续工作的摘要，不能替代任务系统、文件系统、memory、transcript 和验证机制。

这也是为什么后面会继续讲 Memory、Task System、Background Tasks、Agent Teams。它们某种程度上都在回答同一个问题：

```text
如果 active context 注定会丢，什么信息必须放到 context 外面？
```

---

## Q3：成熟 agent 是否应该把内容持久化到硬盘，尤其是中间步骤和大输出？

是的。成熟 agent 一定不能只靠 active context。

更准确地说，要区分两类存储：

```text
1. active context
   -> 给模型下一次推理直接看的内容

2. external storage
   -> 给恢复、追溯、重读、审计用的外部状态
```

Context compact 的本质就是把很多内容从第 1 类迁移到第 2 类：

```text
昂贵的模型上下文
  -> 便宜的硬盘 / artifact / task DB
  -> active context 只保留继续工作最需要的摘要和索引
```

`s08` 教学版已经有两个持久化例子。

### 1. transcript 持久化

`compact_history()` 在总结前先写完整 transcript：

```python
transcript_path = write_transcript(messages)
summary = summarize_history(messages)
return [{"role": "user", "content": f"[Compacted]\n\n{summary}"}]
```

含义是：

```text
完整旧对话 -> .transcripts/
active context -> [Compacted] summary
```

这样旧历史退出 active context，但仍有恢复和审计记录。

### 2. 大 tool_result 持久化

`tool_result_budget()` 会把很大的工具输出写到：

```text
.task_outputs/tool-results/
```

上下文里只留：

```text
<persisted-output>
Full output: path/to/file
Preview:
前 2000 字符
</persisted-output>
```

这样可以避免一个大文件、一次大命令输出直接撑爆 context。模型需要完整内容时，可以通过路径再读。

### 信息应该如何分层

成熟 agent 会把不同信息放到不同位置：

| 信息类型 | 放哪里 |
|------|------|
| 当前目标、关键约束、下一步 | active context / compact summary |
| 完整历史对话 | transcript 文件 |
| 大工具输出 | `.task_outputs/` 或 artifact store |
| 当前任务状态 | todo / task DB |
| 用户长期偏好 | memory |
| 项目规则 | `CLAUDE.md` / rules |
| 代码变更 | 文件系统 / git diff |
| 测试结果 | 日志 / artifact |
| subagent 调查结果 | summary + report file |

所以你说「中间步骤也持久化下来节约 context」是对的，但还要加一个限制：

> 不是所有中间步骤都应该放进 active context；但很多中间步骤值得持久化，方便需要时回查。

例如 subagent 调查 bug，不应该把 50 条工具结果全部塞回 main session，但可以让它写：

```text
reports/parser-investigation.md
```

main agent 当前只看 summary。后续需要细节，再读取报告。

一句话总结：

> Compact 不是删除信息，而是把信息从「昂贵的模型上下文」迁移到「便宜的外部存储」，然后只把继续工作最需要的摘要、索引和当前状态留在上下文里。

---

## 参考链接

| 主题 | 路径 |
|------|------|
| 教学版 Context Compact 说明 | [`s08_context_compact/README.md`](../../s08_context_compact/README.md) |
| 教学版 Context Compact 代码 | [`s08_context_compact/code.py`](../../s08_context_compact/code.py) |
| Error Recovery 问答 | [`error-recovery-qa.md`](error-recovery-qa.md) |
| System Prompt 问答 | [`system-prompt-qa.md`](system-prompt-qa.md) |
| Memory 章节 | [`s09_memory/README.md`](../../s09_memory/README.md) |
| Task System 章节 | [`s12_task_system/README.md`](../../s12_task_system/README.md) |

---

*文档版本：2026-06-28*
