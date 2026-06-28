# Memory 问答

> 本文档整理自学习 `s09_memory` 过程中的 Memory 相关问题。它关注的不是“怎么把聊天记录全部保存下来”，而是 agent loop 如何把少量长期有用的信息持久化，并在未来会话中重新带回上下文。
>
> **创建日期**：2026-06-28 · **最后更新**：2026-06-28

---

## 目录

- [Q1：Memory 这一章的重点是什么？](#q1memory-这一章的重点是什么)
- [Q2：Memory 和 Context Compact、transcript、todo/task 有什么区别？](#q2memory-和-context-compacttranscripttodotask-有什么区别)
- [Q3：Codex 用 SQLite，Claude Code 是不是用了不一样的方法？](#q3codex-用-sqliteclaude-code-是不是用了不一样的方法)
- [Q4：怎么知道 session 中哪些值得记录下来，哪些不值得？](#q4怎么知道-session-中哪些值得记录下来哪些不值得)
- [Q5：是不是还是通过 LLM 来判断哪些值得写入 memory？](#q5是不是还是通过-llm-来判断哪些值得写入-memory)
- [Q6：Memory 在 agent loop 里处在哪些环节？](#q6memory-在-agent-loop-里处在哪些环节)
- [Q7：为什么 memory 要有 index？为什么不把所有 memory 都塞进上下文？](#q7为什么-memory-要有-index为什么不把所有-memory-都塞进上下文)
- [Q8：Memory 写错了怎么办？](#q8memory-写错了怎么办)
- [参考链接](#参考链接)

---

## Q1：Memory 这一章的重点是什么？

Memory 这一章最重要的点是：

> LLM 本身是无状态的，但用户希望 agent 像一个持续工作的助手。Memory 是 harness 给无状态 LLM 补上的长期连续性层。

它解决的问题不是“当前任务怎么继续”，而是：

```text
下次新开一个 session 时，
agent 仍然应该知道哪些用户偏好、项目事实、工作方式和查找线索？
```

`s09_memory` 给 Memory 分了四类：

| 类型 | 解决什么问题 | 例子 |
|---|---|---|
| `user` | 用户是谁 / 用户偏好 | 用户喜欢中文解释，喜欢先看代码再下结论 |
| `feedback` | 应该怎么工作 | 不要 mock 数据库，先复现真实错误 |
| `project` | 项目长期事实 | 这个 repo 的中文学习笔记放在 `docs/zh` |
| `reference` | 东西在哪找 | hooks 相关问题看 `hooks-qa.md` |

所以 Memory 的核心不是“多存”，而是“少量、稳定、未来可复用”。

一个成熟的 memory 系统通常包括四件事：

| 子系统 | 作用 |
|---|---|
| storage | 把 memory 持久化到文件、SQLite、数据库或远端服务 |
| index | 用很小的目录描述已有 memory |
| retrieval | 当前问题相关时，按需加载少量 memory 内容 |
| extraction | 会话结束或用户明确要求时，抽取新的长期信息 |

可以用一句话记：

> **Memory 不是保存过去，而是帮助未来。**

---

## Q2：Memory 和 Context Compact、transcript、todo/task 有什么区别？

这几个机制很容易混在一起，但它们解决的问题不一样。

| 机制 | 保存什么 | 主要作用 | 生命周期 |
|---|---|---|---|
| Memory | 长期偏好、约定、项目事实、查找线索 | 跨 session 复用 | 长期 |
| Context Compact | 当前 session 的压缩摘要 | context 快满时继续当前任务 | 当前 session |
| Transcript | 完整历史记录 | 审计、回放、人工查证 | 长期或按策略保留 |
| todo/task | 当前任务计划和执行状态 | 让 agent 不跑偏 | 当前任务 |

例如：

```text
“下一步先写 memory-qa.md”
```

这应该进 `todo_write` 或 task system，因为它是当前任务计划。

```text
“以后每章学习问题都整理成 docs/zh/*-qa.md”
```

这可以进 Memory，因为它是长期工作偏好。

```text
“刚才工具读到了 500 行日志”
```

这通常不进 Memory。它是当前 session 的临时上下文，应该由 transcript、compact summary 或工具输出文件管理。

```text
“这个项目里 hooks 相关问答在 docs/zh/hooks-qa.md”
```

这可以进 Memory 或 reference，因为它未来能帮 agent 更快找到信息。

所以可以这样分工：

```text
Memory:
  以后也应该知道什么

Context Compact:
  这次任务怎么继续

Transcript:
  过去完整发生了什么

Todo / Task:
  接下来要做什么
```

---

## Q3：Codex 用 SQLite，Claude Code 是不是用了不一样的方法？

是的，至少从公开文档和本地观察看，Codex 和 Claude Code 的持久化形态不一样。

Codex 本地 app 里可以看到类似这些 SQLite 文件：

```text
memories_1.sqlite
state_5.sqlite
goals_1.sqlite
logs_2.sqlite
```

这说明 Codex 会把一部分本地状态、日志、目标、memory 等结构化信息放进 SQLite。

但这不等于：

```text
LLM 直接查询 SQLite
```

更准确的是：

```text
harness 读 SQLite / 文件系统 / 其他持久层
  -> 选出当前相关内容
  -> 注入到 prompt/messages
  -> LLM 基于注入后的上下文推理
```

Claude Code 的公开 memory 机制更偏文件系统：

```text
~/.claude/projects/<project>/memory/
  MEMORY.md
  debugging.md
  api-conventions.md
  ...
```

也就是 Markdown 文件 + YAML frontmatter + `MEMORY.md` index。

两种方式不是本质矛盾，而是存储介质不同：

| 系统 | 存储形态 | 更适合 |
|---|---|---|
| Codex | SQLite + 本地目录 | app 状态、日志、目标、结构化记录 |
| Claude Code | Markdown + `MEMORY.md` | 用户可读、项目可迁移、可手动编辑的长期知识 |

关键点是：

> Memory 的本质不是 SQLite 还是 Markdown，而是 harness 如何选择、加载、注入和更新这些长期信息。

---

## Q4：怎么知道 session 中哪些值得记录下来，哪些不值得？

判断标准可以压缩成一个问题：

> 下次新开 session 时，如果 agent 不知道这件事，会不会明显做错、重复劳动，或者违背用户偏好？

如果答案是“会”，它可能值得进 Memory。

如果答案只是“刚才发生了什么”，通常不应该进长期 Memory。

### 值得记录

| 内容 | 原因 |
|---|---|
| 稳定用户偏好 | 以后每次交互都可能影响回答方式 |
| 重复出现的工作反馈 | 能避免同类错误反复发生 |
| 项目长期事实 | 新 session 也需要知道项目结构和约定 |
| 常用入口 / 查找线索 | 能节约未来搜索成本 |
| 重要约束 | 能避免违规实现或错误决策 |

例子：

```text
用户希望中文对话，但 repo-facing 文档保持英文。
```

```text
用户在这个 repo 里喜欢把每章问答整理到 docs/zh/*-qa.md。
```

```text
调试任务要从真实错误和文件存在性检查开始。
```

### 不值得记录

| 内容 | 原因 |
|---|---|
| 一次性的当前计划 | 应该进 todo/task |
| 大段工具输出 | 太长，且未来不一定相关 |
| 已经写入文件的完整内容 | 文件本身就是事实来源 |
| 模型自己的未经验证推测 | 容易污染后续 session |
| 普通聊天流水账 | 应该由 transcript/history 管 |
| secret、token、隐私数据 | 不应该长期持久化 |

例子：

```text
“下一步我们先读 README。”
```

这是当前任务计划，不是长期记忆。

```text
“刚才 grep 出来 20 个结果。”
```

这是临时工具上下文，不是长期记忆。

```text
“我猜这个 bug 可能来自缓存。”
```

如果没有验证，不应该直接写成 Memory。

---

## Q5：是不是还是通过 LLM 来判断哪些值得写入 memory？

是的，语义判断通常还是靠 LLM，但不能理解成“完全交给 LLM 自由发挥”。

更准确地说：

```text
harness 决定什么时候检查、检查什么范围、允许写什么类型
LLM 判断哪些内容语义上值得长期记住
harness 再做校验、去重、落盘、合并
```

也就是：

| 角色 | 负责什么 |
|---|---|
| LLM | 判断语义价值：偏好、约束、项目事实、查找线索 |
| harness | 控制触发时机、上下文范围、schema、去重、安全、写入位置 |

教学代码里的 `extract_memories()` 就是这个思路：

```python
prompt = (
    "Extract user preferences, constraints, or project facts from this dialogue.\n"
    "Return a JSON array. Each item: {name, type, description, body}.\n"
    "If nothing new or already covered by existing memories, return [].\n\n"
    f"Existing memories:\n{existing_desc}\n\n"
    f"Dialogue:\n{dialogue[:4000]}"
)
```

这里有几个重要约束：

| 约束 | 作用 |
|---|---|
| 只看最近对话 | 避免把整个 session 都塞给抽取器 |
| 给出已有 memory | 避免重复写入 |
| 要求 JSON array | 让 harness 可解析 |
| 限定 memory type | 避免无限发散 |
| 允许返回 `[]` | 鼓励保守写入 |

所以最好的理解是：

> **LLM 是 Memory 的语义筛选器，harness 是 Memory 的制度和边界。**

写入 Memory 应该偏保守，因为写错一次，后面很多 session 都可能被污染。

---

## Q6：Memory 在 agent loop 里处在哪些环节？

Memory 通常出现在 agent loop 的前后两端：

```text
用户输入
  ↓
读取 memory index
  ↓
选择 relevant memories
  ↓
组装 system prompt + messages
  ↓
LLM call
  ↓
tool use / assistant response
  ↓
如果 turn 结束：extract memories
  ↓
必要时 consolidate / dream
```

也就是说，Memory 不是 LLM 内置功能，而是 harness 在调用 LLM 之前和之后做的工作。

### 读取阶段

读取阶段通常发生在 user turn 开始时。

教学代码里有两个层次：

```text
MEMORY.md index
  -> 放进 system prompt，让模型知道有哪些 memory

relevant memory content
  -> 根据当前对话按需加载少量具体 memory
```

这对应：

```python
system = build_system()
memories_content = load_memories(messages)
```

### 写入阶段

写入阶段通常发生在一个 turn 自然结束后，而不是 tool loop 中间。

教学代码里是：

```python
if response.stop_reason != "tool_use":
    extract_memories(pre_compress)
    consolidate_memories()
```

这背后的含义是：

```text
还在 tool_use 中：
  当前任务没有结束，不急着写长期 memory

没有 tool_use 了：
  对话来到一个自然边界，可以尝试抽取长期信息
```

真实系统可能更复杂，例如 fork 一个受限 subagent 做 memory extraction、跳过 transcript、限制 turns、加锁、防止和主 agent 同时写入。但基本思想仍然是：

> 主 loop 负责解决任务；memory extraction 是旁路的长期知识整理。

---

## Q7：为什么 memory 要有 index？为什么不把所有 memory 都塞进上下文？

因为 memory 会越来越多，而上下文窗口和 prompt cache 都很珍贵。

如果每次都把所有 memory 内容塞进去，会有几个问题：

| 问题 | 后果 |
|---|---|
| 上下文膨胀 | 留给当前任务的空间变少 |
| 相关性下降 | 无关 memory 干扰推理 |
| prompt cache 变差 | 动态内容太多，前缀不稳定 |
| 成本上升 | 每轮都重复发送大量旧内容 |
| 污染风险增加 | 不相关旧偏好影响当前任务 |

所以更合理的结构是：

```text
MEMORY.md index:
  小、稳定、适合常驻 system prompt

memory file content:
  大、具体、按需加载
```

可以类比成：

```text
目录常驻
书本按需取
```

教学版里的流程是：

```text
扫描 .memory/*.md
  -> 生成 MEMORY.md
  -> system prompt 中放 index
  -> 当前 turn 通过 name/description 选择最多 5 条相关 memory
  -> 读取这些 memory 的完整内容并注入上下文
```

这和 system prompt caching 也有关。

稳定的 index 可以更容易复用缓存；动态的具体 memory 内容则放在更靠近当前 user turn 的位置，减少对稳定前缀的破坏。

---

## Q8：Memory 写错了怎么办？

Memory 写错不是小问题，因为它会跨 session 影响 agent。

所以成熟系统需要有维护机制：

| 能力 | 作用 |
|---|---|
| 查看 memory | 用户能知道 agent 到底记住了什么 |
| 编辑 memory | 修正错误偏好或项目事实 |
| 删除 memory | 移除过期或错误信息 |
| 去重合并 | 防止同类 memory 越积越多 |
| 过期策略 | 避免旧项目事实长期污染 |
| 敏感信息过滤 | 防止 secret、token、隐私内容入库 |

教学版里有 `consolidate_memories()`，当 memory 文件变多时，把它们交给 LLM 做去重合并。

真实系统里这个过程可能叫 Dream 或 consolidation。它通常不是每轮都跑，而是低频触发，例如满足时间间隔、文件数量、session 数、锁状态等条件后再执行。

这说明 Memory 不是“写进去就永远正确”。

更合理的理解是：

> Memory 是长期上下文候选，需要能被审查、修正、合并和删除。

---

## 一个快速判断公式

当你不确定一句话该不该进 Memory，可以按下面顺序判断：

```text
1. 这是不是未来 session 仍然有用？
2. 这是不是稳定事实，而不是临时步骤？
3. 这是不是用户偏好、工作反馈、项目事实或查找线索？
4. 如果不记住，agent 以后会不会重复犯错或重复劳动？
5. 它是否安全，不包含 secret、token、隐私信息？
6. 已有 memory 是否已经覆盖它？
```

只有大部分答案都偏“是”，才值得写入 Memory。

反过来，如果它只是：

```text
现在要做什么
刚才发生什么
某个工具输出了什么
模型临时猜了什么
```

就不要写入长期 Memory。

---

## 参考链接

| 主题 | 位置 |
|---|---|
| Memory 章节 | [`s09_memory/README.md`](../../s09_memory/README.md) |
| Memory 教学代码 | [`s09_memory/code.py`](../../s09_memory/code.py) |
| Context Compact 问答 | [`context-compact-qa.md`](context-compact-qa.md) |
| System Prompt 问答 | [`system-prompt-qa.md`](system-prompt-qa.md) |
| Todo Write 问答 | [`todo-write-qa.md`](todo-write-qa.md) |
| Claude Code 官方 Memory 文档 | <https://code.claude.com/docs/en/memory> |
