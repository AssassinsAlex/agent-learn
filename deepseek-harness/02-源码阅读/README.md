# 源码阅读索引

## 推荐顺序

1. `docs/architecture.md`：理解术语和事件域。
2. `packages/core/agent-loop/src/agent.ts`：进入实际 driver。
3. `packages/core/session/src/index.ts`、`surface.ts`：理解 durable history 和 projection。
4. `packages/core/system-prompt/src/index.ts`：理解 prompt sections 和 tools assembly。
5. `packages/llm/llm/src/index.ts`、`packages/llm/llm-deepseek/src/adapter.ts`：理解模型边界。
6. `packages/llm/token-meter/src/index.ts`、`surface-fold.ts`：理解 pressure。
7. `packages/compaction/compaction/src/index.ts`：先读 seam contract。
8. `packages/compaction/compaction-basic/src/index.ts`：读 listener 和策略。
9. `packages/compaction/compaction-basic/src/region.ts`：读事务、稳定性和 replacement。
10. `packages/compaction/compaction-tool-result-pruner/src/index.ts`：读无模型裁剪。

11. `TokenMeter估算算法.md`：用完整算例理解 block/message/header 计价、增量 replay、replacement delta 和 provider usage anchor。

## 阅读方法

对每个函数记录四项：

```text
谁调用它
它读取哪些 durable/live state
它追加或替换哪些 event
失败后谁负责 retry / preserve / report
```
