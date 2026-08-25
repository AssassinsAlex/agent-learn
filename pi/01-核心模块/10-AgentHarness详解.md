# AgentHarness 详解

源码入口：[`agent-harness.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/agent-harness.ts)

## 1. 一句话定位

`AgentHarness` 是 agent-core 的应用编排层。它不重新实现 `agentLoop`，而是把 `Session`、本回合 `AgentContext`、`AgentLoopConfig`、模型请求、工具、提示词、持久化和生命周期事件组装成一个可供 CLI、RPC 或其他应用使用的 agent。

```text
Session（历史与树）
  + AgentContext（本回合上下文）
  + AgentLoopConfig（loop 策略、工具钩子、队列）
  + Models.streamSimple（provider 请求）
  + Tools / system prompt / skills
  + 持久化、错误、生命周期事件
```

`agentLoop` 解决“模型流和工具调用如何循环”，Harness 解决“一个产品级 agent 如何运行、保存、暂停和扩展”。

## 2. 成员映射

```text
AgentHarness
├── env: ExecutionEnv                 文件系统、shell 等执行能力
├── session: Session                  历史树与 context 投影
├── models: Models                    provider 请求
├── model / thinkingLevel             当前模型与推理级别
├── tools + activeToolNames           工具注册与启用集合
├── resources                         skills、promptTemplates
├── systemPrompt                      固定字符串或按回合计算的函数
├── steerQueue / followUpQueue / nextTurnQueue
│                                      运行中干预、后续回合、下次 prompt 预置
└── handlers                          AgentEvent 与 harness 钩子
```

构造器会校验工具名唯一且 active 工具存在，默认 `thinkingLevel = off`，steering/follow-up 模式为 `one-at-a-time`。

## 2.1 `ExecutionEnv`：Harness 的运行环境能力

`ExecutionEnv` 是 `FileSystem & Shell` 的接口，不是工具执行器：

```text
ExecutionEnv
├── FileSystem
│   ├── cwd、absolutePath、joinPath、canonicalPath
│   ├── readTextFile/readBinaryFile/readTextLines
│   ├── writeFile/appendFile、listDir、fileInfo、exists
│   ├── createDir/remove、createTempDir/createTempFile
│   └── cleanup
└── Shell
    ├── exec(command, { cwd, env, timeout, abortSignal, onStdout, onStderr })
    └── cleanup
```

它的目的有三个：

1. **平台抽象**：agent-core 不依赖 Node 的 `fs`、`child_process`；Node 场景由 `NodeExecutionEnv` 实现，也可以换成远端容器、浏览器受限文件系统或沙箱实现。
2. **统一错误与取消模型**：文件操作返回 `Result<T, FileError>`，shell 返回 `Result<..., ExecutionError>`；错误有稳定 code，例如 `permission_denied`、`timeout`、`aborted`，而不是把后端异常直接向上抛。
3. **给 harness 周边能力使用**：skill/prompt template 加载、动态 `systemPrompt` 函数，以及具体工具实现，都能通过同一环境访问文件或 shell。

边界尤其重要：`AgentHarness` 只是把 `env` 保存为 `readonly env`，并在计算型 `systemPrompt` 回调中传入；`agentLoop.executeToolCalls()` 不会自动调用 `env.exec()`，也不会把 `env` 隐式注入每一个工具。某个 `bash`、`read` 或自定义工具是否使用它、怎样施加权限与截断，属于该工具/应用的实现。

## 3. 一次 prompt 的完整链路

```text
prompt(text)
  ├─ idle 检查，phase -> turn
  ├─ createTurnState()
  │    ├─ session.buildContext()
  │    ├─ 读取 session metadata
  │    ├─ 解析 systemPrompt
  │    └─ 得到 activeTools、model、streamOptions
  ├─ before_agent_start hook
  │    └─ 可追加 messages 或替换 systemPrompt
  ├─ runAgentLoop(messages, context, loopConfig, handler, signal, streamFn)
  │    ├─ context hook：最后修改 AgentMessage[]
  │    ├─ convertToLlm：转换为 provider Message[]
  │    ├─ before_provider_request/payload hook
  │    ├─ models.streamSimple()
  │    ├─ tool_call hook：可阻止工具
  │    ├─ AgentTool.execute（由 runtime 执行）
  │    ├─ tool_result hook：可修补结果或 terminate
  │    └─ prepareNextTurn：保存并重建 context
  ├─ message_end -> session.appendMessage()
  ├─ turn_end -> flush pending writes -> save_point
  └─ agent_end -> phase idle -> settled
```

Harness 将“本次请求的上下文快照”交给 loop；loop 运行期间不会每一步重读整个 Session，进入下一 turn 时才由 `prepareNextTurn` 刷新。

## 4. Harness 如何配置 Runtime

| loop 插槽 | Harness 实现 | 作用 |
| --- | --- | --- |
| `model` / `reasoning` | 当前模型和 thinking level | 决定 provider 请求 |
| `convertToLlm` | `harness/messages.ts` | 将 compaction/custom/bash 等内部消息映射为 provider 消息 |
| `transformContext` | `context` hook | 在送模型前裁剪或追加消息 |
| `beforeToolCall` | `tool_call` hook | 审批/阻止工具调用 |
| `afterToolCall` | `tool_result` hook | 修改结果、标记错误或终止循环 |
| `prepareNextTurn` | flush + `createTurnState` | 工具回合后持久化并刷新上下文 |
| `getSteeringMessages` | steer 队列 | 将运行中用户干预注入下一轮 |
| `getFollowUpMessages` | follow-up 队列 | 当前任务完成后追加回合 |

所以 `executeToolCalls()` 的实际执行仍属于 runtime；Harness 只通过 `beforeToolCall/afterToolCall` 提供策略与外围能力，工具本身由应用注册并实现。

## 5. Session 与持久化边界

`handleAgentEvent()` 是 runtime 事件到 Session 的桥：

```text
message_end  -> 立即 appendMessage
turn_end     -> flush pending writes -> save_point
agent_end    -> flush -> settled
```

运行期间对模型、thinking level、active tools、custom entry 等变更放入 `pendingSessionWrites`，在 turn 边界顺序刷入。Harness 的 `phase`、队列和当前 model 是内存运行状态；Session 是可分支历史，二者不是同一个 session，也不存在继承关系。

## 6. 三种消息队列

| API | 可调用时机 | 注入位置 |
| --- | --- | --- |
| `steer()` | harness 非 idle | 当前 loop 的下一轮，按 `one-at-a-time/all` 消费 |
| `followUp()` | harness 非 idle | 当前任务完成后触发的后续 turn |
| `nextTurn()` | 任意时机 | 下一个 `prompt()` 开始时，与新 prompt 一起发送 |

队列消费前发 `queue_update`；hook 失败时消息放回队列。`abort()` 清空 steer/follow-up，取消 provider 请求并等待 harness 回到 idle。

## 7. Provider 生命周期

`createStreamFn()` 对 stream options 做回合快照，然后执行：

```text
before_provider_request
  -> 合并 headers/metadata、timeout、retry、transport
before_provider_payload
  -> 可改写实际 payload
models.streamSimple
  -> after_provider_response
```

这些钩子让应用可以做鉴权头、路由、审计和观测，而不污染 `agentLoop`。

## 8. 显式能力

- `prompt()`：普通用户回合。
- `skill(name)`：将 skill 格式化成显式调用文本后复用同一 turn 链路。
- `promptFromTemplate(name, args)`：模板展开后复用同一 turn 链路。
- `compact()`：仅允许 idle；读取当前 branch，生成 compaction entry，发出 `session_compact`。
- `navigateTree(targetId, options)`：切换 active leaf；可生成 branch summary，发出 `session_tree`。

这些是 Harness 的产品级操作，底层 loop 不知道 session tree、skill 或 compaction。

## 9. 错误与 phase

主要 phase 是 `idle`、`turn`、`compaction`、`branch_summary`。非法时机调用会得到稳定的 `AgentHarnessError`，例如运行中再次 `prompt()` 是 `busy`，非运行时 `steer()` 是 `invalid_state`。

loop 抛错时，Harness 构造 `stopReason = error/aborted` 的 assistant failure message，仍走 `message_end -> turn_end -> agent_end` 保存和事件流程；若失败报告也失败，则抛出包含两个原因的 `AggregateError`。

## 10. 与相邻层的边界

```text
Application / AgentSession
  └─ 拥有并配置 AgentHarness
       ├─ 拥有 Session（持久化历史）
       ├─ 调用 prompt/steer/compact
       └─ 订阅事件更新 UI/RPC

AgentHarness
  └─ 调用 agentLoop
       └─ 调用 AgentTool.execute 与 streamFunction
```

1. Harness 不是 UI，也不是 `AgentSession`，而是可复用的通用编排 API。
2. Harness 不负责具体工具实现，但负责注册工具并提供审批和结果修补钩子。
3. Harness 决定 Session 的读写时机；Session 负责树和 context 投影。
4. 自动触发 compaction、overflow recovery 等策略可能在上层 coding-agent `AgentSession`，不能反推为通用 Harness 的职责。

## 源码导航

| 主题 | 文件 |
| --- | --- |
| Harness 主类与 turn 编排 | [`agent-harness.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/agent-harness.ts) |
| 类型、事件、错误、环境 | [`types.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/types.ts) |
| 内部消息到 provider 消息 | [`messages.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/messages.ts) |
| Session/context 投影 | [`session.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/session/session.ts) |
| Loop 与工具执行 | [`agent-loop.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/agent-loop.ts) |
| Compaction | [`compaction.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts) |
