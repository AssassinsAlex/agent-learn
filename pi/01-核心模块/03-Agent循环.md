# Agent 循环

第一讲已整理为：[Agent Core 第一讲：Agent、Session 与 Agent Loop](03-Agent循环-第一讲.md)

## 关注点

- 单轮开始与停止条件
- 流式模型响应的事件类型
- tool call 的解析、执行顺序和结果回填
- 取消、重试、限流、模型错误和工具错误

## 最小状态机

```text
Idle -> Preparing -> Streaming -> ExecutingTools -> Preparing
                           |              |
                           v              v
                        Completed        Failed / Cancelled
```

> 状态名称为学习占位符，需按源码中的真实对象和事件替换。

## 源码证据

- `packages/agent/src/agent-loop.ts:154-274`
- `packages/agent/src/agent-loop.ts:276-356`
- `packages/agent/src/agent-loop.ts:407-551`
- `packages/agent/src/agent.ts:164-374`
- `packages/agent/src/types.ts:AgentLoopConfig, AgentEvent, AgentState`
