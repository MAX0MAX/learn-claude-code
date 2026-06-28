# Error Recovery 问答

> 本文档整理自学习 `s11_error_recovery` 过程中的错误恢复相关问题，基于本仓库 `s11_error_recovery` 章节和前置的 `s10_system_prompt`。与 [System Prompt 问答](system-prompt-qa.md) 互补：System Prompt 管「调用前如何准备上下文」，Error Recovery 管「调用失败或未自然完成时如何恢复」。
>
> **创建日期**：2026-06-28 · **最后更新**：2026-06-28

---

## 目录

- [Q1：错误恢复到底解决什么问题？处在 loop 的哪个环节？](#q1错误恢复到底解决什么问题处在-loop-的哪个环节)
- [Q2：`max_tokens` 是什么意思？为什么要设置这个值？](#q2max_tokens-是什么意思为什么要设置这个值)
- [Q3：如何判断是否需要给更多输出空间？](#q3如何判断是否需要给更多输出空间)
- [Q4：如何判断应该继续给 token，还是模型在空转、应该终止？](#q4如何判断应该继续给-token还是模型在空转应该终止)
- [Q5：`max_tokens` 应该设置多少？](#q5max_tokens-应该设置多少)
- [Q6：为什么超出 `max_tokens` 后先升级重跑，而不是先生成 8K 再继续下一个 8K？](#q6为什么超出-max_tokens-后先升级重跑而不是先生成-8k-再继续下一个-8k)
- [Q7：重试在 harness 代码里如何实现？是在 catch 里做很多异常分支吗？](#q7重试在-harness-代码里如何实现是在-catch-里做很多异常分支吗)
- [参考链接](#参考链接)

---

## Q1：错误恢复到底解决什么问题？处在 loop 的哪个环节？

一句话：

> Error Recovery 不是新增工具，也不是 hook，而是在 LLM call 外面加一层「错误分类 + 有限重试 + 恢复后回到 loop」的 harness 能力。

`s11` 保留了 `s10` 的 system prompt 组装和普通 agent loop，只改变一件事：**LLM 调用不再裸奔**。

```text
s10:
  response = client.messages.create(...)

s11:
  response = with_retry(lambda: client.messages.create(...), state)
```

错误恢复的位置：

```text
messages / system / tools 准备好
  -> try LLM call
      -> API 抛异常：进入 except 分支
      -> API 成功返回：检查 stop_reason
  -> 根据错误类型恢复
  -> continue 回到 loop 顶部重试
```

教学版主要处理三类问题：

| 类型 | 触发 | 恢复方式 |
|------|------|----------|
| 输出被截断 | `stop_reason == "max_tokens"` | 8K -> 64K 升级；仍不够则 continuation |
| 上下文太长 | `prompt_too_long` / `context_length_exceeded` | reactive compact 后 retry 一次 |
| 临时服务错误 | 429 / 529 | exponential backoff + jitter；连续 529 可切 fallback model |

所以错误恢复的核心不是简单 `try again`，而是：

```text
不同错误类型 -> 不同恢复策略 -> 有限次数 -> 成功则回到 loop，失败则干净退出
```

---

## Q2：`max_tokens` 是什么意思？为什么要设置这个值？

`max_tokens` 是**本次 LLM 调用最多允许模型生成多少输出 token**。

它限制的是输出，不是输入。

```text
输入：
  system prompt + messages + tools + tool results
  -> 占 context window

输出：
  assistant 本次生成的回答 / tool_use
  -> 受 max_tokens 限制
```

例如：

```text
max_tokens = 8000
```

意思是：这次模型最多生成约 8000 个 token。到上限还没写完，API 会停止并返回：

```text
stop_reason = "max_tokens"
```

这不是模型出错，而是输出预算用完了。

为什么要设置 `max_tokens`？

| 作用 | 解释 |
|------|------|
| 控制成本 | 输出 token 计费，限制输出避免一次调用爆掉 |
| 控制延迟 | 生成越多越慢，输出上限让响应时间可控 |
| 防止跑偏 | 模型可能过度解释或输出大量无关内容 |
| 给 agent loop 留空间 | 不让一次回答吃掉全部 token budget |
| 方便恢复 | 触发 `max_tokens` 后，harness 能决定升级、续写或停止 |
| 保护系统资源 | 服务端和客户端都需要可预测的请求规模 |

它和 context window 的区别：

```text
context window = 输入 + 输出的总容量上限
max_tokens = 本次最多生成多少输出
```

---

## Q3：如何判断是否需要给更多输出空间？

判断是否要给更多 `max_tokens`，通常分三层。

### 1. 事前估计

有些任务天然需要长输出：

```text
可能需要更大 max_tokens：
- 生成长文档
- 输出完整代码文件
- 写报告 / 方案 / 大量测试用例
- 总结很多文件
- 返回大型 JSON / Markdown 表格

通常不需要很大 max_tokens：
- 只是下一步 tool_use
- 简短问答
- 改代码任务，主要应该用 edit_file / apply_patch，而不是输出整段代码
```

coding agent 里，很多回合只需要模型输出一个 `tool_use`，默认不应该一开始就给特别大。

### 2. 运行时硬信号

最明确的信号是：

```text
response.stop_reason == "max_tokens"
```

这说明模型不是自然停止，而是输出预算用完了。

`s11` 的处理是：

```text
第一次 max_tokens:
  8000 -> 64000
  不 append 残缺输出
  同一请求重试

64K 还 max_tokens:
  保存 partial output
  注入 continuation prompt
  最多继续 3 次
```

### 3. 运行后完整性检查

即使没有硬错误，也可以看输出结构是否完整：

```text
明显没写完：
- 句子断在一半
- Markdown code fence 没闭合
- JSON 不完整
- 列表说有 5 点但只写了 3 点
- 代码块缺右括号 / 缺结尾
```

实用规则：

```python
if stop_reason == "max_tokens":
    give_more_space_or_continue()
elif output_structure_incomplete:
    continue_from_breakpoint()
elif user_explicitly_requested_long_output:
    consider_larger_max_tokens()
else:
    keep_default_budget()
```

---

## Q4：如何判断应该继续给 token，还是模型在空转、应该终止？

`stop_reason == "max_tokens"` 只说明「输出预算用完了」，不说明「继续一定有价值」。

所以 harness 需要一个 continuation gate。

### 适合继续的信号

```text
结构明显没完成：
- JSON / XML / Markdown 表格没闭合
- code fence 没闭合
- 函数 / class / 括号没写完
- 列表说 1-10，只写到 6
- 句子断在半截

内容仍然贴合任务：
- 还在回答用户问题
- 还在推进 plan 中某一步
- 新增内容能接上前文
- 没有明显重复
```

这时可以注入 continuation prompt：

```text
Continue from exactly where you stopped.
Do not recap.
Do not restart.
```

### 应该终止或改策略的信号

```text
模型开始空转：
- 重复前面已经说过的内容
- 不断 recap / apology / meta explanation
- 同一个观点换说法循环
- 越写越偏离用户任务
- 本该用工具改文件，却一直输出大段代码
- 生成内容没有新增信息
- 连续 continuation 后增量很小
- 多次 continuation 仍然被 max_tokens 截断
```

这时继续给 token 只是烧钱，更好的处理是：

```text
停止续写
总结当前有效部分
拆小任务
改用工具写文件
compact 后继续
或者告诉用户输出过长，需要拆分
```

实用判断框架：

```python
if stop_reason != "max_tokens":
    return "no_continuation_needed"

if continuation_count >= 3:
    return "stop"

if output_has_unclosed_structure(output):
    return "continue"

if user_explicitly_requested_long_output and output_is_on_topic(output):
    return "continue"

if output_is_repetitive(output) or output_is_drifting(output):
    return "stop"

if progress_delta_is_small(output, previous_output):
    return "stop"

return "continue_once_or_summarize"
```

最重要的是看 progress delta：

```text
这次 continuation 是否带来了实质新增信息？
```

如果没有，就应该停止。

---

## Q5：`max_tokens` 应该设置多少？

`max_tokens` 是输出预算，需要在成本、延迟、完整性之间取平衡。

`s11` 教学版设置为：

```python
DEFAULT_MAX_TOKENS = 8000
ESCALATED_MAX_TOKENS = 64000
```

也就是：

```text
默认 8K：
  日常 agent loop 足够，成本和延迟可控

遇到 max_tokens 截断：
  说明这次真的不够
  升级到 64K 重试
```

一般可以按任务类型分层：

| 场景 | 倾向 |
|------|------|
| 普通 tool_use / 改代码循环 | 4K-8K |
| 简短问答 / 分类 / 决策 | 1K-4K |
| 总结多个文件 / 写报告 | 8K-16K 或更高 |
| 生成长文档 / 大段代码 / 长 JSON | 16K-64K |
| 不确定任务 | 默认保守，截断后升级 |

还要受硬限制影响：

```text
max_tokens <= 模型允许的最大输出
input_tokens + max_tokens <= 模型 context window
```

所以不能无脑给最大。输入已经很长时，`max_tokens` 太大可能导致 context overflow。

---

## Q6：为什么超出 `max_tokens` 后先升级重跑，而不是先生成 8K 再继续下一个 8K？

因为第一次 8K 被截断的输出，严格说不是一个完整 assistant turn。

它可能断在：

```text
句子中间
JSON 中间
Markdown code fence 中间
函数中间
tool_use 参数中间
推理结构中间
```

如果把这段残缺输出 append 到 messages，再让模型继续：

```text
assistant: 前 8K 残缺文本
user: 请继续
```

会带来几个问题：

| 问题 | 解释 |
|------|------|
| 污染上下文 | 残缺 assistant turn 进入历史，后续模型只能靠文本猜怎么接 |
| 容易重复 | 模型下一轮可能 recap、重启小节、接不上结构 |
| 成本更高 | 下一轮要把前 8K 当输入重新发回去 |
| 结构化输出危险 | 半截 JSON / 代码 / tool call 会让后续更难恢复 |

所以 `s11` 第一次截断时选择：

```text
不保存残缺输出
同一个请求
更大的 max_tokens
重新生成完整版本
```

是的，重跑确实相当于重新算一遍。64K 不是因为更便宜，而是因为更干净：

```text
8K + 继续：
  残缺 turn 进入 messages
  模型根据文本接续
  容易漂移、重复、结构坏掉

64K 重跑：
  messages 不被污染
  同一上下文，只是输出预算变大
  更可能一次性得到完整答案
```

什么时候才用 8K + continuation？

```text
8K 截断
  -> 升级 64K，重新生成

64K 还截断
  -> 说明内容真的太长
  -> 保存 partial output
  -> continuation prompt
  -> 最多继续几次
```

一句话：

> 第一次截断通常是「纸太小」，先换大纸重写；大纸还不够，才把已写内容留下来接着写。

---

## Q7：重试在 harness 代码里如何实现？是在 catch 里做很多异常分支吗？

方向是对的：重试发生在 harness 层，LLM call 外面包恢复逻辑。

但不是简单包整个 loop，而是包**每一次 LLM call**。

教学版结构：

```text
agent_loop
  └─ while True
      └─ outer try/except
          └─ with_retry(...)
              └─ client.messages.create(...)
```

代码形态：

```python
while True:
    try:
        response = with_retry(
            lambda mt=max_tokens, mdl=state.current_model:
                client.messages.create(
                    model=mdl,
                    system=system,
                    messages=messages,
                    tools=TOOLS,
                    max_tokens=mt,
                ),
            state,
        )
    except Exception as e:
        ...
```

### 第一层：`with_retry()` 处理临时错误

`with_retry()` 主要处理 429 / 529：

```text
429 rate limit:
  exponential backoff
  sleep
  continue retry

529 overloaded:
  exponential backoff
  连续多次可切 fallback model
  continue retry

其他错误:
  raise，交给外层
```

### 第二层：outer try/except 处理非瞬时错误

例如 context 太长：

```python
except Exception as e:
    if is_prompt_too_long_error(e):
        if not state.has_attempted_reactive_compact:
            messages[:] = reactive_compact(messages)
            state.has_attempted_reactive_compact = True
            continue
        return

    log_unrecoverable()
    return
```

这里的 `continue` 是回到 `while True` 顶部，重新发起 LLM call。

### 第三类不走 except：`max_tokens` 是 stop_reason

`max_tokens` 不是异常，因为 API 成功返回了，只是没自然完成。

所以它在 response 后检查：

```python
if response.stop_reason == "max_tokens":
    if not state.has_escalated:
        max_tokens = ESCALATED_MAX_TOKENS
        state.has_escalated = True
        continue

    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": CONTINUATION_PROMPT})
    continue
```

因此可以这样理解：

```text
try/except = 处理“调用失败”
stop_reason 分支 = 处理“调用成功但没自然完成”
```

`catch` / `except` 里确实会有很多异常处理分支，但它只是恢复系统的一部分。完整错误恢复还包括 response 状态分支、恢复状态记录、有限重试和停止条件。

---

## 参考链接

| 主题 | 路径 |
|------|------|
| 教学版 Error Recovery 说明 | [`s11_error_recovery/README.md`](../../s11_error_recovery/README.md) |
| 教学版 Error Recovery 代码 | [`s11_error_recovery/code.py`](../../s11_error_recovery/code.py) |
| System Prompt 问答 | [`system-prompt-qa.md`](system-prompt-qa.md) |
| Context Compact 说明 | [`s08_context_compact/README.md`](../../s08_context_compact/README.md) |
| Agent Loop 说明 | [`s01_agent_loop/README.md`](../../s01_agent_loop/README.md) |

---

*文档版本：2026-06-28*
