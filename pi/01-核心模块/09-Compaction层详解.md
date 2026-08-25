# Agent Core Compaction 层详解

> 范围：`packages/agent` 的通用 Harness compaction。本文不把 coding-agent 的自动触发器当成 compaction 算法本身。

## 1. 它解决什么问题

模型的 context window 是有限的。一次长会话会持续累积 user、assistant、tool call、tool result 等消息；如果只做“从数组头部删除”，模型会丢失用户目标、已经完成的工作、关键决策和下一步。

Pi 的 compaction 是一种**上下文投影**：持久化 session history 仍然保留，下一次发给模型的 context 改为“历史摘要 + 最近原文 + 压缩之后的新消息”。因此它不是删除会话，也不是简单截断 `AgentState.messages`。

## 2. 总体数据流

```text
Session.getBranch() / active leaf path
        |
        v
estimateContextTokens + shouldCompact
        |
        v
prepareCompaction
  ├─ 找上一次 compaction 边界
  ├─ findCutPoint（保留最近 keepRecentTokens）
  ├─ 拆出待摘要历史
  └─ 拆出 retainedTail
        |
        v
compact
  ├─ generateSummaryWithUsage（调用模型）
  ├─ split-turn 时额外摘要 turn prefix
  └─ 生成 CompactionResult
        |
        v
Session.appendCompaction(...)
        |
        v
下次 Session.buildContext() 
  = compaction summary + retainedTail + 后续 branch messages
```

通用 Harness 的手动入口在 `AgentHarness.compact()`；它检查 harness 必须处于 idle，读取当前 branch，准备并生成结果，再通过 Session 持久化 compaction entry。源码：[agent-harness.ts:701](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/agent-harness.ts#L701)。

这里要区分“能力”和“触发器”：`shouldCompact()` 在 agent-core 中只是导出的判断函数，通用 Harness 的 `compact()` 是显式操作。当前版本的自动阈值检查、overflow recovery 和 retry 编排位于 coding-agent 的 `AgentSession`，属于上层应用策略，不是 `agentLoop` 自己偷偷执行的动作。

## 3. 触发判断与 token 估算

默认设置定义在 `DEFAULT_COMPACTION_SETTINGS`：

```text
enabled          = true
reserveTokens    = 16384   // 给摘要 prompt 和摘要输出预留
keepRecentTokens = 20000   // 压缩后尽量保留的近期消息量
```

判断公式是：

```ts
contextTokens > contextWindow - reserveTokens
```

`estimateContextTokens()` 优先使用最近一条有效 assistant message 的 provider usage，再对其后的消息做估算；如果没有 usage，则对全部消息采用保守的字符数/4 估算。图片按固定字符预算估算，tool call 的名称和 JSON 参数也计入。这样可以避免 provider usage 只覆盖到某个时间点时重复计算整段历史。

相关源码：[compaction.ts:137](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L137)、[compaction.ts:152](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L152)、[compaction.ts:235](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L235)。

## 4. 为什么不能随便截断：cut point

`findCutPoint()` 从尾部累计消息 token，找到接近 `keepRecentTokens` 的位置，但只允许落在有效消息边界，且不会把 `toolResult` 单独当成切点。切点如果位于一个正在进行的 turn 中间，就标记 `isSplitTurn`：

```text
历史部分 ... | turn prefix | turn suffix（retainedTail）
                 ↑ 摘要         ↑ 保留原文
```

此时 `compact()` 会生成两段语义：历史摘要，以及“这个 turn 前缀做了什么、如何理解保留下来的 suffix”的 turn-prefix 摘要。这样不会把一个工具调用链切成模型无法解释的半截。

源码：[compaction.ts:300](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L300)、[compaction.ts:367](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L367)、[compaction.ts:603](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L603)。

## 5. 摘要是什么格式

摘要由模型生成，但 prompt 要求固定结构，核心段落为：

```text
## Goal
## Constraints & Preferences
## Progress
### Done / In Progress / Blocked
## Key Decisions
## Next Steps
## Critical Context
```

如果已有 compaction summary，下一次摘要使用 update prompt，把新历史合并进旧摘要，而不是从零开始。这使多次 compaction 成为增量 checkpoint。摘要调用的 `maxTokens` 不超过 `0.8 * reserveTokens`，并受模型自身 `maxTokens` 限制；支持 reasoning level，并把 provider usage 返回。

源码：[compaction.ts:418](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L418)、[compaction.ts:494](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L494)。

## 5.1 切点处的 user 消息到底保不保留

这里需要区分两种情况。假设一个 turn 的结构是：

```text
U = user request
A1 = assistant/tool call
T1 = tool result
A2 = assistant continuation
```

### 情况 A：切点正好落在 user entry

`findCutPoint()` 会把 `isUserMessage` 设为 true，`isSplitTurn` 为 false。此时 `U` 是 `firstKeptEntryId` 对应的第一个保留 entry，会作为原文进入 `retainedTail`，不会被第二次 prefix summary 摘要。

### 情况 B：切点落在 turn 中间

例如切点落在 `T1` 或 `A2` 前面。`findTurnStartIndex()` 会向前找到 `U`，于是：

```text
messagesToSummarize = 更早 turn 的消息
turnPrefixMessages  = [U, A1]（切点之前的当前 turn 前缀）
retainedTail        = [T1, A2, ...]（切点之后的后缀）
```

因此，`U` 不会再次作为 raw message 放进 retained tail；它会被包含在第二次 prefix summary 的输入中。这样做是为了避免把同一条 user request 在 summary 和 retained tail 中重复发送，同时仍保留其语义。

还有一个边界：如果从 `startIndex` 向前找不到 user 或 bash 起点，`turnStartIndex` 为 `-1`，这时不会标记 split-turn；实现会退化为普通切点处理。源码：[compaction.ts:340](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L340)、[compaction.ts:391](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L391)。

## 5.2 split-turn 为什么要两次压缩

两次调用不是重复摘要，而是两个不同的语义任务：

### 第一次：历史摘要（history summary）

输入是 `messagesToSummarize`，也就是当前 turn 之前、已经完全结束的历史。若之前存在 compaction entry，还会把旧的 `previousSummary` 放在 `<previous-summary>` 标签中，要求模型在旧 checkpoint 上增量更新。

使用通用摘要 prompt，要求输出：

```text
## Goal
## Constraints & Preferences
## Progress
### Done / In Progress / Blocked
## Key Decisions
## Next Steps
## Critical Context
```

这一次回答的是：“在进入当前 turn 之前，整个任务已经发展到什么状态？”

实际请求不是把多条历史消息原样作为 provider messages，而是先经过 `convertToLlm()` 和 `serializeConversation()`，包装成一个 user prompt：`<conversation>...</conversation>`，后面追加摘要指令。两种摘要共享同一个 system prompt：`You are a context summarization assistant... ONLY output the structured summary.` 历史摘要的输出上限为 `min(0.8 * reserveTokens, model.maxTokens)`。

### 第二次：turn-prefix 摘要

输入是 `turnPrefixMessages`，也就是当前 turn 的前缀，通常包含 user request 以及若干 assistant/tool 交互。

它使用专门的 `TURN_PREFIX_SUMMARIZATION_PROMPT`，要求输出：

```text
## Original Request
[当前 turn 的用户请求]

## Early Progress
[前缀阶段已经完成的决策和工作]

## Context for Suffix
[理解保留 suffix 所必需的信息]
```

这一次回答的是：“保留下来的 suffix 是当前工作的后半段，前缀发生了什么才能让它可理解？”它不要求复述完整长期历史，也不要求产生下一步任务清单。

prefix 摘要同样使用上面的 summarization system prompt，但把输出上限收紧为 `min(0.5 * reserveTokens, model.maxTokens)`；它的 prompt 只包含 prefix conversation 和 `TURN_PREFIX_SUMMARIZATION_PROMPT`，不会附带 `<previous-summary>`。如果启用了 reasoning model 且 thinking level 不是 `off`，两次调用都会分别带上 reasoning 参数。

源码中的调用顺序是：先对 `messagesToSummarize` 调用 `generateSummaryWithUsage()`，再对 `turnPrefixMessages` 调用 `generateTurnPrefixSummary()`。[compaction.ts:725](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L725)

## 5.3 两次结果如何合并

最终 `summary` 不是二选一，而是字符串拼接：

```text
{historyText}

---

**Turn Context (split turn):**

{turnPrefixText}
```

如果没有更早历史，`historyText` 使用 `No prior history.`；如果历史摘要调用失败，整个 compaction 失败，不会继续只生成 prefix summary。若历史摘要成功但 prefix 摘要失败，同样不会持久化半成品。

两次 provider usage 会通过 `combineUsage()` 相加，写入同一个 `CompactionEntry.usage`，所以成本和 token 统计不会丢失。测试明确验证了 input、output、cacheRead、cacheWrite 和 totalTokens 的合并。[compaction.test.ts:638](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/test/harness/compaction.test.ts#L638)

## 5.4 压缩后的实际 context

`Session.defaultContextEntryTransform()` 遇到最新 compaction entry 时，会丢弃它之前的普通 path entries，保留：

```text
compaction entry
  -> compactionSummary message（summary 全文）
  -> retainedTail（如果存在）
  -> compaction 之后追加的 branch entries
```

因此 split-turn 的结果是：用户请求 `U` 的信息出现在 `Turn Context (split turn)` 摘要中，而 `T1/A2` 等 suffix 以原始消息继续存在。模型接收到的是“前缀语义 + 后缀原文”，不是“用户请求原文 + 后缀原文”。源码：[session.ts:54](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/session/session.ts#L54)、[session.ts:116](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/session/session.ts#L116)。

## 5.5 压缩内容被怎样包裹

同一份摘要会经历三个不同层次的包装。不要把它们当成同一个对象：

```text
CompactionResult
  -> 持久化 CompactionEntry
    -> 内存 CompactionSummaryMessage
      -> Provider Message（最终发送给模型）
```

### 第一层：Session Tree 的持久化 entry

`AgentHarness.compact()` 将结果交给 `Session.appendCompaction()`，形成一个普通的树节点：

```ts
{
  type: "compaction",
  id,
  parentId: currentLeafId,
  timestamp,
  summary,
  firstKeptEntryId,
  tokensBefore,
  retainedTail,
  details,      // 如 readFiles / modifiedFiles
  usage,        // 一次或两次摘要调用的合计
  fromHook,
}
```

`parentId` 指向压缩前的 active leaf，因此它仍是可追溯的 session tree entry；被压缩的旧 entry 并未被这一操作删除。源码：[session.ts:260](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/session/session.ts#L260)、[types.ts:368](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/types.ts#L368)。

### 第二层：Agent Core 的专用消息类型

构建 context 时，Session 将该 entry 转成：

```ts
{
  role: "compactionSummary",
  summary: entry.summary,
  tokensBefore: entry.tokensBefore,
  timestamp,
}
```

并紧随其后展开 `retainedTail`。这个 role 是 Pi agent-core 的内部语义类型，方便 session transform、测试和调用方识别“这不是用户刚发的一句话”。源码：[messages.ts:42](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/messages.ts#L42)、[session.ts:123](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/session/session.ts#L123)。

### 第三层：发送给 Provider 的标准 LLM message

`agentLoop` 在真正调用 Provider 前执行 `convertToLlm()`。Provider 并不认识 `compactionSummary` 这个 Pi 私有 role，因此 Pi 把它降级为一条标准 `user` message，文本内容严格包成：

```text
The conversation history before this point was compacted into the following summary:

<summary>
{summary 原文}
</summary>
```

也就是说，最终模型上下文的大致顺序是：

```text
system prompt
user: The conversation history ... <summary>...</summary>
原始 retainedTail 消息（user / assistant / toolResult）
compaction 后的新消息
当前新输入
```

这里使用 `user` role 是一种兼容层做法，不代表这条内容真的是用户本轮输入；前缀文字和 XML 风格边界告诉模型它是历史 checkpoint。源码：[messages.ts:5](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/messages.ts#L5)、[messages.ts:120](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/messages.ts#L120)、[agent-loop.ts:294](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/agent-loop.ts#L294)。

## 5.6 `user` role 会混淆模型吗？连续压缩怎么办？

这是一种有意识的兼容性取舍，而不是完全没有风险的设计。Provider 通常只接受 `system`、`user`、`assistant`、`tool` 等有限 role，`compactionSummary` 是 Pi 内部类型，必须被投影为其中之一；Pi 选择 `user`，再用自然语言前缀和 `<summary>` 边界标明它是历史 checkpoint。

正常路径下，模型**不会同时收到两个相邻的 compaction summary message**。`defaultContextEntryTransform()` 会扫描当前 active branch，选择最后一个 `compaction` entry，并丢弃其之前的普通 path entries：

```text
第 1 次：历史 H0 -> summary S1
第 2 次：S1 + 后续历史 H1 -> 更新后的 summary S2
给模型：S2 + retained tail + 新消息
```

也就是说，第 2 次摘要并不是把 `S1` 和 `S2` 并排发送；`S1` 被作为 `<previous-summary>` 输入给**摘要模型**，产出新的 `S2`。最终 agent model 只看到 `S2` 这一份最新 checkpoint。split-turn 的两段摘要也会先在 `compact()` 内部拼成一个 `summary` 字符串，再以一个 compaction message 发送。

仍需认识到两个残余风险：

1. **语义混淆**：模型可能把摘要附近的内容误读成一条新用户指令。Pi 依靠固定前缀、XML 边界和消息位置缓解，但没有使用 provider 原生的“历史摘要”专用 role，因为大多数 Provider 并没有这个共同能力。
2. **间接提示注入**：摘要包含先前 user/tool 内容的提炼。如果那些内容含有恶意指令，摘要模型可能把它保留下来；它随后又以 user role 进入 agent model。system prompt 的优先级仍高于它，但这不是严格的安全隔离。

因此，从实现评价看：Pi 解决了跨 Provider 的 message schema 问题，也避免了多层 compaction message 叠加；但它把“摘要是可信上下文”的判断交给模型提示约束，不能把该包装视为安全边界。源码：[session.ts:54](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/session/session.ts#L54)、[compaction.ts:632](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L632)。

## 5.7 工具消息有没有额外的压缩规则

有，但主要是**边界保护、序列化裁剪和文件元数据提取**，不是再设计一套“工具调用专用摘要模型”。

### 1. 工具调用不会在压缩阶段重新执行

Compaction 只读取已有的 `assistant.toolCall` 和 `toolResult`，把它们转成摘要输入；它不会重新运行工具，也不会验证工具结果是否仍然有效。

### 2. `toolResult` 不能作为切点

`findValidCutPoints()` 明确不把 `toolResult` 加入合法切点。这样可以避免保留一个孤立的工具结果，却把对应的 assistant tool call 放进历史摘要，或者反过来只保留 tool call 而没有结果。切点可以落在 assistant message，但不能直接落在 tool result entry 上。

### 3. 发送给摘要模型前，会把工具消息转成纯文本

`serializeConversation()` 的格式大致是：

```text
[Assistant tool calls]: read(path="src/a.ts"); edit(path="src/a.ts", ...)

[Tool result]: {工具返回文本}
```

assistant 的 tool call 会保留工具名和 JSON 参数；tool result 会保留文本内容，但单条结果最多保留 `2000` 个字符，超出部分追加：

```text
[... N more characters truncated]
```

注意：这是“给摘要 prompt 的序列化裁剪”，不是删除 session 中原始的 tool result。原始 entry 仍在持久化历史中，`retainedTail` 中的工具消息也不会被这个函数改写。

### 4. 工具相关的 token 估算仍计入阈值

`estimateTokens()` 会计算 assistant tool call 的工具名和 JSON 参数，也会估算 tool result 的文本和图片内容。因此大工具输出会推动 compaction 触发，即使摘要 prompt 后续会对它做 2000 字符裁剪。

### 5. coding-agent 工具会额外生成文件操作索引

压缩范围内的 assistant tool call 会被扫描：

```text
read(path) -> readFiles
write(path) -> modifiedFiles
edit(path) -> modifiedFiles
```

这些路径不会替代摘要，而是追加到最终 summary 的元数据区：

```text
<read-files>
src/a.ts
</read-files>

<modified-files>
src/b.ts
</modified-files>
```

如果前一个 compaction 已经保存了文件详情，新的 compaction 会先继承这些详情，再扫描本轮工具调用；只要文件曾被写入或编辑，它就从 read-only 列表移动到 modified 列表。源码：[utils.ts:24](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/utils.ts#L24)、[utils.ts:62](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/utils.ts#L62)、[compaction.ts:45](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts#L45)。

因此可以这样记：**工具调用的行为语义交给通用摘要模型，工具链边界由 cut point 保护，超长返回由序列化层裁剪，文件变更由结构化 details 额外索引。**

## 6. CompactionEntry 保存什么

`compact()` 返回的 `CompactionResult` 主要包含：

- `summary`：替代旧历史进入未来 context 的结构化摘要。
- `firstKeptEntryId`：保留原始历史从哪个 entry 开始，便于树路径重建。
- `tokensBefore`：压缩前的估算 context token 数。
- `retainedTail`：切点之后保留的近期 AgentMessage 原文。
- `usage`：生成摘要所消耗的模型 usage；split-turn 时合并两次摘要调用。
- `details`：从被摘要消息提取出的读取/修改文件列表。

这意味着“压缩后的 context”与“原始 session log”是两种不同视图：entry 仍可审计和恢复，模型只看到投影后的短视图。

## 7. 和 Agent Loop 如何配合

Harness 在每轮开始通过 `session.buildContext()` 创建 turn state，交给 `runAgentLoop()`。loop 产生的 `message_end` 会由 Harness 写入 Session；下一轮的 `prepareNextTurn` 先 flush pending writes，再重新 `buildContext()`。因此 compaction 一旦写入，后续 turn 自然会看到新的 summary + retained tail，不需要修改 loop 的核心状态机。

源码：[agent-harness.ts:322](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/agent-harness.ts#L322)、[agent-harness.ts:407](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/agent-harness.ts#L407)、[agent-harness.ts:477](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/agent-harness.ts#L477)。

## 8. 与 branch summary 的区别

两者都可能调用摘要模型，但目的不同：

| 机制 | 目的 | 作用范围 | 结果 |
| --- | --- | --- | --- |
| Compaction | context window 快满时压缩当前分支早期历史 | 当前 active branch 的旧消息 | `compaction` entry，影响未来模型 context |
| Branch summary | 从一个 session tree 分支切到另一个分支时保留离开分支的工作信息 | 分支间共同祖先之后的路径 | `branch_summary` entry，描述被离开的分支 |

## 9. 异常与安全边界

- 空路径、末尾已经是 compaction 时，不重复压缩。
- 找不到带 UUID 的首个保留 entry 时返回 `invalid_session`。
- 摘要模型返回 `aborted` 或 `error` 时转为 `CompactionError`，不会写入半成品 entry。
- `AgentHarness.compact()` 要求 idle；执行期间 phase 为 `compaction`，结束后恢复 idle。
- 取消、hook 拒绝和 provider 失败都通过 Harness error normalization 向上报告。

## 10. 建议阅读与验证

1. 先读 [compaction.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts) 的 `shouldCompact`、`findCutPoint`、`prepareCompaction`、`compact`。
2. 再读 [session.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/session/session.ts) 的 `buildContext`，确认 compaction entry 如何重新投影消息。
3. 最后读 [compaction.test.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/test/harness/compaction.test.ts)，重点看 split-turn、retained tail、previous summary、usage 合并和错误测试。

一句话记忆：**compaction = 在持久化历史不丢失的前提下，用模型摘要替换旧 context，并保留最近原文和可定位的树节点。**
