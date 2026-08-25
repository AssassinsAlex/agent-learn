# Agent Core 第一讲：Agent、Session 与 Agent Loop

基线：`pi-earendil` commit `864b35c462f9623579b068e9cab848419f9e1d0f`。

## 1. 先区分三个对象

### `main.ts`

CLI 的组装器，不是 agent 本身。它负责解析参数、判断运行模式、选择/创建 session、创建 cwd 绑定的 services，再把控制权交给 interactive、print 或 RPC mode。

关键路径：

```text
main(args)
  -> resolveAppMode()
  -> createSessionManager()
  -> createRuntime()
       -> createAgentSessionServices()
       -> createAgentSessionFromServices()
  -> createAgentSessionRuntime()
  -> runRpcMode() / InteractiveMode.run() / runPrintMode()
```

证据：`packages/coding-agent/src/main.ts:540-578,615-745,815-855`。

### `Agent`

`packages/agent/src/agent.ts` 中的状态化 runtime 外壳。它持有：

- `AgentState`：system prompt、当前 model、thinking level、tools、transcript、streaming 状态。
- `streamFunction`：实际向 `pi-ai` 发起模型流请求的函数。
- `steeringQueue` / `followUpQueue`：运行中插入消息和 agent 原本要结束后再插入的消息。
- listeners：按订阅顺序接收并等待 `AgentEvent`。
- abort controller：当前 run 的取消边界。

`prompt()` 不允许并发启动；运行中要用 `steer()` 或 `followUp()`。`continue()` 要求最后一条消息不是 assistant。

证据：`packages/agent/src/agent.ts:164-228,230-374`。

### `AgentSession`

`packages/coding-agent/src/core/agent-session.ts` 是 coding-agent 的业务编排层。它包住一个 `Agent`，并额外负责：

- SessionManager 持久化和 session tree 操作。
- system prompt、工具注册/过滤、模型和认证。
- 扩展 hooks、输入转换、命令、compaction、retry、bash 状态。
- 把底层 `AgentEvent` 转发成面向 coding-agent/UI 的 session 事件。
- 在 `message_end` 等事件处保存会话消息。

构造时立即订阅 agent 事件，安装 tool hooks 和 next-turn refresh，再构建 runtime 工具集合。证据：`packages/coding-agent/src/core/agent-session.ts:286-384`。

## 2. `main.ts` 的启动职责

启动顺序不是“创建 Agent 后再补配置”，而是先确定最终 cwd 和 session，再创建 cwd-bound services：

1. `parseArgs`，并按 stdin/stdout TTY、`--print`、`--mode` 决定 `interactive`、`print`、`json` 或 `rpc`。
2. 运行 migrations，创建启动期 SettingsManager。
3. `createSessionManager` 处理新建、恢复、继续、fork、指定 session 和 in-memory session。
4. 依据 session cwd 建立 project trust、settings、resource loader、extensions、skills、prompt templates 和 model runtime。
5. `createAgentSessionFromServices` 创建 `AgentSession`，再由 `createAgentSessionRuntime` 持有它。
6. 准备 stdin/file 初始消息和主题，最后把 runtime 交给对应 mode。

设计要点：session 可能来自其他项目，所以 project-local resources/provider/model 必须等目标 session cwd 确定后解析。证据：`main.ts:568-591`。

## 3. `agentLoop` 的状态机

低层入口有两个：

- `agentLoop(prompts, context, config, signal, streamFunction)`：把新 prompt 追加到 context。
- `agentLoopContinue(context, ...)`：从已有 user/toolResult 继续，禁止从 assistant 继续。

主循环可写成：

```text
agent_start
  turn_start
  [注入 steering]
  stream assistant
  if error/aborted:
      turn_end -> agent_end
  if tool calls:
      execute tool batch
      append ToolResultMessage
  turn_end
  prepareNextTurn
  shouldStopAfterTurn?
  poll steering
  没有 tool/steering 时 poll follow-up
  没有 follow-up -> agent_end
```

源码把它实现为 outer loop + inner loop：inner loop 消化 tool calls 和 steering；outer loop 在 agent 本来要结束时检查 follow-up。证据：`packages/agent/src/agent-loop.ts:154-274`。

## 4. 模型请求边界

`streamAssistantResponse()` 是 AgentMessage 世界和 LLM Message 世界的唯一关键转换边界：

1. `transformContext(messages)`：可做裁剪或外部上下文注入。
2. `convertToLlm(messages)`：过滤 UI-only/custom message，转换为 provider 可接受的消息。
3. 组装 `Context { systemPrompt, messages, tools }`。
4. 每次请求动态解析 API key，适配短生命周期 OAuth。
5. 调用 `streamFunction(model, llmContext, options)`。
6. 逐个消费 `AssistantMessageEvent`，发出 `message_start/update/end`。

证据：`packages/agent/src/agent-loop.ts:276-356`。

## 5. 工具批处理语义

assistant message 中的 `toolCall` 被收集成一个 batch：

- `config.toolExecution === "sequential"`，或任何工具声明 `executionMode: "sequential"` 时，逐个执行。
- 否则先顺序完成参数验证、`beforeToolCall` 等 preflight，再并发执行允许的工具。
- 工具完成事件按完成时机发出；工具结果消息按 assistant 源码顺序组装。
- `beforeToolCall` 可以 block；`afterToolCall` 可以覆盖结果、错误标志和 `terminate`。
- 只有 batch 中所有 finalized 结果都要求 terminate 时，才提前结束循环。
- assistant 以 `stopReason: "length"` 结束时，tool arguments 可能截断，所有调用都会生成失败结果而不是执行。

证据：`packages/agent/src/agent-loop.ts:201-223,407-551`；契约定义在 `packages/agent/src/types.ts` 的 `AgentLoopConfig`、`AgentTool`、`AgentEvent`。

## 6. Session 与 Agent 的事件关系

```text
Agent.processEvents(event)
  -> 先归约 Agent 内部状态
  -> await Agent listeners
       -> AgentSession._handleAgentEvent
       -> 扩展 hooks、事件通知、持久化/compaction/retry
            -> session listeners
                 -> InteractiveMode / RPC / print
```

`AgentSession._handleAgentEvent` 的顺序是：先处理队列显示状态，先发扩展事件，再通知 session listeners，最后在 `message_end` 持久化普通 user/assistant/toolResult 消息；custom 消息走 `appendCustomMessageEntry`，bash/compaction/branch summary 由各自逻辑保存。证据：`packages/coding-agent/src/core/agent-session.ts:577-649`。

`agent_end` 表示不再产生新的 loop 事件，但 `Agent` 仍会等待该事件的 listeners；只有 listeners settle 后 `isStreaming` 才清除，`waitForIdle()` 才完成。证据：`packages/agent/src/agent.ts:230-243,511-573`。

## 7. 第一讲结论

Pi 的核心不是一个巨大的 `AgentSession` 循环，而是三层职责分离：

1. **CLI/runtime 层**决定运行环境和资源。
2. **Session 层**把 coding-agent 的业务能力接到通用 agent 上，并负责持久化、扩展和恢复。
3. **Agent 层**只负责可复用的消息循环、模型流、工具批处理和事件生命周期。

下一讲应逐行阅读 `streamAssistantResponse()`，然后分析 `AgentSession._handleAgentEvent` 如何把每个底层事件写入 session 和 UI。
