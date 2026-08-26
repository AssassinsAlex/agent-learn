# 核心模块索引

每篇模块笔记都按“职责 -> 输入输出 -> 状态 -> 正常/异常流程 -> 源码证据 -> 待验证问题”组织。

## 模块树

```text
DeepSeek Harness
├── 启动与组合
│   ├── apps/cli
│   ├── packages/boot
│   └── packages/bundle
├── Agent Runtime
│   ├── core/agent
│   ├── core/agent-loop
│   └── core/scope
├── Durable State
│   └── core/session
├── Prompt and Tools
│   ├── core/system-prompt
│   └── core/tools
├── Model Boundary
│   ├── llm/llm
│   ├── llm/llm-deepseek
│   └── llm/token-meter
├── Context Management
│   └── compaction/*
└── Host and Product
    ├── host/*
    ├── apps/web
    └── apps/cli
```

## 模块导览

| 模块 | 重点问题 | 源码入口 | 状态 |
| --- | --- | --- | --- |
| 启动与组合 | profile、bundle、patch 如何生成插件树？ | `apps/cli/src/bin.ts`、`packages/boot` | 导览 |
| Agent Loop | turn、step、inbox、tool loop、cancel 如何协作？ | `packages/core/agent-loop/src/agent.ts` | 导览 |
| Session | event log、surface、deriveMessages 如何协作？ | `packages/core/session/src/index.ts`、`surface.ts` | 已完成第一版 |
| System Prompt | section 和 tool schema 如何合并？ | `packages/core/system-prompt/src/index.ts` | 导览 |
| Tools | registry、scope、approval、execution pipeline 如何连接？ | `packages/core/tools` | 导览 |
| LLM | stream、message、adapter、DeepSeek route 如何归一化？ | `packages/llm/llm`、`llm-deepseek` | 导览 |
| Token Meter | 如何估算当前请求压力？ | `packages/llm/token-meter/src/index.ts` | 已完成第一版 |
| Compaction | 如何裁剪、摘要、替换并恢复 overflow？ | `packages/compaction` | 重点 |
| Host/API | Session event 如何投影到 Web/RPC？ | `packages/host`、`apps/web` | 提纲 |

## 阅读顺序

```text
01 运行时与插件系统
  -> 02 Agent 循环与 Session
    -> 03 上下文管理
      -> 04 Compaction 全景
        -> 05 模型、工具与安全边界
```
