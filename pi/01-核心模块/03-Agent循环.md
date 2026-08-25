# Agent Loop：决策状态机与轨迹语义

基线：[`pi` commit `864b35c`](https://github.com/earendil-works/pi/tree/864b35c462f9623579b068e9cab848419f9e1d0f)。本篇从算法角度补充[第一讲](03-Agent循环-第一讲.md)的运行链路；第一讲回答“代码如何串起来”，本篇回答“loop 在什么状态下做什么决策”。

## 1. 状态、动作、观测

把一次 Pi run 抽象为受 context budget 约束的交互过程：

```text
S_t = {systemPrompt, transcript, model, thinkingLevel, tools,
       steeringQueue, followUpQueue, abortSignal}

A_t = 模型输出的 assistant message
      = text / thinking / 0..n 个 toolCall

O_t = 每个 toolCall 的 ToolResultMessage
      = content（回注模型）+ details（UI/日志）+ isError
```

`AgentContext` 是发给本次 provider 请求的快照；`AgentState` 是 `Agent` 持有的可变运行快照。不要把 session persistence 当成 loop state：session 保存和恢复更长期的历史，loop 只在当前 run 内消费和追加 transcript。

## 2. 两层循环，而非简单 while(toolCall)

`agentLoop()` 用新 prompt 开始，`agentLoopContinue()` 则要求既有 context 末尾不是 assistant。它采用 outer + inner 两层：

```text
agent_start
  outer loop: agent 原本可以结束时，处理 follow-up
    inner loop: 当前工作链，处理 model -> tools -> steering
      turn_start
      取出并注入 steering（若有）
      stream assistant
      error / aborted ? -> turn_end -> agent_end
      tool calls ? -> 执行一个 tool batch，追加 observations
      turn_end
      prepareNextTurn（可替换下一 turn 的 context/model/config）
      shouldStopAfterTurn ? -> agent_end
      steeringQueue 有消息 ? -> 继续 inner loop
      有可继续的 tool batch ? -> 继续 inner loop
    followUpQueue 有消息 ? -> 加入 transcript，开始下一 outer iteration
    否则 -> agent_end
```

因此两个队列语义不同：

| 队列 | 什么时候进入下一请求 | 用途 |
| --- | --- | --- |
| steering | 当前 assistant turn 的工具处理完成后、agent 本来结束前 | 打断/引导正在工作的 run，但不跳过本 turn 工具。 |
| follow-up | 当前 inner loop 已自然耗尽时 | 将新消息排到 agent 已完成当前工作之后。 |

源码：[agent-loop.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/agent-loop.ts)、[agent.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/agent.ts)。

## 3. 每个 turn 的真实转移

| 阶段 | 输入状态 | 产生的事件 / 状态变化 | 决策含义 |
| --- | --- | --- | --- |
| `turn_start` | 有可请求的 context | 开始一次 provider turn | 一个 policy action 的开始，不等同于完整任务。 |
| stream | context、tools、model | `message_start/update/end` | provider stream 被收敛为一个 assistant action。 |
| model failure | `stopReason: error/aborted` | `turn_end`，退出 | 不是 tool observation；本 run 不会自然继续。 |
| tool batch | assistant 含完成的 `toolCall` | `tool_execution_start/update/end`、tool results | 把 action 变成可观察环境反馈。 |
| `turn_end` | assistant + 本 turn tool results | hook 可检视完整局部轨迹 | 此时才适合 context selection、routing 或评测打点。 |
| `prepareNextTurn` | 完整 turn snapshot | 可替换 context / model / thinking / config | Pi 显式预留的 per-turn policy intervention 点。 |
| stop / queue poll | 完整 turn | 终止、注入 steering 或 follow-up | 决定当前 run 是否继续。 |

`streamAssistantResponse()` 依次调用 `transformContext`、`convertToLlm`，构造 `pi-ai Context`，解析当次凭据并消费 `AssistantMessageEventStream`。这意味着 context compression、长期记忆 retrieval、跨模型消息修复都应放在这条边界或其上游，而不应侵入 tool executor。

## 4. 工具 batch 是 action 后的环境步

一个 assistant message 可含多个 tool call。Pi 的语义是：

1. 先收集本 message 的 tool calls；`stopReason: length` 时，arguments 可能不完整，全部转为失败 tool result 而不执行。
2. 若全局 `toolExecution` 为 sequential，或任一被调用 tool 声明 sequential，则顺序执行；否则 preflight 之后可并发执行。
3. `beforeToolCall` 能阻塞一个 action；`afterToolCall` 能覆盖 observation 或要求 terminate。
4. 并发工具的完成事件按完成时间出现，但写回给模型的 tool results 仍按 assistant 的原始调用顺序排列，保证 replay 稳定。
5. 仅当 batch 中的结果共同要求 terminate，loop 才不会因该 batch 继续。

这是 agent 算法常见的“并行 environment action、稳定 trajectory serialization”折衷。研究工具调用策略时，应分别记录 wall-clock completion order 和 transcript order，不能混为同一标签。

## 5. 终止条件不是单一 stop token

| 条件 | loop 行为 |
| --- | --- |
| assistant `error` 或 `aborted` | 结束当前 run。 |
| `shouldStopAfterTurn()` 返回 true | 在 tool batch 和 `prepareNextTurn` 后终止；此 hook 不应 throw。 |
| tool batch 的 terminate | 不再用该 batch 驱动下一 turn。 |
| 没有 tool、没有 steering、没有 follow-up | 自然结束，发出 `agent_end`。 |
| 用户/应用 abort signal | 传递到 stream/tool；最终走 aborted 事件和 run settlement。 |

`agent_end` 发出后，`Agent` 仍会等待 listeners 完成，再把 streaming 状态清空。因此在指标代码中，`agent_end` 是“不会再生成 loop event”，`waitForIdle()` 才是“所有持久化、UI、hook 后处理都结算完”。

## 6. `Agent` 是状态 reducer + 并发门

`agentLoop` 本身产出事件；`Agent` 才负责：

- 禁止两个 `prompt()` 并发 run；运行中使用 `steer()` / `followUp()`；
- 将事件归约进 `AgentState`，然后按订阅顺序 `await` listeners；
- 为当前 run 创建 abort controller；
- 暴露 `waitForIdle()`，使上层 `AgentSession` 在 event listener 后处理结束时获得正确生命周期。

这条“先 reduce、后 await listeners”的顺序很重要：listener 看到的是已经反映该事件的 state。扩展若改变未来行为，应通过 hooks / queue / `prepareNextTurn`，而不是异步地修改某个过期 event。

## 7. 算法扩展的正确插入点

| 目标 | 推荐位置 | 不应做什么 |
| --- | --- | --- |
| Context retrieval / compaction | `transformContext` 或 `prepareNextTurn` | 在工具执行中临时修改 transcript。 |
| 模型路由 / thinking-level 调整 | `prepareNextTurn` 返回配置更新 | 把 provider 专有逻辑塞进 loop 主体。 |
| Tool permission / action policy | `beforeToolCall` | 解析未完成的 `toolcall_delta` 后抢先执行。 |
| Tool observation refinement | `afterToolCall` / tool wrapper | 只靠 UI 文本推断最终工具结果。 |
| 长期经验写入 | `turn_end` 或 `agent_end` listener，异步且可失败 | 让经验抽取失败阻塞核心循环。 |
| 评测与 reward logging | `turn_end`、`agent_end`、session persistence | 只以最终自然语言回复评估过程。 |

## 8. 最小实验与指标

1. 构造一个 mock `StreamFunction`，依次产生 text、toolCall、done，记录 event 与 transcript 的差异。
2. 用两个耗时不同的 mock tools 验证并发完成顺序和 tool result 写回顺序。
3. 在工具执行期间调用 `steer()`，确认它在 batch 后进入下一请求而非取消当前工具。
4. 在 `prepareNextTurn` 切换 mock model 或压缩 context，记录下一 turn 的 `Context` 是否改变。
5. 对每个 turn 保存：输入 token、工具调用数/错误率、是否 steering/follow-up、stop reason、wall time、任务 reward；这是比较 routing、memory、compaction 的最小数据集。

## 9. 源码证据

- [agent-loop.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/agent-loop.ts)：主循环、模型 stream、工具 batch。
- [agent.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/agent.ts)：队列、并发门、状态归约、listeners、abort。
- [types.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/types.ts)：`AgentLoopConfig`、`AgentContext`、`AgentEvent`、`AgentTool` 契约。
