# 输入 Prompt 示例：从原始文本到模型 Context

前置阅读：[Prompt 改写与包装](11-Prompt改写与包装.md)。本文基于 [`AgentSession.prompt()`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts#L1098)、[`expandPromptTemplate()`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/prompt-templates.ts#L269) 与 [`_expandSkillCommand()`](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts#L1284)。

## 1. 示例边界

下面的 `AGENTS.md`、skill、template 与 extension handler 是为了说明流程而假设的输入数据；改写顺序、包装格式和消息类型来自源码。时间戳、历史消息和完整工具 schema 均省略。

Pi 在模型边界有两层表示：

```text
AgentMessage[]                  provider Context / HTTP payload
─────────────────────────────   ─────────────────────────────────
user / assistant / toolResult   provider 能识别的消息和工具 schema
custom 等 Pi 内部消息            convertToLlm() 决定如何投影或过滤
```

本文展示 provider 无关的逻辑 `Context`。真实 HTTP JSON 会随 Anthropic、OpenAI 等 provider 改变。

## 2. 假设环境

```text
cwd: /repo/app
active tools: read, bash, edit

/repo/AGENTS.md:
  Tests must be run before finishing.

skill review:
  file: /home/me/.pi/skills/review/SKILL.md
  body: Inspect the diff. Report findings by severity.

template fix:
  body: Fix the issue in $1. Run $@ and report the result.
```

与该例相关的 system prompt 片段会类似：

```xml
<project_context>
  <project_instructions path="/repo/AGENTS.md">
  Tests must be run before finishing.
  </project_instructions>
</project_context>

<available_skills>
  <skill>
    <name>review</name>
    <description>Review a change for correctness and regressions.</description>
    <location>/home/me/.pi/skills/review/SKILL.md</location>
  </skill>
</available_skills>
```

真实 system prompt 前面还包含 Pi 基础指令、当前工具说明、guidelines 与 cwd。

## 3. 普通输入：user text 不变

原始输入：

```text
修复 src/auth.ts 的登录错误
```

若没有 `input` extension，且不匹配 skill/template，Pi 生成：

```ts
[
  {
    role: "user",
    content: [{ type: "text", text: "修复 src/auth.ts 的登录错误" }]
  }
]
```

这一例没有重写 user text；“包装”主要在并列发送的 system prompt 中。

### 默认不做“专业改写”

Pi 没有默认的用户输入优化器：不会把“修复登录错误”自动改成“请先分析根因、最小化修改、运行测试并报告风险”之类的长指令，也不会为每条普通 user message 自动加 XML、角色前缀或任务模板。

默认普通输入的路径就是：

```text
原始 text -> create user message -> agent loop
```

它之所以常显得“更专业”，通常来自并列的 system prompt：其中已经带有工具使用原则、项目 `AGENTS.md`、skill 元数据和当前工作目录。模型是在“原始 user text + 丰富 system prompt + 历史上下文”的组合下作答，而不是 Pi 先重写了 user text。

会实际改变 user text 的例外是显式且可追溯的：

| 机制 | 是否改 user text | 谁控制 |
| --- | --- | --- |
| `/template` | 是，原命令替换为模板正文 | 用户选择的 template |
| `/skill:name` | 是，原命令替换为 skill 全文包装 | 用户选择的 skill |
| extension `input` | 可以 | extension handler |
| `pi.sendMessage()` | 通常不展开 template，按传入内容送入 | extension / 调用方 |
| `before_agent_start` | 不改原 user text；可换 system、追加 message | extension handler |

所以若你要在 Pi 中实现“把用户输入自动专业化”，合理扩展点是 `input` event，或由应用在调用 `prompt()` 前显式改写文本；这不是 Pi 默认策略。

## 4. Template：slash 命令被正文替换

原始输入：

```text
/fix src/auth.ts npm test
```

`parseCommandArgs()` 得到：

```text
["src/auth.ts", "npm", "test"]
```

模板正文：

```text
Fix the issue in $1. Run $@ and report the result.
```

展开后的 user text：

```text
Fix the issue in src/auth.ts. Run src/auth.ts npm test and report the result.
```

源码只做位置替换，不理解参数里的文件、命令和 flag；原始 `/fix ...` 不会额外保存在 message 中。

## 5. 显式 Skill：全文内联到 user message

原始输入：

```text
/skill:review src/auth.ts
```

Pi 读取 `SKILL.md`、移除 YAML frontmatter，然后生成：

```xml
<skill name="review" location="/home/me/.pi/skills/review/SKILL.md">
References are relative to /home/me/.pi/skills/review.

Inspect the diff. Report findings by severity.
</skill>

src/auth.ts
```

这整个文本成为一个 `role: "user"` message。它不是 system message，也不会保留 `/skill:review` 原命令。

普通输入“请 review 这个改动”则只有 system prompt 里的 `<available_skills>` 元数据；模型需要自行调用 `read` 获取 skill 全文。

## 6. Extension `input`：在 skill/template 之前

假设 extension handler 为：

```ts
pi.on("input", async (event) => {
  if (event.text.startsWith("bug ")) {
    return { action: "transform", text: `Investigate and fix: ${event.text.slice(4)}` };
  }
});
```

原始输入：

```text
bug /fix src/auth.ts npm test
```

先改为：

```text
Investigate and fix: /fix src/auth.ts npm test
```

现在它不再以 `/` 开头，因此不会命中 `/fix` template；最终 user message 就是这段改写后的文本。

若 handler 返回 `{ action: "handled" }`，Pi 直接结束 `prompt()`，不会构造 user message，也没有 provider 请求。

## 7. `before_agent_start`：改 system，追加消息

在第 3 节普通输入后，假设 extension 返回：

```ts
{
  systemPrompt: "You are a security-focused coding assistant.",
  message: {
    customType: "policy-note",
    content: "Treat authentication changes as security-sensitive.",
    display: false
  }
}
```

进入 runtime 的逻辑状态变成：

```ts
systemPrompt = "You are a security-focused coding assistant."

messages = [
  { role: "user", content: [{ type: "text", text: "修复 src/auth.ts 的登录错误" }] },
  { role: "custom", customType: "policy-note", content: "Treat authentication changes as security-sensitive." }
]
```

它替换的是完整 system prompt，并在 user message 后追加 custom message；不是把字符串简单拼接到原 user text 前后。

## 8. Context hook：最后的 provider 无关改写

假设 extension `context` handler 返回：

```ts
messages.filter((message) => message.role !== "toolResult")
```

则 Session / Agent 内存历史仍可能保存 tool result，但本次 `convertToLlm()` 收到的投影已不包含它：

```text
持久化历史：user -> assistant(toolCall) -> toolResult -> 新 user
发送投影：  user -> assistant(toolCall) -> 新 user
```

这就是“改请求投影”与“改 Session 历史”的区别。

## 9. 最终逻辑 Context

以第 5 节 skill 为例，provider 前的逻辑对象可以理解为：

```ts
{
  systemPrompt: "Pi 基础指令 + project_context + available_skills + cwd",
  messages: [
    {
      role: "user",
      content: [{ type: "text", text: "<skill ...>...</skill>\n\nsrc/auth.ts" }]
    }
  ],
  tools: [readTool, bashTool, editTool]
}
```

随后 `pi-ai` 用当前 provider 编码 payload。`before_provider_request` 是 provider payload 形成后的改写点，`before_provider_headers` 则改 HTTP headers。

## 10. 检查清单

1. 输入是否被 extension command 截走？
2. `input` extension 是否改变了 slash command 的匹配条件？
3. 这是普通 skill 登记，还是 `/skill:` 全文内联？
4. system prompt 是基础构建结果，还是被 `before_agent_start` 覆盖？
5. context hook 改的是请求投影，还是 Session 历史？
6. 当前看到的是 Pi 逻辑 Context，还是 provider 专属 payload？

## 源码导航

| 主题 | 源码 |
| --- | --- |
| 输入处理顺序 | [agent-session.ts:1098](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts#L1098) |
| skill 全文包装 | [agent-session.ts:1284](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/agent-session.ts#L1284) |
| template 识别和替换 | [prompt-templates.ts:269](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/prompt-templates.ts#L269) |
| system prompt 拼装 | [system-prompt.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/system-prompt.ts) |
| extension hook 串联 | [runner.ts](https://github.com/earendil-works/pi/blob/864b35c462f9623579b068e9cab848419f9e1d0f/packages/coding-agent/src/core/extensions/runner.ts) |
