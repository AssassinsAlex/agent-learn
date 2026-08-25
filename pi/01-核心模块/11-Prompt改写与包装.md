# Prompt 改写与包装

源码入口：[agent-session.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts)、[system-prompt.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/system-prompt.ts)、[prompt-templates.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/prompt-templates.ts)。

## 1. 总结：Pi 不是单点 prompt rewriter

Pi 的“改写/包装”分散在进入 runtime 的不同边界，主要是确定性文本拼装、占位符替换和 extension hook 覆盖，**不会在普通用户输入前调用另一个 LLM 改写**。

```text
原始输入
  -> extension input（拦截、替换或直接处理）
  -> /skill:name 展开
  -> /template 参数替换
  -> user message
  -> before_agent_start（追加 custom message、覆盖 system prompt）
  -> context hook（替换完整历史）
  -> provider payload hook（替换协议 payload）
```

system prompt 则在输入进入前，已经由工具、项目上下文、skills 与配置拼装完成。

## 2. System Prompt：多来源的固定包装

`AgentSession._rebuildSystemPrompt()` 收集当前 active tools 的 prompt snippet/guideline，以及 ResourceLoader 资源后调用 `buildSystemPrompt()`：

```text
默认 coding-agent 基础说明
  + Available tools（当前启用且有 snippet 的工具）
  + Guidelines（随工具集合和扩展定义变化）
  + appendSystemPrompt（配置/资源附加文本）
  + <project_context>
      <project_instructions path="...">AGENTS.md 或 CLAUDE.md</...>
    </project_context>
  + <available_skills>（name/description/location，不含全文）
  + Current working directory
```

`customPrompt` 会替换默认基础说明，但 Pi 仍追加 `appendSystemPrompt`、项目 context、可见 skill 列表与 cwd。

ResourceLoader 搜索全局 agent 配置目录，以及从文件系统根到当前 `cwd` 的各级目录；每级优先 `AGENTS.md`，其次 `CLAUDE.md`。祖先文件按由外到内的顺序加入，离当前目录更近的项目指令排在后面。这只是 system 字符串拼接，不是 role 隔离。

## 3. Skill：默认登记，显式调用才内联全文

普通情况下，`formatSkillsForPrompt()` 只加入：

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

模型需要用 `read` 工具读取完整 `SKILL.md`；`disable-model-invocation: true` 的 skill 不进入该列表。

用户显式输入 `/skill:name 额外要求` 时，`_expandSkillCommand()` 读取 skill 文件、去 frontmatter，再改写为：

```xml
<skill name="name" location="absolute-path">
References are relative to skill-dir.

完整 SKILL.md 正文
</skill>

额外要求
```

这仍是一个 `role: user` message，文件读失败或 skill 不存在则保持原输入。

## 4. Prompt Template：slash command 到纯文本

`/review src/a.ts strict` 匹配 template 后，`expandPromptTemplate()` 会用简单 shell 风格单/双引号解析参数，并替换 `$1`、`$2`、`$@`、`$ARGUMENTS`、`${@:N}`、`${@:N:L}`。它以 template 正文完全替换原 slash command，没有额外 XML 包裹，也不保留原命令。

## 5. 用户输入的运行期改写顺序

`AgentSession.prompt()` 的顺序：

```text
1. `/...` 若为 extension command，交给 extension，自身不进模型
2. extension input event
     - handled：结束，根本不发模型
     - transform：替换 text/images
3. `/skill:` 展开
4. `/template` 展开
5. 运行中则按 steer/followUp 入队；空闲则继续
6. 创建 role:user message（expandedText + images）
7. 插入 pending nextTurn messages
8. extension before_agent_start
     - 追加 custom messages
     - 覆盖整个 systemPrompt
9. agent.prompt(messages)
```

`input` 看到原始输入；`before_agent_start` 看到已展开的 `expandedText`。extension 的 `pi.sendMessage()` 显式关闭 template expansion，避免二次展开。

## 6. 送模型前的最后两个插槽

### Context hook

runtime 每次 provider 调用前执行 `transformContext`。coding-agent 将它连接到 extension `context`：扩展依次接收并可替换完整 `AgentMessage[]`。这影响当前请求的模型上下文，但不会自动改写 Session 持久化历史。

### Provider payload hook

`pi-ai` 已将 system prompt、messages、tools 转成某 provider 的协议 payload 后，coding-agent 调用 extension `before_provider_request`，扩展可替换 payload；之后 `before_provider_headers` 可原地改 HTTP headers。

```text
context hook：改 AgentMessage（provider 无关）
payload hook：改 HTTP/SDK 协议对象（provider 相关）
```

## 7. 通用 AgentHarness 的对应机制

`agent-core AgentHarness` 有同样的抽象，但不是 Pi CLI 当前主路径：

| 时机 | Harness hook | 可做什么 |
| --- | --- | --- |
| turn 开始 | `before_agent_start` | 追加 message 或覆盖 system prompt |
| 每次模型调用前 | `context` | 替换 AgentMessage[] |
| provider 请求前 | `before_provider_request` | 改 timeout/header/metadata 等 options |
| payload 形成后 | `before_provider_payload` | 替换 payload |
| 显式 skill | `skill(name)` | 用 `<skill ...>` 包装全文 |
| 显式模板 | `promptFromTemplate()` | 仅做参数替换 |

当前 coding-agent 直接使用 runtime `Agent` 与自己的 `AgentSession` 实现对应处理，不能把 Harness 的 API 反推为 CLI 的真实调用栈。

## 8. 安全与理解边界

1. XML 标签只是模型提示结构，不是安全沙箱；项目文件和 skill 仍是会影响模型的非信任文本。
2. `input`、`before_agent_start`、`context`、payload hook 都有强改写权，应单独评估 extension 信任边界。
3. context hook 改的是请求投影，未必回写 Session；compaction 是另一条会写 session entry 的历史压缩路径。
4. `/skill:` 内联全文；普通 skill 自动发现只登记定位信息，两者 token 成本和指令强度不同。

## 源码导航

| 主题 | 源码 |
| --- | --- |
| 输入、展开、before start | [agent-session.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts) |
| system prompt 组装 | [system-prompt.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/system-prompt.ts) |
| AGENTS/CLAUDE 发现 | [resource-loader.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/resource-loader.ts) |
| template 参数替换 | [prompt-templates.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/prompt-templates.ts) |
| skill 列表包装 | [skills.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/skills.ts) |
| extension hook 串联 | [runner.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/extensions/runner.ts) |
| 通用 Harness 对应机制 | [agent-harness.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/agent/src/harness/agent-harness.ts) |
