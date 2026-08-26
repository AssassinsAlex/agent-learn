# 实验与验证

## 实验原则

优先使用已有 Vitest fixtures 和 mock LLM，不依赖真实 API key。每个实验记录源码 commit、配置、输入 Session events、预期 surface、实际模型请求和最终事件序列。

## Compaction 实验清单

| 编号 | 实验 | 目标 |
| --- | --- | --- |
| C-01 | 小 context window 压力触发 | 验证 `thresholdRatio` 和 `retainRatio` |
| C-02 | 超长 tool result | 验证 prune 后跳过摘要的路径 |
| C-03 | prune 后仍超阈值 | 验证摘要请求和 checkpoint replacement |
| C-04 | tool-call/result 边界 | 验证 range 不拆 pair |
| C-05 | 摘要期间追加输入 | 验证 surface stability rejection |
| C-06 | 摘要模型失败 | 验证原 surface 保留和错误事件 |
| C-07 | provider context overflow | 验证 overflow retry 和 generation proof |
| C-08 | manual compactNow | 验证 idle admission、flush 和 `turn:null` |
| C-09 | compaction close 失败 | 验证 unmatched start 的诊断行为 |
| C-10 | 多 provider/model policy | 验证 exact target override |

## 现有测试入口

- `packages/compaction/compaction/tests/compaction.spec.ts`
- `packages/compaction/compaction/tests/tool-pairing.spec.ts`
- `packages/compaction/compaction-tool-result-pruner/tests/tool-result-pruner.spec.ts`
- `packages/core/session/tests/derived-cache.spec.ts`
- `packages/core/agent-loop/tests/request-error.spec.ts`
- `packages/llm/token-meter/tests/token-meter.spec.ts`

## 建议输出

实验不要只看最终回答；至少同时保存：

```text
model request messages
session event types and seqs
surface.nodes before/after
token measurement before/after
retry count
```
