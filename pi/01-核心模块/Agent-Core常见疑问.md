# Agent Core 常见疑问

## 1. Runtime 是 Harness 的一部分吗？

不是严格的包含关系。更准确的关系是：

```text
Harness 使用 Runtime

AgentHarness
  ├── 持有 session
  ├── 构造 AgentContext
  ├── 构造 AgentLoopConfig
  ├── 注入工具和 hooks
  └── 调用 agentLoop / Agent
          └── executeToolCalls()
```

### `executeToolCalls()` 属于谁？

它属于 `packages/agent/src/agent-loop.ts` 的 Runtime 层，是通用的工具调度算法。它负责：

1. 从 assistant message 找出 `toolCall`。
2. 校验和准备参数。
3. 调用 `beforeToolCall`。
4. 调用 `AgentTool.execute()`。
5. 调用 `afterToolCall`。
6. 发出 `tool_execution_*` 事件。
7. 生成 `ToolResultMessage`，交回 agent loop。

Runtime 并不知道工具是不是 bash、读文件还是数据库查询，也不知道权限系统长什么样。工具本身通过 `AgentTool` 注入：

```typescript
const tool: AgentTool = {
  name: "read_file",
  parameters: schema,
  execute: async (id, args, signal, onUpdate) => {
    // 具体能力由应用提供
    return result;
  },
};
```

因此答案是：

- `executeToolCalls()` 的**调度能力由 Runtime 自己提供**。
- 工具的**具体执行能力由调用方提供**。
- Harness 可以提供工具、权限 hook、事件 hook，但 Runtime 不依赖 Harness 才能执行工具。
- 直接使用 `Agent` 时，也可以完全不使用 Harness，手工传入 `AgentTool[]`。

源码证据：`packages/agent/src/agent-loop.ts:201-223,410-551`；`packages/agent/src/types.ts:380-404`。

## 2. Harness 的 Session 如何和 Agent Loop 配合？

先纠正一个容易混淆的点：**低层 `agentLoop` 没有 Session。**

低层 loop 只接收：

```text
AgentContext {
  systemPrompt,
  messages,
  tools
}
```

它把新的 assistant/tool messages 放到当前 context 和 `newMessages` 中，最后通过事件和返回值交给上层。它不调用 `SessionStorage.appendEntry()`，也不理解 JSONL、fork、leaf 或 compaction entry。

### Harness 的桥接过程

```text
1. Harness 从 session.buildContext() 读取当前 leaf 路径
                         |
                         v
2. createContext() 生成 AgentContext
                         |
                         v
3. agentLoop / Agent 开始运行
                         |
4. AgentEvent 被 Harness.handleAgentEvent() 接收
                         |
                         v
5. message_end -> session.appendMessage()
   turn_end    -> flush pending session writes() + save_point
                         |
                         v
6. 下一 turn 的 prepareNextTurn()
   重新 session.buildContext()
```

源码证据：`packages/agent/src/harness/agent-harness.ts:322-365,407-463,477-520`。

### 为什么每一轮要重新构建 Context？

因为 Harness 允许在 turn 之间改变：

- system prompt
- model / thinking level
- active tools
- session 中的 custom/compaction/branch 状态
- provider request options

`prepareNextTurn()` 会先 flush session writes，然后再次调用 `createTurnState()`，从最新 Session 生成下一次模型请求所用的 context。这意味着 session 是“事实来源”，Agent loop 的 context 是“本次请求快照”。

## 3. 消息到底保存在哪里？

同一条消息会经历三个不同层次：

```text
LLM response
  -> AgentState.messages              当前运行内存
  -> AgentEvent.message_end            生命周期事件
  -> Session.appendMessage()           持久化 SessionTreeEntry
  -> Session.buildContext()            下一轮恢复为 AgentMessage[]
```

所以：

- `AgentState.messages` 是运行时快照。
- `SessionTreeEntry` 是可恢复、可分支的历史记录。
- `AgentContext.messages` 是发送给模型前的上下文输入，可能经过 compaction 和 transform。

这三者通常内容相关，但不是同一个数组，也不承担同一个生命周期。

## 4. `coding-agent` 的 `AgentSession` 又是什么？

`packages/coding-agent/src/core/agent-session.ts` 是 coding-agent 自己的应用层 session facade，不应和 `packages/agent/src/harness/agent-harness.ts` 当成同一个类。它做的事情相似：

- 持有底层 `Agent`。
- 订阅 AgentEvent。
- 在 `message_end` 写入 `SessionManager`。
- 把工具、扩展、compaction、retry 和 UI 事件接到 Agent 上。

差别是：

```text
pi-agent-core AgentHarness
  通用、可嵌入、提供抽象 ExecutionEnv 和 Session

pi-coding-agent AgentSession
  Pi coding agent 的业务实现，接入 SessionManager、TUI、扩展、bash/read/edit/write
```

它们都采用“Runtime 发事件，上层订阅并持久化”的模式，但上层能力和具体存储实现不同。

## 5. 最简单的心智模型

把它类比成数据库应用：

```text
Agent loop = 事务执行器
AgentState = 当前事务内存
Session = 事件/历史数据库
Harness = ORM + 事务协调器 + 生命周期 middleware
AgentTool = 注入的业务函数
```

Runtime 决定“如何执行一个 tool call”；Harness 决定“这个 tool 从哪里来、是否允许、结果何时写入 session、下一轮用什么上下文”。
