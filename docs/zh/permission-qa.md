# Permission 问答

> 本文档整理自学习过程中的权限与授权相关问题，基于本仓库 `s03_permission`、`s04_hooks` 章节及 Claude Code（CC）源码附录说明。与 [Validation 问答](validation-qa.md) 互补：Validation 管「参数合法吗」，Permission 管「操作允许吗」。
>
> **创建日期**：2026-06-26 · **最后更新**：2026-06-26

---

## 目录

- [Q1：s03 Permission 有什么可以讲的？有什么重点？](#q1s03-permission-有什么可以讲的有什么重点)
  - [问题背景：s02 的 bash 无限制](#问题背景s02-的-bash-无限制)
  - [三道闸门模型](#三道闸门模型)
  - [循环不变，执行前插入检查](#循环不变执行前插入检查)
  - [与 Validation 的分工](#与-validation-的分工)
  - [CC 附录要点速览](#cc-附录要点速览)
  - [学习路径 s02 → s03 → s04](#学习路径-s02--s03--s04)
  - [动手试：s03 README 推荐 prompt](#动手试s03-readme-推荐-prompt)
- [Q2：CC 是如何判断一个 tool 给不给 permission 的？](#q2cc-是如何判断一个-tool-给不给-permission-的)
  - [4 种 PermissionResult behavior](#4-种-permissionresult-behavior)
  - [管线入口：checkPermissionsAndCallTool](#管线入口checkpermissionsandcalltool)
  - [8 个规则来源与优先级](#8-个规则来源与优先级)
  - [Hooks 与 resolveHookPermissionDecision](#hooks-与-resolvehookpermissiondecision)
  - [YoloClassifier 处理 ask 决策](#yoloclassifier-处理-ask-决策)
  - [示例：bash `rm test.txt` vs `rm -rf /`](#示例bash-rm-testtxt-vs-rm--rf-)
- [Q3：Yolo 是所有的 tool 都能执行吗？连最危险的 rm -rf 也能吗？](#q3yolo-是所有的-tool-都能执行吗连最危险的-rm--rf-也能吗)
  - [Yolo 只处理 ask 路径](#yolo-只处理-ask-路径)
  - [deny 与安全检查先于 Yolo](#deny-与安全检查先于-yolo)
  - [Yolo 内部决策流程](#yolo-内部决策流程)
  - [bypassPermissions 是另一回事](#bypasspermissions-是另一回事)
- [Q4：hasPermissionsToUseToolInner「命中就停」是什么意思？是 chain 吗？](#q4haspermissionstousetoolinner命中就停是什么意思是-chain-吗)
  - [是的：责任链 / 瀑布 / 短路](#是的责任链--瀑布--短路)
  - [有序检查表（s03 附录）](#有序检查表s03-附录)
  - [为什么用 chain](#为什么用-chain)
  - [与完整权限管线、教学版三道闸门的关系](#与完整权限管线教学版三道闸门的关系)
  - [「不可绕过 rules」的含义](#不可绕过-rules的含义)
- [参考链接](#参考链接)

---

## Q1：s03 Permission 有什么可以讲的？有什么重点？

### 问题背景：s02 的 bash 无限制

`s02` 给 Agent 加了 5 个工具。file tools 有 `safe_path()` 保护，但 **bash 不受限制**。让模型「清理一下项目」，它可能执行 `rm -rf /`。

安全不能靠信任模型，要靠代码——在工具**真正执行之前**做判断。这就是 `s03_permission` 要解决的核心问题（见 [`s03_permission/README.md`](../../s03_permission/README.md)「问题」一节）。

### 三道闸门模型

`s03` 的教学版用**三道闸门**概括权限决策，顺序固定：**硬拒绝优先，软询问次之，都没命中就放行**。

| 闸门 | 作用 | 命中后 |
|------|------|--------|
| 1. 拒绝列表 | 永远禁止的操作（`rm -rf /`、`sudo`） | 直接拒绝，不执行 |
| 2. 规则匹配 | 取决于上下文的操作（写工作区外、`rm` 文件） | 交给闸门 3 |
| 3. 用户审批 | 闸门 2 命中后，暂停等用户确认 | 用户决定允许或拒绝 |

三道都没命中 → 直接执行。大部分日常操作走这条路。

```
模型发出 tool_use
       │
       ▼
┌────────────────────────────────────────┐
│  闸门 1：DENY_LIST（硬拒绝）            │  命中 → ⛔ deny
├────────────────────────────────────────┤
│  闸门 2：PERMISSION_RULES（规则匹配）   │  命中 → 进入闸门 3
├────────────────────────────────────────┤
│  闸门 3：ask_user（用户审批）           │  y → allow / N → deny
└────────────────────────────────────────┘
       │ 三道都没命中
       ▼
   TOOL_HANDLERS 执行
```

### 循环不变，执行前插入检查

`s03` **完全保留** `s02` 的 agent loop。唯一变动：在 `TOOL_HANDLERS[name](**input)` 之前插入 `check_permission(block)`。

```python
# 在 agent_loop 中——s02 的循环只加了一行：
for block in response.content:
    if block.type == "tool_use":
        if not check_permission(block):           # ← 新增
            results.append({... "content": "Permission denied."})
            continue
        output = TOOL_HANDLERS[block.name](**block.input)  # s02 原有
        results.append(...)
```

重点：**门控在循环里，但决策逻辑在 `check_permission()` 函数里**——为 `s04` 把这段逻辑移到 hook 上做了铺垫。

### 与 Validation 的分工

两者都出现在「工具执行之前」，但回答的问题不同（详见 [Validation 问答 Q2](validation-qa.md#q2validationpermissionhooks-有什么区别为什么要单独抽象)）：

| 维度 | Validation（`s02`） | Permission（`s03`） |
|------|---------------------|---------------------|
| 核心问题 | 这次调用的参数**合法吗**？ | 这次操作**允许执行吗**？ |
| 判断依据 | Schema、`safe_path()`、工具语义 | 拒绝列表、规则匹配、用户审批 |
| 典型结果 | 验证失败 → 返回错误 | `allow` / `deny` / 询问用户 |
| 失败能否问用户 | 否（参数错了问也没用） | 是（策略性决策） |

在 CC 生产管线中，Validation（步骤 1–2）先于 Permission（步骤 6）执行；教学版在 `s03` 只加 Permission 层，`s02` 的 schema / `safe_path` 仍然保留。

### CC 附录要点速览

`s03` README 附录深入 CC 源码，以下是要点摘要：

| 主题 | 要点 |
|------|------|
| **4 种 behavior** | `allow` / `deny` / `ask` / `passthrough`（教学版只有前 3 种对应关系） |
| **8 个规则来源** | userSettings、projectSettings、localSettings、flagSettings、policySettings、cliArg、command、session |
| **Hooks 交互** | PreToolUse 可返回 `permissionBehavior`，但 hook `allow` **不能绕过** settings 的 deny/ask 规则 |
| **YoloClassifier** | auto 模式下用分类器 LLM 自动审批 `ask` 决策，减少弹窗 |
| **权限冒泡** | 子 Agent `permissionMode: 'bubble'`，弹窗冒泡到父 Agent 终端 |
| **isDestructive** | **纯 UI 标签**，不参与权限决策 |

### 学习路径 s02 → s03 → s04

| 阶段 | 做法 | 痛点 / 收获 |
|------|------|-------------|
| `s02` | 直接 `TOOL_HANDLERS[name](**input)` | bash 无限制，危险命令可执行 |
| `s03` | 循环内插入 `check_permission()` | 有门控了，但每加检查就改循环 |
| `s04` | `trigger_hooks("PreToolUse", block)` | 循环稳定，权限检查挂到 hook 上 |

演进动机：**先理解「执行前要有门控」（s03），再理解「门控不应写死在循环里」（s04）**。

### 动手试：s03 README 推荐 prompt

```sh
cd learn-claude-code
python s03_permission/code.py
```

| # | Prompt | 预期行为 |
|---|--------|----------|
| 1 | `Create a file called test.txt in the current directory` | 直接通过（写工作区内） |
| 2 | `Delete the file test.txt` | bash + `rm` 触发闸门 2 → 闸门 3 询问 |
| 3 | `What files are in the current directory?` | 只读，全部通过 |
| 4 | `Try to write a file to /etc/something` | 写工作区外，触发闸门 2 |

观察重点：哪些操作直接通过？哪些需要你确认？哪些被直接拒绝？

---

## Q2：CC 是如何判断一个 tool 给不给 permission 的？

### 4 种 PermissionResult behavior

教学版三道闸门与 CC 不完全一一对应。CC 的 `PermissionResult` 有 **4 个 behavior**（`types/permissions.ts:241-266`）：

| behavior | 含义 | 教学版对应 |
|----------|------|-----------|
| `allow` | 直接允许 | 三道闸门都没命中，或闸门 3 用户选 y |
| `deny` | 直接拒绝 | 闸门 1 命中 |
| `ask` | 弹出对话框问用户 | 闸门 2 命中 |
| `passthrough` | 工具不表态，交给通用管线决定 | 教学版无（CC 中最终常转为 `ask`） |

### 管线入口：checkPermissionsAndCallTool

CC 的权限判断不是孤立的 `check_permission()`，而是嵌在完整工具执行管线里（`toolExecution.ts:599-1745`）：

```
模型发出 tool_use
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│  1. Zod schema 验证                                       │
├──────────────────────────────────────────────────────────┤
│  2. validateInput()                                       │
├──────────────────────────────────────────────────────────┤
│  3. backfillObservableInput()                             │
├──────────────────────────────────────────────────────────┤
│  4. PreToolUse hooks（可返回 permissionBehavior）          │
├──────────────────────────────────────────────────────────┤
│  5. resolveHookPermissionDecision()                       │
├──────────────────────────────────────────────────────────┤
│  6. hasPermissionsToUseToolInner()  ← Permission 核心     │
│     （含 YoloClassifier 处理 ask）                         │
├──────────────────────────────────────────────────────────┤
│  7. tool.call()                                           │
└──────────────────────────────────────────────────────────┘
```

步骤 1–5 属于 Validation + Hooks；步骤 6 是 Permission 层的核心决策函数。

### 8 个规则来源与优先级

CC 没有单一的 `DENY_LIST` 文件。权限规则来自 **8 个来源**（`types/permissions.ts:54-62`）：

| 来源 | 配置位置 |
|------|----------|
| `userSettings` | `~/.claude/settings.json` |
| `projectSettings` | `.claude/settings.json` |
| `localSettings` | `settings.local.json` |
| `flagSettings` | Feature flags |
| `policySettings` | 企业管理策略 |
| `cliArg` | `--allowedTools` / `--deniedTools` |
| `command` | 内联命令 |
| `session` | 会话内临时授权 |

每条规则格式示例：`{ toolName: "Bash", ruleBehavior: "deny", ruleContent: "npm publish:*" }`。

多个来源的规则合并，**高优先级来源覆盖低优先级**（从低到高：user < project < local < flag < policy，加上 cliArg、command、session）。

### Hooks 与 resolveHookPermissionDecision

PreToolUse hook 可以返回 `permissionBehavior: allow/deny/ask/passthrough`（`s04` 附录）。但 CC 有一条**关键不变式**（`toolHooks.ts:325-331`）：

> **Hook 返回 `allow` 时，仍然要检查 settings 的 deny/ask 规则。**

`resolveHookPermissionDecision()` 负责协调 hook 决策与 settings 规则——即使用户的 hook 脚本说「允许」，settings 里的 deny 规则仍可覆盖。

```
PreToolUse hook 返回 allow
       │
       ▼
resolveHookPermissionDecision()
       │
       ├─ settings 有 deny rule 命中？ → deny（hook allow 被覆盖）
       ├─ settings 有 ask rule 命中？  → ask
       └─ 无冲突                      → allow
```

### YoloClassifier 处理 ask 决策

当管线得出 `ask` 决策时，在 **auto 模式**下不会每次都弹对话框。`classifyYoloAction`（`utils/permissions/yoloClassifier.ts:1012`）用分类器 LLM 判断是否可自动批准（详见 Q3）。

### 示例：bash `rm test.txt` vs `rm -rf /`

假设模型发出：

```json
{
  "type": "tool_use",
  "name": "bash",
  "input": { "command": "rm test.txt" }
}
```

| 阶段 | `rm test.txt` | `rm -rf /` |
|------|---------------|------------|
| 1–2 Validation | Schema ✅，`validateInput()` ✅ | Schema ✅（字符串合法） |
| 3–4 PreToolUse hooks | 日志 hook 记录，可能返回 passthrough | 同上 |
| 5 resolveHook | 无 hook deny → 继续 | 无 hook deny → 继续 |
| 6 hasPermissionsToUseToolInner | 未命中 deny rule；可能命中 ask rule（含 `rm `）→ `ask` | 命中 deny rule / 安全检查 → **`deny`** |
| 6b Yolo（若 ask） | 分类器可能自动批准删测试文件 | **到不了 Yolo**（已在 deny 短路） |
| 7 执行 | 用户确认或 Yolo 批准后执行 | ⛔ 直接拒绝 |

教学版对照：

| 闸门 | `rm test.txt` | `rm -rf /` |
|------|---------------|------------|
| 1. 硬拒绝列表 | 未命中 | ⛔ 命中 `rm -rf /` |
| 2. 规则匹配 | ⚠️ 命中「潜在破坏性命令」 | （已在闸门 1 拒绝） |
| 3. 用户审批 | 等待用户 y/N | — |

更完整的跨层示例见 [Validation 问答「综合示例」](validation-qa.md#综合示例bash-rm-命令的完整旅程)。

---

## Q3：Yolo 是所有的 tool 都能执行吗？连最危险的 rm -rf 也能吗？

### Yolo 只处理 ask 路径

**不能。** Yolo（YoloClassifier）**不是**「所有 tool 都能自动执行」的万能通行证。它只介入管线已经得出 **`ask`** 决策的路径——帮用户自动回答「这次要不要批准？」。

| 管线结果 | Yolo 是否介入 |
|----------|--------------|
| `deny` | ❌ 不介入，直接拒绝 |
| `allow` | ❌ 不介入，直接执行 |
| `ask` | ✅ 尝试自动审批 |
| `passthrough` | 先被转为 `ask`，再可能进入 Yolo |

### deny 与安全检查先于 Yolo

在 `hasPermissionsToUseToolInner()` 责任链中，以下检查**排在 Yolo 之前**，命中即短路：

- 整个工具被 **deny rule** 禁用 → `deny`
- 工具自己 `checkPermissions()` 返回 deny → `deny`
- **安全检查违规** → `ask`（不可绕过），但若规则本身是 deny 则更早拒绝

因此 `rm -rf /` 这类命令：

1. 通常在 deny rule 或安全检查阶段就被 **`deny`**
2. **根本不会进入** YoloClassifier
3. 即使用户开了 auto 模式，最危险的操作仍有硬屏障

### Yolo 内部决策流程

当决策为 `ask` 时，`classifyYoloAction` 按以下顺序尝试（`permissions.ts:620-686`、`yoloClassifier.ts:1012`）：

```
ask 决策
   │
   ├─ 1. acceptEdits 模式模拟
   │      acceptEdits 允许 → 直接批准
   │
   ├─ 2. 安全工具白名单
   │      命中白名单 → 直接批准
   │
   ├─ 3. 分类器 LLM（classifyYoloAction）
   │      LLM 判断安全 → 自动批准
   │      LLM 判断不安全 → 拒绝或回退
   │
   └─ 4. 回退到人工审批
          分类器连续拒绝太多次 → 弹窗问用户
```

Yolo 的设计目标是**减少低风险 `ask` 的弹窗**（如读文件、删临时测试文件），而非放开所有操作。

### bypassPermissions 是另一回事

CC 还有一个独立的 **`bypassPermissions` 模式**（`permissions.ts` 责任链中的一步）：

- 这是**显式的危险绕过模式**，与 YoloClassifier 是不同机制
- 在责任链中表现为：命中 bypass → 直接 `allow`
- 正常使用不应依赖此模式；它与 Yolo 的「智能审批 ask」是两条路径

| 机制 | 作用 | 能绕过 deny？ |
|------|------|--------------|
| YoloClassifier | 自动回答 `ask` | ❌ 不能（deny 先于 Yolo） |
| bypassPermissions | 跳过权限检查 | ⚠️ 是显式绕过模式，非默认 |

---

## Q4：hasPermissionsToUseToolInner「命中就停」是什么意思？是 chain 吗？

### 是的：责任链 / 瀑布 / 短路

**是的。** `hasPermissionsToUseToolInner()`（`utils/permissions/permissions.ts:1158-1310`）实现的是典型的 **责任链（Chain of Responsibility）** / **瀑布（Waterfall）** / **短路（Short-circuit）** 模式：

- 按**固定顺序**依次检查
- **第一个命中**的检查决定最终结果
- 命中后**立即停止**，不再执行后续检查

```typescript
// 概念示意（非源码摘录）
function hasPermissionsToUseToolInner(tool, input) {
  if (toolDeniedByRule)        return deny      // 命中 → 停
  if (toolAskedByRule)         return ask       // 命中 → 停
  if (tool.checkPermissions)   ...            // 命中 → 停
  if (requiresUserInteraction) return ask       // 命中 → 停
  if (contentAskRule)          return ask       // 不可绕过，命中 → 停
  if (securityViolation)       return ask       // 不可绕过，命中 → 停
  if (bypassPermissions)       return allow     // 命中 → 停
  if (toolAllowedByRule)       return allow     // 命中 → 停
  return passthrough  // 全部未命中 → 交给上层（常转为 ask）
}
```

### 有序检查表（s03 附录）

`s03_permission/README.md` 附录「二、生产版的验证阶段」列出了 `hasPermissionsToUseToolInner()` 内部的有序检查：

| 顺序 | 检查项 | 命中结果 | 可绕过？ |
|------|--------|----------|----------|
| 1 | 整个工具被 deny rule 禁用 | `deny` | — |
| 2 | 整个工具被 ask rule 标记 | `ask` | — |
| 3 | `tool.checkPermissions()` 工具自己的判断 | 视工具而定 | — |
| 4 | 工具自己返回 deny | `deny` | — |
| 5 | `requiresUserInteraction()` | `ask` | — |
| 6 | 内容相关的 ask 规则 | `ask` | **不可绕过** |
| 7 | 安全检查违规 | `ask` | **不可绕过** |
| 8 | `bypassPermissions` 模式 | `allow` | 显式绕过 |
| 9 | 整个工具被 allow rule 放行 | `allow` | — |
| 10 | 全部未命中 | `passthrough` | 上层常转为 `ask` |

### 为什么用 chain

| 原因 | 说明 |
|------|------|
| **可预测** | 固定顺序 → 同样输入永远同样结果，便于调试和文档化 |
| **安全优先** | deny 检查排在 allow 之前；不可绕过的 ask 规则排在 bypass 之前 |
| **性能** | 命中即停，避免无谓的后续规则匹配和 LLM 分类器调用 |

如果改成「所有规则并行投票」，会出现优先级冲突、难以推理、安全规则可能被低优先级 allow 覆盖等问题。

### 与完整权限管线、教学版三道闸门的关系

三层抽象可以对照理解：

```
完整 CC 管线                    教学版 s03
─────────────────────────────────────────────────
checkPermissionsAndCallTool     agent_loop
  ├─ Validation (1-2)             ├─ (s02 schema / safe_path)
  ├─ Hooks (4-5)                  ├─ (s04 才有)
  └─ hasPermissionsToUseToolInner   └─ check_permission()
       （责任链，10 步）                 （三道闸门，3 步）
```

- **教学版三道闸门**是 `hasPermissionsToUseToolInner` 的极简投影
- **责任链**是 CC 生产版 Permission 层的内部实现模式
- 两者都是「有序检查、命中就停」，只是 CC 步骤更多、来源更复杂

### 「不可绕过 rules」的含义

步骤 6、7 标注为**不可绕过**，含义是：

- 即使 hook 返回 `allow`
- 即使后面有 `allow rule` 或 `bypassPermissions`
- 只要内容相关 ask 规则或安全检查命中，**仍然得到 `ask`（或更早的 `deny`）**

这正是 `s04` 附录强调的安全不变式在责任链层面的体现：**某些策略性规则优先级高于「放行」类决策**，防止扩展点（hook、allow rule）意外打开安全漏洞。

---

## 参考链接

| 主题 | 仓库路径 |
|------|----------|
| 权限三道闸门与 CC 权限阶段 | [`s03_permission/README.md`](../../s03_permission/README.md) |
| Hook 事件、Hook allow 不变式 | [`s04_hooks/README.md`](../../s04_hooks/README.md) |
| Validation 与 Permission 分工 | [`validation-qa.md`](validation-qa.md) |
| 工具分发与验证管线（附录） | [`s02_tool_use/README.md`](../../s02_tool_use/README.md) |
| Agent loop 基础 | [`s01_agent_loop/README.md`](../../s01_agent_loop/README.md) |
| 教学版 s03 可运行代码 | [`s03_permission/code.py`](../../s03_permission/code.py) |
| 教学版 s04 可运行代码 | [`s04_hooks/code.py`](../../s04_hooks/code.py) |

**CC 源码参考**（附录中引用，本仓库不含 CC 源码）：

- `toolExecution.ts` — `checkPermissionsAndCallTool()` 主编排
- `utils/permissions/permissions.ts` — `hasPermissionsToUseToolInner()`
- `types/permissions.ts` — `PermissionResult`、8 个规则来源
- `utils/permissions/yoloClassifier.ts` — `classifyYoloAction()`
- `toolHooks.ts` — PreToolUse 钩子、`resolveHookPermissionDecision` 相关逻辑
- `tools/AgentTool/forkSubagent.ts` — 权限冒泡（`permissionMode: 'bubble'`）
- `Tool.ts` — `isDestructive()`（UI 标签，不参与权限）

---

*文档版本：2026-06-26*
