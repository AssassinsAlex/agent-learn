# TokenMeter 估算算法：完整演示

## 先说结论

DeepSeek Harness 的 `token-meter` 做的是固定启发式估算，不是调用模型对应的精确 tokenizer：

```text
文本 token ≈ ceil(字符数 / 4)
再加 content block、message、system/tools 的结构开销
```

它同时采用增量 replay：

```text
Session append-only log
  -> 增量消费新事件
  -> 维护当前 surface 的 message-level token 价格
  -> replacement 时扣除旧节点、加入新节点
  -> measure() 返回当前 pressure 快照
```

注意：估算单位是 message node，不是长期保存每个 content block 的 token 表；replay 增量进行，但 `measure()` 返回前会复制全部当前 surface nodes，因此快照构造是 `O(surface)`。

源码入口：

- `packages/llm/token-meter/src/estimate.ts`：单个 header/message/content 的价格。
- `packages/llm/token-meter/src/surface-fold.ts`：surface append/replacement 的增量价格。
- `packages/llm/token-meter/src/index.ts`：Session replay、pressure baseline 和 provider usage anchor。

## 一、基础常量

实现中的三个核心常量是：

```ts
CHARS_PER_TOKEN = 4
BLOCK_OVERHEAD = 4
ROLE_OVERHEAD = 4
```

下面都使用 ASCII 文本，便于手算。源码实际使用 JavaScript `string.length`，即 UTF-16 code unit 计数；emoji 等非 BMP 字符可能占两个 code unit。

## 二、content block 如何计价

### 1. text block

公式：

```text
textPrice = ceil(text.length / 4) + 4
```

例子：

```text
文本："abcdefghij"
长度：10
ceil(10 / 4) = 3
block price = 3 + 4 = 7
```

边界：

```text
"abcd" -> ceil(4 / 4) + 4 = 5
""     -> ceil(0 / 4) + 4 = 4
```

### 2. reasoning block

`reasoning` 与 `text` 使用同一公式。meter 会给 reasoning 计价，但 provider usage 汇总不会把 reasoning 再额外加一次，因为它属于 output bucket 的细分。

### 3. tool-call block

```text
toolCallPrice
  = ceil(name.length / 4)
  + ceil(arguments.length / 4)
  + 4
```

例子：

```text
name      = "read"              -> 1
arguments = '{"path":"a.txt"}'  -> 长度 16，得到 4
结构开销                         -> 4

tool-call price = 1 + 4 + 4 = 9
```

arguments 按字符串长度估算，不解析 JSON 后另算字段数量。

### 4. tool-result block

`tool-result` 是递归结构：

```text
toolResultPrice = estimateContent(toolResult.content) + 4
```

内部一个 10 字符 text：

```text
inner text      = ceil(10 / 4) + 4 = 7
tool-result     = 7 + 4 = 11
```

内部两个 text block：

```text
"abcd"       -> 1 + 4 = 5
"abcdefghij" -> 3 + 4 = 7
tool-result  -> 5 + 7 + 4 = 16
```

### 5. image 和未知 block

非 text/reasoning/tool-call/tool-result 类型走默认分支：

```text
unknownBlockPrice = 4 + ceil(JSON.stringify(block).length / 4)
```

这个价格只反映 JSON 结构长度，不代表图片真实字节或视觉 token 成本。

## 三、message 如何计价

```text
messagePrice = sum(each content block price) + 4
```

示例 A：

```text
message.content = [text("abcdefghij")]
text block      = 7
role overhead   = 4
message total   = 11
```

示例 B：

```text
reasoning: "thinking" 长度 8 -> 2 + 4 = 6
tool-call: name "read" 长度 4，arguments "{}" 长度 2
            -> 1 + 1 + 4 = 6
message total = 6 + 6 + 4 = 16
```

示例 C：tool-result 内部 text 长度 10：

```text
inner text    = 7
tool-result   = 7 + 4 = 11
message total = 11 + 4 = 15
```

## 四、system 和 tools 如何计价

```text
systemPrice = ceil(system.length / 4) + 4
toolsPrice  = ceil(JSON.stringify(tools).length / 4) + 4
```

没有 system 或 tools 时对应价格为 0；tools 为空数组时 tools price 也为 0。

例子：

```text
system = "abcd" -> 1 + 4 = 5
tools JSON = '{"name":"x"}'，长度 12 -> 3 + 4 = 7
```

当前实现的 `estimateHeader()` 主要由 system 和 tools 组成。provider/model、采样参数等由 adapter 处理，不能把这个启发式 header 价格当作完整 provider serializer 的精确价格。

## 五、一个完整 request 示例

假设：

```text
system = "abcd"                         -> 5
tools JSON = '{"name":"x"}'             -> 7
user message: text("abcdefghij")        -> 11
assistant: reasoning + tool-call          -> 16
tool message: tool-result with 10 chars  -> 15
```

于是：

```text
headerTokens  = 5 + 7 = 12
surfaceTokens = 11 + 16 + 15 = 42
heuristic total = 12 + 42 = 54
```

没有 provider usage anchor 时，`totalTokens` 就是这个启发式总量。

## 六、为什么是增量 replay

每个 Session 的 replay state 大致是：

```ts
{
  consumedEvents,
  header,
  surface: [{ seq, tokens }, ...],
  surfaceTokens,
  stepStart,
  anchor,
}
```

如果已经处理到 seq 100，新增 seq 101：

```text
_sync()
  -> 只读取 seq 101
  -> deriveEventMessage(seq 101)
  -> estimateMessage(message)
  -> append { seq: 101, tokens: x }
  -> surfaceTokens += x
```

不会因为新增一个事件而重新估算 seq 0 到 seq 100 的全部历史。

## 七、replacement 如何增量更新

当前 surface：

```text
node A = 11
node B = 16
node C = 15
node D = 20
surfaceTokens = 62
```

把 A 到 C 替换为一个 18 token checkpoint：

```text
shadowedTokens    = 11 + 16 + 15 = 42
replacementTokens = 18
delta              = 18 - 42 = -24
```

更新后：

```text
surface = [checkpoint: 18, D: 20]
surfaceTokens = 62 - 24 = 38
```

旧节点仍在 append-only log，但不再计入当前 surface pressure。

## 八、provider usage anchor

provider usage 总量按不重叠 bucket 计算：

```text
usageTotal
  = inputTokens
  + cacheReadTokens
  + cacheWriteTokens
  + outputTokens
```

reasoning 不再额外添加。

假设某次完整启发式价格是 100，provider 返回：

```text
input       = 70
cacheRead   = 10
cacheWrite  = 5
output      = 20
usageTotal  = 105
```

如果 canonical request header 完全匹配，且 `105 >= 100`，就建立 usage baseline 105。之后 surface 新增 20：

```text
totalTokens = 105 + 20 = 125
```

之后 replacement 造成 `surfaceDelta = -40`：

```text
totalTokens = 105 - 40 = 65
```

system、tools、provider、model 或 request 配置变化，都会使旧 anchor 不能复用，回退为：

```text
estimateHeader(currentHeader) + currentSurfaceTokens
```

## 九、边界场景

### 空 Session

```text
header = undefined
surface = []
baseline = none(0)
surfaceTokens = 0
totalTokens = 0
```

### 只有 system

```text
system = "abcd" -> 5
surfaceTokens = 0
totalTokens = 5
```

### 只有 log-only event

`compaction/start`、`compaction/summary`、`compaction/end` 主要影响日志和锁状态，不是普通 model-visible surface message，通常不增加 `surfaceTokens`。

### replacement 反而变大

```text
old range = 10
replacement = 14
delta = +4
```

meter 会如实增加 4。Compaction 事务另行检查 framed summary 必须小于 shadowed content，避免正常摘要提交变大 checkpoint。

### replacement range 无效

如果 replacement 指向不在当前 surface 的 seq，或 start 位于 end 之后，`foldSurfaceTokens()` 会抛错，而不是跳过，因为这表示日志和 surface 不一致。

### 中文、emoji、JSON

固定 4 字符/token 是近似规则：中文通常会被低估；`string.length` 对一个 emoji 可能算作 2；JSON 的字段名、括号和引号全部按字符计价；tools schema 也可能被低估。因此它适合压力趋势、压缩触发和相对预算，不是 provider 精确 context usage。

## 十、最终心智模型

```text
estimate.ts
  = 给 block、message、system、tools 定价

surface-fold.ts
  = append/replacement 时维护当前 surface 总价

TokenMeter._sync()
  = 从上次位置增量 replay Session log

TokenMeter.measure()
  = baseline + surface delta
  + 返回 O(surface) 的不可变 nodes 快照

compaction-basic
  = 使用这些价格判断 threshold、retain budget 和摘要收敛
```
