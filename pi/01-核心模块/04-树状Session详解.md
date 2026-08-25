# 树状 Session 详解

基线：`pi-earendil` commit `864b35c462f9623579b068e9cab848419f9e1d0f`。

## 1. 这是 Pi 首创的吗？

不能把它说成 Pi 首创。树状历史是一个广泛使用的工程思想：版本控制、编辑器 undo tree、对话分支、事件溯源系统都需要“一个共同历史 + 多个后续分支”。

Pi 的贡献在于把这个思想具体化为一套适合 agent 的持久化协议：每个 entry 有 `id`、`parentId`、`timestamp`；当前活动位置由持久化的 `leaf` entry 指向；消息、模型变化、工具变化、压缩摘要和分支摘要进入同一个 entry log；当前模型上下文由 root 到 leaf 的活动路径重新计算。

准确说法是：**树状 Session 不是 Pi 发明的通用概念，但 Pi 在 agent 场景中提供了具体、完整、可持久化的实现。**

## 2. 一个最小树

```text
root
└── U1: 用户：实现登录功能
    └── A1: 助手：分析方案
        └── U2: 用户：先写数据库层
            └── A2: 助手：修改了 schema
                ├── U3a: 用户：继续写 API
                │   └── A3a: 助手：完成 API
                │       └── leaf -> A3a
                └── U3b: 用户：先补测试
                    └── A3b: 助手：完成测试
                        └── leaf -> A3b
```

`A2` 是共同祖先。回到 `A2` 后，不必删除 `U3a/A3a`；把活动 leaf 改到 `A2`，再追加新消息即可形成 `U3b/A3b`。

## 3. Entry 的含义

| entry 类型 | 作用 |
| --- | --- |
| `message` | user、assistant、toolResult 消息 |
| `model_change` | 记录模型切换，作用于该分支后续 |
| `thinking_level_change` | 记录思考级别切换 |
| `active_tools_change` | 记录当前分支启用的工具名 |
| `compaction` | 用摘要替代旧历史，同时保留切点/尾部信息 |
| `branch_summary` | 从另一条分支返回时，为被离开的分支留下摘要 |
| `custom` / `custom_message` | 应用扩展状态或自定义消息 |
| `label` / `session_info` | UI 标签和会话元信息 |
| `leaf` | 持久化当前活动树指针 |

`Session.appendMessage()` 使用当前 leaf 作为新 entry 的 `parentId`；`setLeafId()` 不是内存游标，而是追加 durable `leaf` entry。因此重启后仍能恢复当前分支。

源码：`packages/agent/src/harness/session/session.ts:214-227,338-357`；entry 类型在 `packages/agent/src/harness/types.ts`。

## 4. 当前上下文如何从树得到

```text
当前 leaf
   -> 沿 parentId 向上到 root/compaction
   -> 反转为 root -> leaf
   -> defaultContextEntryTransform()
   -> sessionEntryToContextMessages()
   -> SessionContext { messages, model, thinkingLevel, activeToolNames }
```

模型看到的不是整个 session 文件，而是当前 leaf 对应的 active branch。`Session.buildContext()` 从路径推导模型、thinking level、active tools，再把 entry 投影为 `AgentMessage[]`。

源码：`packages/agent/src/harness/session/session.ts:39-147,179-190`。

## 5. 回到旧消息时发生什么

### 同一 session 内导航

`moveTo(targetId)` 会把 leaf 改到目标节点，持久化 leaf 变化，并可选追加 `branch_summary`。旧分支仍留在树上，但不属于当前 active path；下一次 prompt 从新 leaf 重新构建 context。

### 新 session fork

coding-agent 还可以从历史 session 创建一个新 session 文件。新文件与原 session 后续独立，适合保存一条独立探索路线。同一 session 的 tree navigation 则适合在同一历史中来回切换。

## 6. Branch summary 与 Compaction

```text
共同祖先
  ├── 当前主线
  └── 被离开的探索分支
       └── branch_summary: 目标、完成内容、文件变化、下一步
```

- `compaction`：压缩当前分支较早历史，解决上下文窗口问题。
- `branch_summary`：离开/返回另一分支时保存工作上下文，解决分支切换的信息丢失。

两者都可能生成特殊 `AgentMessage`，但语义不同。

## 7. 为什么不用线性数组

树结构支持重新尝试、保留多个实验路径、回到共同祖先、对分支单独压缩，以及重启后恢复当前分支。代价是需要维护 parent/leaf 一致性、分支摘要、上下文路径和崩溃恢复。

## 8. 和 Agent Loop 的边界

```text
Session tree
  -> buildContext()
  -> AgentContext.messages
  -> agentLoop 执行
  -> AgentEvent.message_end
  -> appendMessage() 创建新树节点
  -> 下一轮重新 buildContext()
```

Agent loop 不遍历树、不选择 leaf、不创建 fork；树的读取、写入和导航属于 Harness/Session 层。

## 9. 后续阅读问题

1. 新 entry 的 `parentId` 从哪里取得？
2. leaf 什么时候变化，是否持久化？
3. `buildContext()` 遇到 compaction 时保留什么？
4. model/thinking/tools 如何沿 branch 继承？
5. fork 是复制文件、复制 entry，还是建立 parent session 引用？
6. 崩溃发生在 summary、leaf 或 message 写入之间时如何恢复？
