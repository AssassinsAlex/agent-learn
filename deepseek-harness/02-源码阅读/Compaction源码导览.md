# Compaction 源码导览

## 推荐阅读顺序

### 1. 先看能力契约

从 `packages/compaction/compaction/src/index.ts` 开始，确认 `CompactionEngine` 的两个入口：

- `compactIfNeeded(agent, trigger, signal)`：自动 pressure/overflow 路径；
- `compactNow(agent, signal, sourceCommandId?)`：idle maintenance 路径。

同目录的类型和事件定义说明 `CompactionResult`、`compaction/start`、`compaction/summary`、`compaction/end`、`compaction/prune` 如何组成可追踪记录。

### 2. 再看自动编排

读 `compaction-basic/src/index.ts` 的 `BasicCompactionEngine`：

```text
agent/pre-step
  -> compactIfNeeded(agent, pressure)
  -> resolveModelInfo(contextWindow)
  -> measure / threshold
  -> prune / remeasure
  -> compactRegion / retry

agent/request-error
  -> failure.code === CONTEXT_WINDOW_EXCEEDED
  -> prune / select range / compact
  -> replaceGeneration advanced ? retry : preserve error
```

重点不要把两个触发器混为一谈：pressure 需要容量和阈值，overflow 使用 provider 已确认的事实；前者以低于阈值为成功标准，后者以 surface replacement generation 前进为 retry 证明。

### 3. 看策略解析

读 `compaction-basic/src/config.ts`：默认 `thresholdRatio=0.8`、`retainRatio=0.16`、摘要 `maxTokens=8192`、两类 retry 默认 1。`retainRatio` 与 `retainTokens` 互斥；ratio retention 必须小于 threshold；`modelPolicies` 按精确 provider/model 匹配，不依赖 `listModels()`。

### 4. 看范围选择

读 `compaction-basic/src/region.ts` 的 `selectCompactableRange()`：它从 surface 尾部向前累计 token，保留近期 tail，head 作为压缩候选，再通过 `toolPairingBalancedBefore/After` 修正边界。

要验证的问题：

- 空 surface、只有一个不可分节点时是否返回 `null`；
- open step 或不平衡 tool-call/result 是否被拒绝；
- 超大的单个 tool step 是否先交给 pruner；
- checkpoint 是否可以在后续周期再次进入候选范围。

### 5. 看事务与稳定性

仍在 `region.ts`，追踪 `compactSurfaceRegion()`：

```text
assert entry state
validate range
append compaction/start
measure + build summarization input
await summarize
assert stability
append summary + replace message
append compaction/end
flush (manual)
```

自动路径使用 `whole-surface` 检查；手动路径使用 `selected-span` 检查。任一提交前失败都会尝试追加错误的 `compaction/end`；close 本身失败则留下活动 marker。

### 6. 看 pruner

读 `compaction-tool-result-pruner/src/index.ts`：先 snapshot 当前 surface 的 `tool/result`，逐项按 code point 计算文本长度，追加 `compaction/prune` shadow-price，再追加只替换 `content` 的 `tool/result`。注意：同一 pass 前面已经成功的 replacement 不会因后续异常回滚。

### 7. 最后看摘要器与计量

`compaction-basic/src/summarizer.ts` 负责复用原 system/tools/region messages 并追加固定指令；`llm/token-meter/src/index.ts` 负责 request envelope、surface node 和 replacement 的统一价格。摘要输出只有文本进入 checkpoint，不能把辅助调用的 reasoning/tool call 当成主会话事件。

## Token-meter 重点证据链

不要把 `measure()` 理解为每次从整个 Session 日志重新估算。推荐按下面顺序读：

```text
TokenMeter.states: WeakMap<Session, ReplayState>
  -> _sync(): 只消费 consumedEvents 之后的新日志
  -> _foldEvent(): 更新 header、step、surface 和 usage anchor
  -> foldSurfaceTokens(): 对 append/replacement 做 token delta
  -> measure(): 组合 baseline + surfaceDelta，并复制 nodes 快照
```

对应源码：

- `packages/llm/token-meter/src/index.ts:79`：每 Session 的 replay state。
- `packages/llm/token-meter/src/index.ts:159`：增量同步日志。
- `packages/llm/token-meter/src/surface-fold.ts:42`：surface 节点的增量计价。
- `packages/llm/token-meter/src/index.ts:116`：measurement 返回与 `O(surface)` 快照复制。
- `packages/llm/token-meter/src/estimate.ts:26`：固定启发式估算。

重点观察三个层次：

```text
日志 replay 是增量的
当前 surface node price 是长期维护的
measure 返回的 nodes 快照需要复制当前 surface
```

## 固定启发式估算表

| 对象 | 估算方式 |
| --- | --- |
| text/reasoning block | `ceil(text.length / 4) + 4` |
| tool-call | name 与 arguments 各按 4 字符/token，再加 block overhead |
| tool-result | 递归估算内部 content，再加 block overhead |
| message | content 总价 + role overhead `4` |
| system | 文本按 4 字符/token + role overhead |
| tools | `JSON.stringify(tools)` 按 4 字符/token + block overhead |

这是压缩门控和预算选择用的近似价格，不是 provider 精确 tokenizer 的计费结果。成功请求的 provider usage 只有在 canonical request envelope 完全匹配、并且 usage 不低于完整 heuristic anchor 时才会复用。

## 最小证据链

```text
tokenMeter.measure(session)
  -> resolveCompactSpec()
  -> selectCompactableRange()
  -> compactSurfaceRegion()
  -> buildSummarizationInput()
  -> summarizeWithLlm()
  -> compaction/summary
  -> user/message(surfaceOp=replace)
  -> compaction/end
  -> session.deriveMessages()
```

## 建议的实验矩阵

| 场景 | 观察点 |
| --- | --- |
| pressure 恰低于阈值 | 不剪枝、不摘要 |
| 大 tool result 使 pressure 达阈值 | pruner 先落盘；重新计量后可能跳过摘要 |
| 摘要输出不小于源范围 | 不提交 replacement，记录失败 close |
| 摘要期间追加消息 | 自动 whole-surface 失败；手动 selected-span 可接受范围外追加 |
| provider 返回 overflow | 不依赖 capacity，replacement generation 前进后 retry |
| 无可压缩范围 | 保留原请求错误或返回 null，不任意截断 |
| idle 期间 `/compact` | maintenance、standalone marker、flush |
