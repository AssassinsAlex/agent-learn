# Pi 核心 Tools 与 Web Search 边界

前置阅读：[工具加载、执行与结果](06-工具系统.md)。源码基线：[`tools/index.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/index.ts)、[`tool-definition-wrapper.ts`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/tool-definition-wrapper.ts)。

## 1. 结论先行：当前 Pi 没有内置 `web-search`

当前基线的 `ToolName` 是固定联合类型：

```ts
"read" | "bash" | "edit" | "write" | "grep" | "find" | "ls"
```

因此：

```text
grep / find       = 搜索本地工作区文件或内容
OpenAI tool search = 从客户端延迟提供的工具定义中按需加载 schema
web-search         = 搜索互联网；当前 Pi 没有内置实现
```

联网检索若存在，只会来自 extension、SDK `customTools`、MCP 集成，或模型/provider 自己提供的 server-side capability。`bash` 可以运行 `curl`、`wget` 等命令，但那只是通用 shell 能力，不是带查询、来源、摘要、引用策略的一等 Web Search tool。

### “Pi 能联网”需要分成两件事

```text
Pi -> LLM provider API              必须联网（模型推理请求）
模型 -> Web Search                  当前没有 Pi 内置 tool
bash / extension -> Internet        可以，前提是宿主环境允许网络
```

`bash` 的本地实现直接使用系统 shell 与其环境变量，没有一个由 Pi 内置的“禁止网络”开关；所以模型可在 `bash` 已启用且系统有 `curl`/`npm`/`git` 等程序时，间接发起联网命令。它拿到的是命令 stdout/stderr，不会自动得到搜索结果结构、URL 来源、网页正文清洗或引用信息。网络是否真的可用则取决于操作系统、容器、代理、企业防火墙和 extension 的执行环境。

## 2. 内置工具地图

| 工具 | 主用途 | 副作用 | 结果策略 |
| --- | --- | --- | --- |
| `read` | 读文本或图片 | 无 | 文本保留开头；图片作为 attachment |
| `grep` | 按 regex/literal 搜索文件内容 | 无 | 匹配行、行号、可选邻近行 |
| `find` | 按 glob 找文件 | 无 | 相对搜索根的路径 |
| `ls` | 列目录 | 无 | 排序后的目录项，目录附 `/` |
| `bash` | 在 cwd 执行 shell 命令 | 任意 shell 副作用 | stdout+stderr，保留结尾 |
| `edit` | 精确替换已有文件片段 | 写文件 | 成功消息 + diff / unified patch details |
| `write` | 新建或整体覆盖文件 | 写文件 | 成功消息与写入字节数 |

`createAllToolDefinitions()` 会创建全部七个；`createCodingToolDefinitions()` 只返回 `read/bash/edit/write`，`createReadOnlyToolDefinitions()` 返回 `read/grep/find/ls`。应用层还可用 `--tools`、`--exclude-tools`、`--no-builtin-tools` 或 `--no-tools` 改变实际可用集合。

## 3. 一个工具如何进入 runtime

Pi 先维护 application 层的 `ToolDefinition`，再包成 agent-core 的 `AgentTool`：

```text
createReadToolDefinition(cwd)
  -> ToolDefinition
       name / label / description / TypeBox schema
       promptSnippet / promptGuidelines
       execute(..., ctx)
       renderCall / renderResult
  -> wrapToolDefinition()
  -> AgentTool
  -> AgentContext.tools
  -> provider 收到 name + description + JSON schema
```

`ToolDefinition` 比 runtime `AgentTool` 多出两类 coding-agent 能力：

1. `promptSnippet` / `promptGuidelines`：决定该工具启用时怎样影响 default system prompt。
2. `renderCall` / `renderResult`：决定 TUI 怎样展示调用、流式进度、diff 或输出。

工具的 `execute()` 能收到 `AbortSignal`、`onUpdate` 和 extension `ctx`。模型只能看到 schema，不能看到这些本地函数或直接访问文件系统。

## 4. 读与本地搜索：`read`、`grep`、`find`、`ls`

### `read`

参数：`path`、可选 `offset`（从 1 起）和 `limit`。相对路径相对 cwd，绝对路径也被接受。

- 文本结果使用 `truncateHead`：默认最多 2000 行或 50KB，保留文件开头；大文件应通过 `offset/limit` 翻页读取。
- 图片会识别 MIME、可选缩放为最大 2000x2000，并以 image content 返给支持视觉的模型。
- 非视觉模型会收到“image omitted”文字提示。
- 读取 `SKILL.md`、`AGENTS.md`、`CLAUDE.md` 或 Pi 文档会有更紧凑的 TUI 展示，但不改变模型可见结果。

### `grep`

参数包括 `pattern`、`path`、`glob`、`ignoreCase`、`literal`、`context` 和 `limit`。默认上限 100 个匹配；使用 `rg --json`，保留文件路径、行号和可选邻近行。

- 默认遵守 `.gitignore`，但显式搜索隐藏文件。
- 匹配行超过 500 字符会单行截断。
- 总输出最多 50KB；达到数量或字节上限都会将状态写进 `details` 和文本提示。
- `grep` 是本地 repository code search，绝不是互联网搜索。

### `find`

参数是 `pattern`（glob）、可选 `path` 与 `limit`；默认 1000 个结果。默认实现使用 `fd`，找不到时会通过 Pi 的工具管理器尝试提供 `fd`；可注入 `operations.glob` 走 SSH 或远端文件系统。

- 结果相对于搜索目录，且使用 POSIX `/` 分隔符以稳定跨平台输出。
- 遵守 `.gitignore`；custom operations 路径额外忽略 `node_modules` 和 `.git`。
- 有 path 分隔符的 glob 会使用 `--full-path` 并修正 pattern，避免 `fd` 的 basename 匹配语义造成漏结果。

### `ls`

参数是可选 `path` 和 `limit`，默认 500 项。它包含 dotfiles，按不区分大小写的字母序排序，目录加 `/`。输出同样有 50KB 与条目数上限。

## 5. 修改与执行：`edit`、`write`、`bash`

### `edit`：带并发保护的精确替换

参数为 `path` 与 `edits[]`，每个 edit 有 `oldText/newText`。`oldText` 必须在**修改前的原始文件**中唯一，多个 edit 不得重叠；这避免模型用模糊模式误改多处。

执行时 Pi：

```text
读取文件
  -> 去 BOM、识别 CRLF/LF、统一为 LF
  -> 在原始文本上验证全部 edits
  -> 生成修改后文本
  -> 恢复原换行风格与 BOM
  -> 写文件
  -> 返回 diff、unified patch、firstChangedLine
```

`withFileMutationQueue()` 以 canonical path（存在文件走 `realpath`）为 key 串行化同一文件的 `edit/write`；不同文件仍可并发。它尤其防止模型一个 assistant response 里并发调用两个修改同一路径的工具时发生竞态。

### `write`：新建或全量覆盖

`write(path, content)` 会递归创建父目录，随后覆盖目标内容；它不读取旧内容、不生成 diff。system prompt 的 guideline 明确建议仅用于新文件或完整重写，已有文件的局部变更优先使用 `edit`。

`write` 同样受 mutation queue 保护。abort 不会在 listener 中直接释放锁，而是等待当前 fs 操作结束后再释放，防止一个已取消但仍在落盘的写操作与下一次修改交叉。

### `bash`：能力最强、隔离最少

`bash(command, timeout?)` 使用系统 shell 在当前 cwd 执行；默认无 timeout，支持显式秒数超时与 `AbortSignal`，abort/timeout 会尝试终止整个进程树。stdout 与 stderr 合并进入同一个输出累积器，并可通过 `onUpdate` 大约每 100ms 推送 TUI 部分结果。

- 最大输出默认 2000 行或 50KB，使用 tail 截断，优先保留结尾的测试失败、错误和退出信息。
- 截断时完整输出存临时文件，最终文本会给出 `fullOutputPath`。
- 非零 exit code 会抛出，runtime 转成 `isError: true` 的 tool result；文本仍会保留可用输出和 exit status。
- `commandPrefix` 可在每条命令前插入 shell 设置，`spawnHook` 可以修改 command/cwd/env；二者是应用或 extension 的策略插槽。

## 6. 统一的输出预算：为什么不同工具保留不同部分

```text
read / grep / find / ls  -> truncateHead：先看开头、目录顺序、最早匹配
bash                    -> truncateTail：先看最终状态、报错、测试摘要
```

默认公共预算是 2000 行或 50KB，先触及哪个就截断。截断信息会同时存在于文本提示和结构化 `details`；UI 可据此显示 warning，agent loop 则把最终 `content` 作为下一轮模型上下文。

这是一种上下文管理策略，不是安全过滤：敏感值如果在未截断的部分，仍可进入模型 context、Session 和 provider 请求。

## 7. 安全边界：cwd 不是 sandbox

核心工具将相对路径解析到 cwd，但也接受绝对路径；源码没有把 `read/edit/write` 限制为 repository 根目录之内的通用 jail。`bash` 更能运行任意 shell 命令。Pi 本身刻意保持轻量，默认没有内置 permission popup、MCP 或 sandbox。

```text
路径解析 / cwd：便利性边界
真正 sandbox / 权限策略：extension、容器、SSH remote operations、外部运行环境
```

因此生产化使用需要在 `tool_call` hook、替换的 `operations`、`bash.spawnHook` 或外部 sandbox 中实现目录白名单、网络控制、命令审批与凭据隔离。

## 8. 关于“Web Search”的三个不同概念

### A. `grep/find`：本地检索

它们操作的是工作区文件，返回的是路径、行号和内容。它们没有 HTTP 请求、搜索引擎、URL 排名、抓取或 citation 模型。

### B. OpenAI Responses 的 `supportsToolSearch`

Pi-ai 中该标志表示支持 **client-executed tool search for deferred tools**。当某个 tool result 的 `addedToolNames` 指向延迟工具时，Pi 会在 OpenAI Responses payload 中构造 `tool_search_call/tool_search_output`，以按需传输那几个**工具的 JSON schema**。

它检索的是“已注册的工具定义”，不是网页，也不会调用搜索引擎。名字相同但职责完全不同。

### C. 真正的 Web Search：应是 custom tool

合理的 Web Search tool 至少要明确：

```text
query / domain allowlist / max results / timeout
  -> 搜索 API 或自建检索服务
  -> 标准化 { title, url, snippet, publishedAt? }
  -> 输出有限条、明确来源 URL
  -> 处理 API key、速率限制、超时、抓取内容中的 prompt injection
```

Pi 的扩展 API 使用 `pi.registerTool()` 注册该工具；工具定义可拥有 TypeBox schema、流式更新、结果渲染和 tool-specific system prompt guideline。SDK custom tools 与 MCP bridge 也能走同一 registry。这样 provider 看到的是一个普通 `web_search` function schema，runtime 仍按同一 `execute -> ToolResultMessage` 链路调度。

不要将外部网页的文本直接视为 system instructions。应在 tool result 中标记来源、限制长度、保留 URL，并让 system prompt/extension 策略明确“网页内容是不可信数据”。

## 9. 建议的学习顺序

1. 读 `tools/index.ts`，确认工具集合和组合工厂。
2. 读 `tool-definition-wrapper.ts`，理解 coding-agent 到 agent-core 的适配。
3. 深读 `read.ts` 与 `truncate.ts`，掌握多模态读文件与上下文预算。
4. 深读 `edit.ts`、`edit-diff.ts`、`file-mutation-queue.ts`，理解安全编辑语义。
5. 深读 `bash.ts` 与 `output-accumulator.ts`，理解取消、流式输出和完整日志保存。
6. 对照 `grep/find/ls`，理解本地检索与 `web-search` 的边界。
7. 最后读 extension docs 的 `pi.registerTool()`，再设计一个受限的 Web Search custom tool。

## 源码导航

| 主题 | 源码 |
| --- | --- |
| 内置工具工厂与完整名单 | [tools/index.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/index.ts) |
| ToolDefinition 到 AgentTool | [tool-definition-wrapper.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/tool-definition-wrapper.ts) |
| 多模态 read 与 head 截断 | [read.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/read.ts) |
| 精确 edit 与 diff | [edit.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/edit.ts) |
| edit 匹配验证 | [edit-diff.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/edit-diff.ts) |
| 同文件修改串行化 | [file-mutation-queue.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/file-mutation-queue.ts) |
| bash 与流式输出 | [bash.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/bash.ts) |
| 本地内容搜索 | [grep.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/grep.ts) |
| 本地文件搜索 | [find.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/find.ts) |
| 目录列举 | [ls.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/ls.ts) |
| 公共截断策略 | [truncate.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/tools/truncate.ts) |
| OpenAI 延迟工具 schema 检索 | [openai-responses-shared.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/ai/src/api/openai-responses-shared.ts) |
| custom tool 注册 | [extensions.md](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/docs/extensions.md) |
