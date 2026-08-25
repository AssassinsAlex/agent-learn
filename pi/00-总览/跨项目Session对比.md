# 跨项目 Session 对比

调研范围：`C:\workspace\open_source_agent` 下当前可读的 `pi-earendil`、`deepseek-harness` 和 `codex-cli` 源码/文档。

## 结论先行

Pi 的树状 Session 不是 Agent 项目普遍默认的实现，也不能宣称是 Pi 首创的通用思想。但在本次对比范围内，Pi 的特点非常明确：

> **Pi 把“同一个持久化 session 内的多分支历史导航”建模为一等能力，用 entry 的 `parentId` 和持久化 `leaf` 指针恢复 active branch。**

DeepSeek Harness 和 Codex 都有 append-only history、resume 和 fork，但它们的主要模型不同：

| 项目 | 基本持久化模型 | 分支方式 | 当前上下文来源 |
| --- | --- | --- | --- |
| Pi | `SessionTreeEntry` 树状 entry log | 同一 session 内 `moveTo`/leaf 导航；也支持新 session fork | root -> active leaf 路径，经 compaction transform 投影 |
| DeepSeek Harness | `SessionEvent` 追加日志 | `ctx.sessions.fork(source, boundary, childSessionId)` 创建另一 session；没有 Pi 式 entry leaf 导航证据 | `deriveMessages()` 从当前 session event log 派生 |
| Codex | thread/rollout JSONL 日志 | `thread/fork` 创建独立 thread/rollout，记录 `forked_from_id`；subagent 另有 `parent_thread_id` | 当前 thread/rollout 的事件历史 |

## DeepSeek Harness

源码文档明确把 Session 定义为：

- append-only `SessionEvent` log
- 单一事实来源
- LLM message history 从 log 派生
- `turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*` 都是可持久化事件
- fork/resume/transcript/persistence 都从这条事件流派生

它的架构重点是“事件日志 + 插件事件域”：

```text
SessionEvent log
  -> deriveMessages()
  -> Agent request
  -> session/event replay
```

DeepSeek 的 fork API 存在，但语义是 fork 一个 session；本次扫描没有看到 Pi 那种每条 entry 通过 `parentId` 连接、并用同一 session 的 `leaf` 指针切换 active branch 的模型。

证据：

- `deepseek-harness/docs/architecture.md` 的 Session log 与 Fork 说明
- `deepseek-harness/docs/subsystems/session.md` 的 `SessionEventMap`
- `deepseek-harness/docs/architecture.md` 的 `ctx.sessions.fork(...)`

## Codex

Codex 的持久化对象主要是 thread/rollout：

```text
Thread A
  -> rollout-A.jsonl

thread/fork(Thread A)
  -> Thread B
  -> rollout-B.jsonl
  -> B.forked_from_id = A
```

Codex 的源码和测试显示：

- rollout 以 JSONL 保存 session meta、response item 和 event message。
- `thread/fork` 返回一个不同 ID 的 thread。
- fork 后的 rollout 只包含指定边界以前的历史，不继续写入原 thread 的文件。
- `forked_from_id` 记录来源 thread。
- `parent_thread_id` 主要用于 subagent/thread lineage，不等于 Pi 的每条消息 parentId。

因此 Codex 也有“会话谱系”，但它更像：

```text
Thread graph / lineage
  A -> B (forked thread)
```

而不是 Pi 的：

```text
One session tree
  entry -> entry -> entry
             ├── branch A
             └── branch B
```

证据：

- `codex-cli/codex-rs/app-server/tests/suite/v2/thread_fork.rs`
- `codex-cli/codex-rs/app-server/tests/common/rollout.rs`
- `codex-cli/sdk/python/src/openai_codex/generated/v2_all.py` 的 `ThreadForkParams`/`Thread` 字段

## Pi 的亮点如何表述

建议在学习总览中使用以下表述：

> **Pi 的一个架构亮点是 Session Tree：它不把 fork 仅实现为复制出一个新 conversation，而是在同一 durable session 中保留共同祖先和多个后续分支，并用 `parentId + leaf` 计算当前 active branch。这样可以在不删除历史的情况下回到旧节点、重新尝试方案，并对分支执行 compaction 或 branch summary。**

不要使用：

> Pi 首创了树状 Session。

因为树状历史本身是广泛存在的设计思想，当前对比只能证明 Pi 的具体实现与 DeepSeek/Codex 不同，不能证明行业首创。

## 后续学习问题

1. Pi 的 `SessionManager.forkFrom` 与 `Session.moveTo` 在持久化边界上分别做什么？
2. DeepSeek 的 fork session 是否复制完整事件前缀，还是使用共享引用？
3. Codex 的 `forked_from_id` 是否只用于 lineage/元数据，还是也参与上下文恢复？
4. 三者在 compaction 后如何保证恢复上下文与原始日志一致？
