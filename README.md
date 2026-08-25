# Agent Learn

面向开源 Agent 实现的源码学习档案。每个项目以独立目录沉淀，按“总览 -> 核心模块 -> 源码追踪 -> 实验与复盘”的结构组织。

## 项目索引

| 项目 | 当前重点 | 入口 | 源码基线 |
| --- | --- | --- | --- |
| Pi | 已深入 `agent-core`；下一层为 CLI Runtime、`pi-ai`、extensions/trust | [Pi 学习档案](pi/README.md) · [覆盖审查](pi/00-总览/全项目学习覆盖审查.md) | [`pi` commit `864b35c`](https://github.com/earendil-works/pi/tree/864b35c462f9623579b068e9cab848419f9e1d0f) |

## 统一约定

- 每个项目目录都有自己的 `README.md`，作为该项目的学习导航。
- 项目内 Markdown 跳转使用相对路径，以保证整体上传 GitHub 后仍可用。
- 外部源码证据使用固定 commit 的 GitHub permalink，避免上游演进导致笔记与代码错位。
- 结论应区分源码事实、设计推断和待验证问题。

## 当前目录

```text
agent-learn/
├── README.md
└── pi/
    ├── README.md
    ├── 00-总览/
    ├── 01-核心模块/
    ├── 02-源码阅读/
    ├── 03-实验与验证/
    └── 04-问题与复盘/
```
