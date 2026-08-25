# Pi Agent 学习档案

本目录用于系统学习 Pi（面向终端与代码任务的个人 AI agent）项目。学习记录采用“总览 -> 模块 -> 源码证据 -> 实验”的结构：先建立整体心智模型，再逐一追踪具体实现，最后用可复现实验验证结论。

## 学习导航

| 顺序 | 文档 | 目标 |
| --- | --- | --- |
| 0 | [学习路线](00-总览/学习路线.md) | 明确范围、问题与学习顺序 |
| 1 | [整体架构](00-总览/整体架构.md) | 建立 agent 运行闭环的全景图 |
| 2 | [模块索引](01-核心模块/README.md) | 从总览进入各核心部件 |
| 3 | [源码阅读索引](02-源码阅读/README.md) | 将结论关联到具体文件和符号 |
| 4 | [实验索引](03-实验与验证/README.md) | 通过最小实验验证运行机制 |
| 5 | [问题清单](04-问题与复盘/问题清单.md) | 记录待确认问题与阶段复盘 |

当前导读：[Agent Core 总体架构](./01-核心模块/03-Agent-Core整体架构.md)

架构亮点：[跨项目 Session 对比](./00-总览/跨项目Session对比.md)

## 记录原则

- **事实与推断分开**：源码、日志和官方文档是事实；分析必须标记为推断。
- **结论可追溯**：每个关键结论都应回链到源码位置、命令输出或实验记录。
- **先主干后细节**：先理解一次请求如何变成一次可观察的 agent 循环，再深入单个模块。
- **版本必记**：Pi 的版本、提交号、运行时版本与配置会影响行为，所有实验均需记录。

## 当前状态

- 项目源码位置：`C:\\workspace\\open_source_agent\\pi-earendil`
- 当前版本/提交：`0.80.10` / `864b35c462f9623579b068e9cab848419f9e1d0f`
- 本地运行环境：Node `>=22.19.0`，npm workspace，TypeScript ESM
- 当前学习阶段：阶段 2，已完成 Agent Core 映射、Session Tree 与 Compaction 层第一版，确认 Session Tree 是 Pi 的显著实现亮点。

下一步：继续从 `packages/agent/src/harness/compaction/compaction.ts` 和 `packages/agent/src/harness/session/session.ts` 做源码实验，再回到 coding-agent 的自动 compaction 触发链路。
