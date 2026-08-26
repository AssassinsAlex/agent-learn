# DeepSeek Harness 学习档案

本目录用于系统学习 DeepSeek Harness（`dsh`）：一个以 Cordis 为运行时、采用“一切皆插件”架构的 Agent Harness。学习记录沿用 Agent Learn 的统一结构：先建立全景图，再按模块导览，随后用源码链路和实验验证设计结论。

## 学习导航

| 顺序 | 文档 | 目标 |
| --- | --- | --- |
| 0 | [项目档案](00-总览/项目档案.md) | 固定源码版本、运行环境和学习边界 |
| 0.1 | [学习路线](00-总览/学习路线.md) | 明确阅读顺序和每个阶段的问题 |
| 1 | [整体架构](00-总览/整体架构.md) | 建立插件树、请求闭环和能力 seam 的全景图 |
| 2 | [核心模块索引](01-核心模块/README.md) | 导览入口、运行时、Agent、Session、LLM、Tools、Compaction 和宿主层 |
| 3 | [源码阅读索引](02-源码阅读/README.md) | 将架构结论关联到文件、函数和事件 |
| 4 | [主链路追踪](02-源码阅读/主链路追踪.md) | 跟踪一次输入从 inbox 到模型、工具和持久化 |
| 5 | [Compaction 源码导览](02-源码阅读/Compaction源码导览.md) | 深入上下文压力、裁剪、摘要、替换和 overflow recovery |
| 5.1 | [TokenMeter 估算算法](02-源码阅读/TokenMeter估算算法.md) | 手算 block、message、header、surface delta 和 provider usage anchor |
| 6 | [实验索引](03-实验与验证/README.md) | 设计最小可复现实验 |
| 7 | [问题清单](04-问题与复盘/问题清单.md) | 记录未验证的设计问题和下一阶段任务 |

当前重点：[Compaction 全景](01-核心模块/04-Compaction全景.md)。

## 我重点关注的上下文设计

### 1. Surface：模型当前看到什么

```text
append-only Session log
  -> surface projection
  -> deriveMessages()
  -> 当前模型请求
```

`surface` 是完整 Session 日志的当前可见投影，不是完整历史。压缩或裁剪通过追加 replacement 节点 shadow 旧节点；旧事件仍保留在日志中，但不再进入当前模型上下文和 `surfaceTokens`。

### 2. Tool-result pruning：先便宜减压，再决定是否摘要

只有 pressure 或 provider overflow 已经触发后，才运行可选的 tool-result pruner。默认规则：

```text
超过 8192 个 Unicode code point 的文本
  -> 保留前 4096
  -> 插入 "[... tool result middle pruned ...]"
  -> 保留后 1024
```

它按整个 tool result 的 text 内容累计长度，非文本 block 保持相对位置；每个结果通过 `compaction/prune` + `tool/result(surfaceOp=replace)` 持久化。裁剪后重新测量：如果已经低于压力阈值，就跳过摘要模型。详细说明见 [Compaction 全景](01-核心模块/04-Compaction全景.md)。

### 3. Token 估算：启发式 node price，不是精确 tokenizer

`TokenMeter` 使用固定启发式：

```text
文本/reasoning block = ceil(text.length / 4) + 4
message              = content blocks 总价 + 4
tool-result           = 内部 content 总价 + 4
system                = ceil(system.length / 4) + 4
tools                 = ceil(JSON.stringify(tools).length / 4) + 4
```

它不是每次全量重算：每个 Session 增量 replay 新事件，维护当前 surface 每个 message node 的 token price 和 `surfaceTokens`；replacement 使用：

```text
surfaceTokens += replacementTokens - shadowedTokens
```

但 `measure()` 为返回稳定快照会复制当前 surface，因此快照构造是 `O(surface)`。完整算例见 [TokenMeter 估算算法](02-源码阅读/TokenMeter估算算法.md)。

### 4. Provider usage anchor：校准总体，不分配到 node

如果最近成功请求的 canonical request envelope 仍然匹配，且 provider usage 满足保守下界，meter 会使用：

```text
当前 totalTokens
  = provider usage anchor
  + (当前 surfaceTokens - anchor surfaceTokens)
```

因此 provider 的真实 usage 用于校准整体压力，本地 surface token 估算用于追踪之后新增、裁剪和压缩带来的变化。provider usage 不会被反向分摊到各个 surface node；压缩区域选择仍依赖 node-level heuristic price。

### 5. 压缩区域：保留尾部，压缩头部

从 surface 尾部向前累加启发式 token，直到达到 `retainTokens`；该尾部保留原样，之前的连续 head 作为压缩区域。边界还必须满足完整的 tool-call/tool-result 配对和已结束 step，不能任意按字符截断。

## 一句话心智模型

```text
Cordis plugin tree
  -> Agent inbox
  -> Session append-only event log
  -> current surface projection
  -> system prompt + tools + deriveMessages()
  -> LLM stream
  -> assistant/tool events
  -> token pressure
  -> prune or compact surface
  -> next request
```

## 记录原则

- 源码事实、架构推断和实验结论分开记录。
- 关键结论必须能回链到源码文件或测试。
- 注意区分 Cordis 的 runtime `ctx`、Agent 的运行时上下文和模型真正看到的 context。
- Compaction 重点记录“原始日志保留什么”和“当前 surface 给模型看到什么”的差异。
- 本项目处于 developer preview，版本演进可能带来兼容性变化。

## 当前状态

- 源码位置：`C:\workspace\open_source_agent\deepseek-harness`
- 学习基线：`0.1.1-rc.2` / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`
- 运行要求：Node `^22.19.0 || >=24.0.0`，pnpm workspace，TypeScript ESM
- 已完成：整体架构预览、请求闭环、Session surface、上下文管理、Compaction、tool-result pruning、TokenMeter 估算模型
- 下一阶段：补运行实验，验证自动压力压缩、工具结果裁剪、摘要失败和 overflow recovery
