# Pi AI：Message 与 Stream 归一化

基线：[`pi` commit `864b35c`](https://github.com/earendil-works/pi/tree/864b35c462f9623579b068e9cab848419f9e1d0f)。

## 1. 要解决的不是“统一 API 地址”

Anthropic、OpenAI Chat Completions、OpenAI Responses、Google 等 provider 的差异不止请求 URL：system prompt 位置、tool schema、tool-call id 限制、thinking/reasoning 的续传 token、图像支持、SSE event、结束原因、usage 字段都不同。

`pi-ai` 的目标是让上层 `agentLoop` 只面对一个稳定契约：

```text
AgentContext
  -> pi-ai Context / Message / Tool
  -> provider-specific request payload
  -> provider-native stream / SSE / WebSocket
  -> AssistantMessageEvent stream
  -> AssistantMessage（成功）或 AssistantMessage(error / aborted)
```

它不是抹掉一切差异：`AssistantMessage` 保留 `api`、`provider`、`model`、`responseId`、thinking signature 与 diagnostics；`Model` 也保留 input、context window、tool、thinking、cache 等能力标志。归一化的是**上层消费的控制面和数据形状**，不是模型能力本身。

源码入口：[types.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/ai/src/types.ts)、[transform-messages.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/ai/src/api/transform-messages.ts)。

## 2. 统一的轨迹数据模型

### 输入：`Context`

每个 stream adapter 收到同一种 `Context`：

| 字段 | 语义 | 算法含义 |
| --- | --- | --- |
| `systemPrompt` | 每次模型请求的系统约束 | policy / memory / safety 注入位置。 |
| `messages` | `user`、`assistant`、`toolResult` 序列 | trajectory 的可见状态，不等于完整 session 日志。 |
| `tools` | JSON-schema 参数定义的工具集合 | 模型 action space 的结构化部分。 |

`Message` 共有三种 role：

```text
user       = 文本或文本 + 图像
assistant  = text | thinking | toolCall 的有序 content blocks
toolResult = toolCallId + toolName + content + isError + details
```

`details` 主要供 UI/日志用；真正再次喂给模型的是 `toolResult.content`。算法上应把它当作 observation channel，而不是把任意执行细节默认塞入 context。

### 输出：`AssistantMessage`

一个 provider turn 最终收敛为一个完整 `AssistantMessage`。其中 `stopReason` 被标准化为：

```text
stop | length | toolUse | error | aborted
```

`Usage` 也被统一为 input、output、cacheRead、cacheWrite、totalTokens 和 cost；provider 没有上报的字段由 adapter 用约定值处理。这样上层才可横向比较成本、上下文使用和终止原因。

## 3. Stream 是主契约，final message 是流的终点

所有 adapter 需实现：

```ts
type StreamFunction = (
  model: Model,
  context: Context,
  options?: StreamOptions,
) => AssistantMessageEventStream;
```

关键约束是：请求、模型或运行时失败应编码在返回流的终止事件中，而不是直接抛出异常。`AssistantMessageEventStream` 的 terminal event 为 `done` 或 `error`，其 `result()` 分别得到成功 message 或带 `stopReason: error | aborted` 的 message。

统一事件序列为：

```text
start(partial)
  -> text_start / text_delta* / text_end
  -> thinking_start / thinking_delta* / thinking_end
  -> toolcall_start / toolcall_delta* / toolcall_end
  -> done(message) | error(error-message)
```

`partial` 是同一个逐步填充的 assistant message：UI 可显示增量文字，agent loop 则只在终止事件后把完整 message 写入正式 transcript。`toolcall_delta` 的 arguments 是流式 JSON；adapter 需要累积字符串并做容错解析，只有 `toolcall_end` 才代表一个可执行的结构化调用完成。

`pi-messages` adapter 是最直接的参照：它把后端 SSE 的同名事件重建成 `partial`，并把最终 usage、response id、rewrite diagnostics 写入 terminal message。[pi-messages.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/ai/src/api/pi-messages.ts)

## 4. 历史重放前的语义修复：`transformMessages`

历史并不能原样发给任意 provider。`transformMessages(messages, targetModel, normalizeToolCallId)` 在请求编码前完成跨模型兼容处理：

1. **能力降级**：目标模型不支持图像时，用明确 placeholder 替换 user/tool output 图像，保留“此处有图”的事实。
2. **thinking 签名约束**：同一 provider/api/model 时保留 opaque reasoning signature，以支持该 API 的多轮连续性；切换模型时丢弃 redacted/opaque signature，并把可见 thinking 降为普通 text。
3. **tool-call id 迁移**：某些 API 的 id 很长或含特殊字符，另一些 API 有长度/字符限制；跨模型时规范化 call id，并同步重写对应 tool result 的 `toolCallId`。
4. **不完整轨迹修复**：跳过 error/aborted assistant turn；若历史出现没有 tool result 的 tool call，在下一个 assistant/user 前补一个 `No result provided` 的 synthetic error result。

这说明 canonical message 不是“无损的通用 AST”。它有 provider 特有残留，并在每一次 target-model replay 前修复成可接受的近似历史。做模型切换、离线重放或经验检索时，必须保留这个 transform，而不能直接序列化 messages 后转发。

## 5. Adapter 的两层工作

```text
canonical Context
  -> transformMessages（跨模型语义修复）
  -> provider request encoder（角色、content blocks、tools、cache/reasoning/options）
  -> upstream transport
  -> provider event decoder（SSE/WebSocket/chunk）
  -> canonical AssistantMessageEvent
```

以 OpenAI Completions 与 Anthropic 为例：两者都会调用 `transformMessages`，但前者要按 `tool_calls` / `tool_call_id` 编码，后者要构造 `tool_use` / `tool_result` content blocks。它们都将 native finish reason 和 usage 映射为统一终止事件。对上层而言，`agentLoop` 不需要知道这些差异。

学习源码：[openai-completions.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/ai/src/api/openai-completions.ts)、[anthropic-messages.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/ai/src/api/anthropic-messages.ts)。

## 6. 与算法工作的连接

| 课题 | 应在此层观测 / 控制什么 |
| --- | --- |
| Model routing | `Model` capability、context window、usage、latency、cost，而不是只按模型名路由。 |
| Tool policy | `toolcall_end` 后的完成调用、失败率、参数有效率；不要从 delta 猜测最终 action。 |
| Long-term memory | 注入 `Context.systemPrompt` 或 messages 前，先经过 token budget 和目标模型兼容处理。 |
| Offline replay | 保留 provider/model 元数据与 `transformMessages`，否则 reasoning signature/tool id 可能使重放失真。 |
| Provider benchmark | 分别统计 provider error/aborted、`length`、tool-use、cache 和 usage，避免将协议问题误归因于 agent 策略。 |

## 7. 易混淆点

- `supportsToolSearch` 不是 internet web search，而是 provider 支持延迟发送 tool schema 的能力。
- `StreamOptions.maxRetries` 是 provider 请求层重试参数；agent 的上下文 overflow recovery 属于更上层的 `AgentSession` 策略。
- 流式 `partial` 用于展示和实时观测；最终 `AssistantMessage` 才是后续 loop 和 session 的稳定状态。
- adapter 将错误编码成 stream event，不表示所有调用方都无需处理异常；上层 hook/transform 的实现仍应满足各自契约。

## 8. 建议实验

1. 为 `pi-messages` 写 mock SSE，断言从 delta 到 terminal message 的重建结果。
2. 以同一带 tool call 的 history 分别喂给 OpenAI/Anthropic encoder，比较 tool id 与 tool result 的对应关系。
3. 切换模型后检查 thinking signature 与图像 placeholder，确认跨模型重放不是机械复制。
4. 记录每个 terminal `stopReason`、usage、latency，作为后续 routing 或 memory 策略的基线。
