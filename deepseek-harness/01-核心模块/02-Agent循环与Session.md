# Agent 循环与 Session

## Agent Loop

`ReactLoopAgent` 维护 Agent 的 live phase：`idle`、`maintenance`、`running`。它还维护 inbox、AbortController、当前 turn/step 和 Agent-scoped Cordis context。

主要 API：

| API | 语义 |
| --- | --- |
| `followup()` | 放入下一 turn 并唤醒 |
| `steer()` | 放入当前 turn 的下一 step 并唤醒 |
| `inject()` | 放入下一 step，但不唤醒 |
| `cancel()` | 清空或保留 inbox，并 abort 当前活动 |
| `runMaintenance()` | 在 idle 状态下执行不会发普通模型 turn 的维护任务 |

## Session 的双重角色

Session 同时承担：

- append-only durable event log；
- 当前模型可见 surface 的投影。

原始事件用于恢复、UI、fork、transcript 和 telemetry；surface 用于生成下一次请求。

## 请求边界

每个 step 会：

1. 从 inbox claim 输入。
2. 组装 system prompt 和 tools。
3. 通过 `agent/pre-step` 接受、拒绝或重写消息。
4. 把最终输入追加为 `user/message`。
5. 用 `session.deriveMessages()` 生成历史。
6. 记录 request header/context。
7. 调用 `llm.stream()`。

## 工具循环

assistant message 中的 tool calls 会进入工具流水线，工具结果追加为 `tool/result`，然后放入 `next-step`。如果没有 live tool call 或新 steering，turn 才会结束。

## 源码入口

- `packages/core/agent-loop/src/agent.ts`
- `packages/core/agent/src/inbox.ts`
- `packages/core/session/src/index.ts`
- `packages/core/session/src/surface.ts`

## 关键判断

模型看到什么不是由一个内存 `messages[]` 决定，而是由“已记录事件 + surface projection + 当前 prompt assembly”共同决定。
