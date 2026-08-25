# 核心模块索引

每篇模块笔记应包含：职责、关键对象/函数、输入输出、状态、正常与异常流程、源码证据、待验证问题。

## 模块树

```text
Pi Coding Agent
├── 入口层
│   └── CLI / main.ts
│       └── [01-入口与CLI.md](01-入口与CLI.md) · 待开始
│
├── 应用编排层（coding-agent）
│   ├── AgentSession
│   │   ├── coding-agent SessionManager（Pi CLI 实际使用）
│   │   └── [04-会话与上下文.md](04-会话与上下文.md) · 仅有提纲；待补 CLI JSONL 与恢复
│   ├── 工具注册与 coding tools
│   │   └── [06-工具系统.md](06-工具系统.md) · 已完成第一版
│   │       └── [13-核心Tools与WebSearch边界.md](13-核心Tools与WebSearch边界.md) · 已完成第一版
│   ├── 扩展、命令与生命周期
│   │   └── [07-扩展与命令.md](07-扩展与命令.md) · 仅有提纲；P0 待开始
│   └── 配置、认证与信任边界
│       └── [08-配置与安全边界.md](08-配置与安全边界.md) · 仅有提纲；P0/P1 待开始
│
├── Agent Core Runtime（agent）
│   ├── AgentState / AgentContext / AgentEvent / AgentTool
│   │   └── [03-Agent-Core整体架构.md](03-Agent-Core整体架构.md) · 已完成第一版
│   ├── Agent（状态、队列、订阅、abort）
│   ├── agentLoop（turn、模型流、工具 batch、停止条件）
│   │   └── [03-Agent循环-第一讲.md](03-Agent循环-第一讲.md) · 已完成第一讲
│   │       └── [03-Agent循环.md](03-Agent循环.md) · 已完成决策状态机补充
│   ├── 通用 Harness
│   │   ├── Session / SessionStorage（通用实现）
│   │   ├── Session Tree / branch path
│   │   │   └── [04-树状Session详解.md](04-树状Session详解.md) · 已完成第一版
│   │   ├── Compaction / context projection
│   │   │   └── [09-Compaction层详解.md](09-Compaction层详解.md) · 已完成第一版
│   │   ├── AgentHarness（loop 与 session 的编排）
│   │   │   └── [10-AgentHarness详解.md](10-AgentHarness详解.md) · 已完成第一版
│   │   └── Prompt 改写与包装（跨 coding-agent / Harness）
│   │       └── [11-Prompt改写与包装.md](11-Prompt改写与包装.md) · 已完成第一版
│   │           └── [12-输入Prompt示例.md](12-输入Prompt示例.md) · 已完成第一版
│   └── Harness（Agent + Session + 环境 + hooks）
│
├── 模型层（pi-ai）
│   └── Provider / Model / streamFunction / Message
│       └── [05-模型与Provider.md](05-模型与Provider.md) · 已完成第一版（Message / Stream 归一化）
│
└── 交互层（tui / modes）
    └── TUI、Interactive、Print、RPC、流式渲染
        └── [02-交互与渲染.md](02-交互与渲染.md) · 仅有提纲；P1 待开始
```

## 模块导航

| 层级 | 模块 | 学习笔记 | 源码入口 | 状态 |
| --- | --- | --- | --- | --- |
| 入口层 | CLI / `main.ts` | [入口与 CLI](01-入口与CLI.md) | [main.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/main.ts) | 仅有提纲；P0 待开始 |
| 应用编排层 | `AgentSession` | [会话与上下文](04-会话与上下文.md) | [agent-session.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts) | 仅有提纲；P0 待开始 |
| 应用编排层 | `SessionManager`（coding-agent 实际使用） | [会话与上下文](04-会话与上下文.md) | [session-manager.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/session-manager.ts) | 仅有提纲；P1 待开始 |
| 应用编排层 | 工具注册与 coding tools | [工具加载、执行与结果](06-工具系统.md) | [agent-session.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts) | 已完成第一版 |
| 应用编排层 | 内置核心 Tools / Web Search 边界 | [核心 Tools 与 Web Search 边界](13-核心Tools与WebSearch边界.md) | [tools/index.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/index.ts) | 已完成第一版 |
| 应用编排层 | 扩展、命令与生命周期 | [扩展与命令](07-扩展与命令.md) | [extensions/index.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/extensions/index.ts) | 仅有提纲；P0 待开始 |
| 应用编排层 | 配置、认证与信任 | [配置与安全边界](08-配置与安全边界.md) | [config.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/config.ts) | 仅有提纲；P0/P1 待开始 |
| Agent Core | 状态与类型契约 | [Agent Core 总体架构](03-Agent-Core整体架构.md) | [types.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/types.ts) | 已完成第一版 |
| Agent Core | `Agent` | [Agent Core 总体架构](03-Agent-Core整体架构.md) | [agent.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/agent.ts) | 已完成第一版 |
| Agent Core | `agentLoop` | [Agent 循环第一讲](03-Agent循环-第一讲.md) · [决策状态机](03-Agent循环.md) | [agent-loop.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/agent-loop.ts) | 已完成两讲 |
| Agent Core Harness | 通用 `Session` / `SessionStorage` | [树状 Session 详解](04-树状Session详解.md) | [session.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/session/session.ts) | 已完成第一版 |
| Agent Core Harness | Compaction / context projection | [Compaction 层详解](09-Compaction层详解.md) | [compaction.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/compaction/compaction.ts) | 已完成第一版 |
| Agent Core Harness | `AgentHarness` | [AgentHarness详解](10-AgentHarness详解.md) | [agent-harness.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/agent-harness.ts) | 已完成第一版 |
| Coding-agent / Agent Core | Prompt 改写与包装 | [Prompt 改写与包装](11-Prompt改写与包装.md) | [agent-session.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts) | 已完成第一版 |
| Coding-agent / Agent Core | 输入 Prompt 示例 | [输入 Prompt 示例](12-输入Prompt示例.md) | [agent-session.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts) | 已完成第一版 |
| 模型层 | Provider / Model / stream | [模型与 Provider](05-模型与Provider.md) | [ai/index.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/ai/src/index.ts) | 已完成第一版：Message / Stream 归一化 |
| 交互层 | TUI / modes | [交互与渲染](02-交互与渲染.md) | [tui/index.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/tui/src/index.ts) | 仅有提纲；P1 待开始 |

## 关系说明

```text
main.ts
  └─ 创建/组装 → AgentSessionRuntime
                    └─ 持有/编排 → AgentSession
                                      └─ 持有/订阅 → Agent
                                                        └─ 调用 → agentLoop
                                                                    ├─ 调用 → pi-ai streamFunction
                                                                    └─ 调用 → AgentTool.execute

AgentSession
  ├─ 读取/写入 → Session / SessionStorage
  ├─ 注入 → tools / extensions / compaction / auth
  └─ 发出 → UI / RPC / print events
```

这里的“继承”主要是**能力和职责的层级关系**，不是 TypeScript `extends`：

- `AgentSession` 不继承 `Agent`，而是持有一个 `Agent` 并订阅它的事件。
- `Harness` 不继承 `agentLoop`，而是为 loop 构造 `AgentContext` 和 `AgentLoopConfig`。
- `AgentTool` 不继承 Runtime，而是由 Runtime 调度、由应用注入实现。
- `Session` 不继承 `AgentState`；Session 是持久化历史，AgentState 是当前运行快照。

## 阅读顺序

```text
01 入口与 CLI
  -> 03 Agent Core 总体架构
    -> 03 Agent 循环第一讲
  -> 04 coding-agent 会话与上下文
        -> Agent Core Harness / 树状 Session
          -> 09 Compaction 层
            -> 05 模型 Provider
            -> 06 工具系统
              -> 07 扩展与命令
                -> 02 交互与渲染
```
