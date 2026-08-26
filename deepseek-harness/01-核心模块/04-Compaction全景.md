# Compaction 全景

## 结论先行

DeepSeek Harness 的上下文压缩，是对 `Session.surface` 做可回放的持久化替换：旧事件不删除，模型下一次请求只从新的 surface 派生消息。

```text
Session events (append-only, complete history)
        |
        +--> surface projection (current model-visible history)
                    |
                    +--> tokenMeter.measure()
                    +--> optional tool-result pruning
                    +--> select compactable range
                    +--> summary model
                    +--> surface replace with checkpoint
```

这里有四个职责边界：`tokenMeter` 负责计量，`compaction-basic` 负责策略和摘要事务，`tool-result-pruner` 负责低成本裁剪，Session 负责日志与 surface 的一致性。

## 一、压缩的整体设计

### 1. 能力拆分

```text
dsh-compaction
  Service Definition、CompactionResult、compaction/* 事件
dsh-compaction-basic
  压力策略、保留预算、摘要、溢出恢复、事务编排
dsh-compaction-tool-result-pruner
  不调用模型的 tool/result 头中尾裁剪
dsh-command-compact
  /compact 命令，调用 compactNow()
```

这使摘要器可以替换，而 Session replacement、配对边界、计量和失败语义仍由基础实现统一保证。

### 2. 为什么不是删除历史

一次成功压缩大致追加：

```text
compaction/start
compaction/summary
user/message(surfaceOp=replace, start..end)
compaction/end
```

`compaction/summary` 保存摘要、原范围、原 surface seq 和 token 数；replacement 节点带 `sourceEventSeqs`。因此完整原文仍可审计、回放和诊断，模型只看到 checkpoint 加上未被替换的近期内容。

### 3. 压缩对象是什么

压缩单位是 surface node，不是字符区间。选择范围必须是连续的，并且不能拆开 assistant 的 tool-call 与对应 tool-result，也不能包含尚未闭合的 step。轮次边界不是天然保护边界，旧 turn 中的旧 step 仍可能进入压缩范围。

## 二、压缩的触发时机

### 1. 正常压力触发

自动 listener 挂在 `agent/pre-step`，即下一次模型请求前的步骤边界。它先用 `ctx.tokenMeter.measure(session)` 计量最新 durable request envelope、system、tools 和当前 surface。

对模型上下文容量 `W`，默认配置为：

```text
thresholdRatio = 0.8
retainRatio = 0.16
maxTokens = 8192
compactionRetries = 1
auto = true
```

```text
thresholdTokens = floor(W * thresholdRatio)
retainTokens = floor(W * retainRatio)
```

只有 `totalTokens >= thresholdTokens` 才进入压缩；低于阈值时不会剪枝，也不会调用摘要模型。`W` 来自当前 durable routed provider/model 的 adapter model info，而不是模型发现列表。

### 压力数字是如何得到的

压力由单例 `ctx.tokenMeter` 提供。它对每个 Session 维护一个 replay fold，增量消费新的 SessionEvent，并维护当前 surface 每个 message node 的启发式 token 值和总量。追加事件只计算新增 message；replacement 则用：

```text
surfaceTokens += replacementTokens - shadowedTokens
```

所以压缩后的 token 压力可以立即下降。`measure()` 返回前会复制整个当前 surface，形成不可变快照，因此不是每次重放全部日志，但 measurement 快照本身是 `O(surface)`。

这个估算器使用固定启发式而非 provider tokenizer：文本约为每 4 个字符一个 token，另加 message、content block、tool-call、system 和 tools 的结构开销。provider usage 只有在 canonical request envelope 完全匹配且满足保守下界时才建立 anchor；路由或请求 envelope 改变后会重新使用启发式估算。

### 2. provider 确认的 overflow

`agent/request-error` 收到规范化的 `CONTEXT_WINDOW_EXCEEDED` 时，说明实际请求已经被 provider 拒绝。该路径：

- 不要求先有可解析的 context capacity；
- 不等待达到正常 pressure threshold；
- 先做可选 tool-result pruning，再选择可压缩范围；
- 只有 `surface.replaceGeneration` 前进，才允许 retry；
- 由 `maxOverflowRetries` 限制次数，默认 1。

如果没有产生替换、已取消、或错误不是规范 overflow，则保留原始 request error。

### 3. 手动压缩

`compactNow()` 通过 `agent.runMaintenance()` 执行，要求 Agent idle 且没有排队的唤醒工作。它选择一个有用的范围，使用 `turn: null` 的 standalone transaction，提交后 flush Session。自动路径要求整个 surface 稳定；手动路径只要求被选 span 稳定，因此摘要期间 span 外追加的新内容可以保留。

## 三、实际压缩流程

```text
1. 读取 durable route，解析目标策略
2. tokenMeter.measure()
3. 若是 pressure：检查 totalTokens 是否达到 threshold
4. 若有 pruner：扫描当前 surface 的 tool/result 并追加 replacement
5. 重新 measure；若已低于 threshold，结束（不摘要）
6. 从 tail 向前保留 retainTokens，选出 head
7. 修正边界，保证 tool-call/result 配对和 step 完整
8. 追加 compaction/start，形成持久化 lock
9. 回放 system、tools 和选中区域消息，调用摘要模型
10. 检查摘要 framed token 是否小于被替换内容
11. 重新检查 surface 稳定性
12. 追加 summary、checkpoint replacement 和 compaction/end
13. 重新 measure；仍超阈值时按 compactionRetries 重复
```

### 摘要请求如何构造

摘要调用是独立的一次 `ctx.llm.stream()`，用途标记为 `purpose: compaction`，不经过仅服务于 Agent loop 的 `agent/request` waterfall。输入包含原请求的 system、tools，以及被压缩区域的 derived messages，最后追加固定 compaction instruction。

只有摘要文本写回 checkpoint；摘要模型的 reasoning 和 tool call 不进入主会话。摘要必须真正小于被替换范围，否则事务失败，避免“压缩后反而变大”。

### checkpoint 的模型语义

replacement 的 user message 使用 `<compacted-summary>...</compacted-summary>` 包裹摘要，并告诉后续模型把它当作已建立的背景继续工作。后续压缩可以再次替换这个 checkpoint；它不是永久不可压缩节点。

## 四、tool-result pruning

这是摘要前的无模型减压阶段，只在已经满足 pressure 或 overflow 条件后运行。

默认按 Unicode code point 统计文本字符：

```text
thresholdChars = 8192
headChars = 4096
tailChars = 1024
```

超过阈值的 `tool/result` 被改写为：

```text
head + "\\n\\n[... tool result middle pruned ...]\\n\\n" + tail
```

非文本 block 保持相对顺序，原始 tool result 仍在日志中。裁剪预算是字符预算，不是 token 预算，所以必须重新走 token meter。若裁剪后已经低于压力阈值，整个压缩周期不需要摘要模型。

边界包括：不拆 UTF-16 surrogate pair，但可能拆 grapheme cluster；只按头尾保留，不理解中间内容的语义；配置要求 `head + marker + tail <= threshold`，且替换必须严格小于原结果。

## 五、并发、一致性与失败

### 1. 稳定性检查

摘要是异步调用，期间 surface 可能变化。自动压缩使用 `whole-surface` 稳定性策略并比较整个测量结果；任何新增或替换都使本次摘要不能提交。手动压缩使用 `selected-span`，只重验证原选中 span，span 外追加可接受。

### 2. 持久化锁

`compaction/start` 在摘要前写入，未匹配的 start 表示活动压缩。若 close 失败，故意留下这个 marker，后续操作会报告 busy，避免把不完整事务当成可继续状态。较新的 `session/end-seed` 可以证明旧生命周期的 marker 已过期。

### 3. 局部成功

pruner 的 replacement 已经落盘后，后续摘要失败不能回滚该 replacement；overflow recovery 可以据此用 `replaceGeneration` 证明已有可重试的持久进展。摘要失败时主 surface 保持最近一次已提交状态，并继续携带它请求或保留原始 provider error，取决于触发路径。

### 4. 收敛与不可压缩情况

可能无法压缩到阈值以下：system/tools/envelope 本身过大、剩余节点不可拆、工具配对仍不可裁剪，或摘要输出不比源范围小。此时不能任意截断消息；达到 retry 上限后报错。

## 六、源码阅读入口

- `packages/compaction/compaction-basic/src/index.ts`：两个自动触发器、策略解析、pruning 顺序和 retry。
- `packages/compaction/compaction-basic/src/config.ts`：ratio/absolute retention、model policy 和配置校验。
- `packages/compaction/compaction-basic/src/region.ts`：范围选择、事务、稳定性和 checkpoint replacement。
- `packages/compaction/compaction-basic/src/summarizer.ts`：摘要请求的 system/tools/messages 回放。
- `packages/compaction/compaction-tool-result-pruner/src/index.ts`：字符预算和 tool/result replacement。
- `packages/llm/token-meter/src/index.ts`：统一 token 计量与 shadow price。
- `packages/llm/token-meter/src/surface-fold.ts`：surface append/replacement 的增量 token fold。
- `packages/llm/token-meter/src/estimate.ts`：4 字符/token 与结构开销的固定启发式。
- `packages/core/session/src/surface.ts`：replacement 后模型可见历史的投影。

## 与 Pi 的关键区别

Pi 更强调 session tree 的分支和树状回放；DeepSeek Harness 更强调 append-only SessionEvent 加 surface replacement。两者都保留可恢复历史，但 Harness 把“模型当前看到什么”显式建模为可持久化的 surface，并把压缩当成一种带 start/summary/end 事件的 durable transaction。
