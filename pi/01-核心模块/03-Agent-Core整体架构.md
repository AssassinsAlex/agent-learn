# Agent Core 整体架构映射

基线：`pi-earendil` commit `864b35c462f9623579b068e9cab848419f9e1d0f`。

`@earendil-works/pi-agent-core` 不是单个 Agent 类，而是一个从低层循环到可持久化 agent harness 的分层包。理解它时要先区分“核心运行时”和“上层能力”。

## 1. 总体地图

```text
┌──────────────────────────────────────────────────────────────┐
│ Application / coding-agent                                  │
│ AgentSession、UI、扩展、coding tools、认证和业务策略           │
└──────────────────────────────┬───────────────────────────────┘
                               │ Agent API / events / hooks
┌──────────────────────────────v───────────────────────────────┐
│ Harness                                                     │
│ AgentHarness、prompt/skill/system prompt、compaction、branch │
│ summary、session tree、JSONL/memory repo、ExecutionEnv       │
└──────────────────────────────┬───────────────────────────────┘
                               │ Agent / AgentLoop contracts
┌──────────────────────────────v───────────────────────────────┐
│ Runtime                                                     │
│ Agent、agentLoop、AgentState、AgentContext、AgentEvent       │
│ streamAssistantResponse、tool execution、steer/follow-up     │
└──────────────────────────────┬───────────────────────────────┘
                               │ streamFunction(Context)
┌──────────────────────────────v───────────────────────────────┐
│ pi-ai                                                       │
│ Message、Model、Provider、AssistantMessageEventStream       │
└──────────────────────────────┬───────────────────────────────┘
                               v
                    Anthropic / OpenAI / Google / ...
```

入口出口总览在 `packages/agent/src/index.ts`：它导出 runtime、loop、harness、compaction、session、skills、system-prompt、proxy 和 types。

## 2. Runtime 层：最小可复用 Agent

### 核心文件

| 文件 | 角色 |
| --- | --- |
| `src/types.ts` | 契约层：`AgentMessage`、`AgentState`、`AgentContext`、`AgentTool`、`AgentEvent`、`AgentLoopConfig` |
| `src/agent.ts` | 有状态 facade：状态、队列、订阅、prompt/continue/abort、事件归约 |
| `src/agent-loop.ts` | 无 UI 的控制循环：模型流、工具 batch、turn 和停止条件 |
| `src/proxy.ts` | 将远程代理的 SSE/JSON 事件恢复成 `AssistantMessageEventStream` |
| `src/node.ts` | Node 入口，额外导出 Node 文件/执行环境 |

### 三个核心数据对象

```text
AgentState
  systemPrompt
  model / thinkingLevel
  tools
  messages (AgentMessage[])
  isStreaming / streamingMessage / pendingToolCalls / errorMessage

AgentContext
  systemPrompt
  messages
  tools

AgentLoopConfig
  model
  transformContext?
  convertToLlm
  streamFunction
  getApiKey?
  beforeToolCall? / afterToolCall?
  prepareNextTurn? / shouldStopAfterTurn?
  getSteeringMessages? / getFollowUpMessages?
  toolExecution
```

`AgentState` 是长期运行状态；`AgentContext` 是某次 loop 请求上下文；`AgentLoopConfig` 是本次运行的策略和依赖注入。三者不要混为一个“大对象”。

## 3. Runtime 的调用关系

```text
Agent.prompt()
  -> runPromptMessages()
  -> agentLoop()
       -> runLoop()
            -> streamAssistantResponse()
                 -> transformContext()
                 -> convertToLlm()
                 -> streamFunction(model, Context)
            -> executeToolCalls()
                 -> validate args
                 -> beforeToolCall
                 -> AgentTool.execute
                 -> afterToolCall
            -> append tool results
            -> next turn / steering / follow-up
  -> processEvents()
       -> reduce AgentState
       -> await subscribers
```

重要边界：`streamFunction` 不直接接收 `AgentMessage[]`，它只接收 `pi-ai` 的 `Context`；因此 `transformContext` 和 `convertToLlm` 是 agent 扩展自定义消息、裁剪上下文和注入外部信息的正式插槽。

## 4. Harness 层：从 runtime 到完整 agent

`src/harness/agent-harness.ts` 是更高层的编排器。它把 runtime 连接到：

- session storage 和 session tree
- compaction / branch summarization
- prompt templates 和 skills
- system prompt 构建
- 文件系统和 shell 执行环境
- provider 请求生命周期 hooks
- retry、save point、settled、abort 等 harness 事件

因此，低层 `Agent` 可以脱离文件系统运行；Harness 才通过 `ExecutionEnv` 获得读文件、写文件、列目录、创建临时目录和执行 shell 的能力。

### Harness 的环境抽象

`src/harness/types.ts` 定义 backend-independent 能力：

```text
ExecutionEnv
├── FileSystem
│   ├── read/write/append
│   ├── fileInfo/listDir/exists
│   ├── canonicalPath
│   └── temp file/dir
└── Shell
    └── exec(stdout/stderr/exitCode/timeout/abort)
```

所有失败通过 `Result<T, Error>` 和稳定错误码返回，而不是让具体 Node backend 的异常泄漏到 harness API。`src/node.ts` 提供 `NodeExecutionEnv`。

## 5. Session 层：消息不是唯一状态

Session 使用树状 entry，而非简单的 `Message[]`：

```text
SessionTreeEntry
├── message
├── thinking_level_change
├── model_change
├── active_tools_change
├── compaction
├── branch_summary
├── custom / custom_message
├── label / session_info
└── leaf
```

核心接口：

- `SessionStorage`：创建 entry id、追加 entry、读取 entry、查找类型、计算 stats、取得 root 到 leaf 的路径。
- `SessionRepo`：create/open/list/delete/fork。
- `Session`：在 storage 之上提供 session tree 导航和上下文构建。
- `JsonlSessionStorage`：默认的 append-only JSONL 实现。
- `MemorySessionStorage`：测试和无持久化场景。

### 为什么是树

树结构支持 resume、fork、branch、切换 leaf 和从某个历史节点继续。`parentId` 描述历史关系，`leaf` 记录当前活动指针；因此“当前对话”是从 leaf 向 root 计算出来的路径，而不是文件中最后几行的简单切片。

## 6. Compaction 层：上下文预算管理

`src/harness/compaction/compaction.ts` 不属于低层 loop，它是 session-aware 的上下文压缩器：

```text
Session path entries
  -> estimateContextTokens()
  -> findCutPoint()
  -> messagesToSummarize + retainedTail
  -> LLM structured summary
  -> CompactionEntry
  -> 后续 context 从 summary + retainedTail 恢复
```

它还处理 split turn、previous summary、文件操作提取和 usage 统计。`branch-summarization.ts` 则针对从历史分支切出新路径的摘要。

## 7. Prompt/Skill/System Prompt 层

这些模块属于“模型可见上下文的构建”，但不属于 agent loop 本身：

- `harness/messages.ts`：消息构造和转换辅助。
- `harness/system-prompt.ts`：把基础指令、环境信息、skills 等组合成 system prompt。
- `harness/skills.ts`：加载并格式化 `SKILL.md` 能力。
- `harness/prompt-templates.ts`：显式命令/模板展开。

它们最终通过 `AgentContext.systemPrompt`、user message 或 `convertToLlm` 进入模型请求。

## 8. 事件映射

```text
pi-ai AssistantMessageEvent
  start / text_delta / thinking_delta / toolcall_delta / done
             |
             v
agent-core AgentEvent
  message_start / message_update / message_end
  tool_execution_start / update / end
  turn_start / turn_end / agent_start / agent_end
             |
             v
Harness events
  context / before_provider_request / tool_call / save_point
  settled / abort / compaction / branch_summary
             |
             v
Application UI / session persistence / extensions
```

低层事件描述“运行时发生了什么”；Harness 事件描述“应用编排需要在哪些生命周期点介入”。

## 9. 作为学习地图的阅读顺序

```text
第 1 层  types.ts
         先掌握对象和事件契约
第 2 层  agent.ts
         再掌握状态、队列和事件归约
第 3 层  agent-loop.ts
         追踪模型流和工具循环
第 4 层  session/*.ts
         理解树状历史和持久化
第 5 层  compaction/*.ts
         理解上下文窗口如何被压缩
第 6 层  agent-harness.ts + harness/types.ts
         理解完整应用编排和环境能力
第 7 层  coding-agent AgentSession
         看 Pi 如何把通用 core 变成 coding agent
```

## 10. 边界结论

1. **Agent core runtime 不拥有具体文件工具**；工具通过 `AgentTool` 注入。
2. **Agent loop 不负责 session 持久化**；它只发事件和返回消息。
3. **Session 不等于 AgentState**；Session 是可分支的历史存储，AgentState 是当前运行快照。
4. **Compaction 不等于普通 prompt 裁剪**；它会读取 session tree、调用模型生成摘要并写入结构化 entry。
5. **`pi-coding-agent` 是 core 的应用层消费者**，通过 `AgentSession` 把模型、工具、扩展、TUI 和持久化接起来。
