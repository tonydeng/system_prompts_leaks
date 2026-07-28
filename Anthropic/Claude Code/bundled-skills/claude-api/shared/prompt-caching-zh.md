# 提示缓存 — 设计与优化

本文件涵盖如何设计提示构建代码以实现有效缓存。关于特定语言的语法，请参阅各语言 README 或单文件文档中的 `## Prompt Caching` 章节。

## 一切不变式的基础

**提示缓存是前缀匹配。前缀中任何位置的任何更改都会使其后的一切失效。**

缓存键源自渲染提示到每个 `cache_control` 断点的精确字节。位置 N 处的一个字节差异，一个时间戳、一个重新排序的 JSON 键、列表中一个不同的工具，都会使位置 ≥ N 的所有断点的缓存失效。

渲染顺序是：`tools` → `system` → `messages`。在最后一个 system 块上设置断点会同时缓存 tools 和 system。

围绕这个约束来设计提示构建路径。把顺序弄对，大部分缓存就能自然工作。弄错了，再多 `cache_control` 标记也无济于事。

---

## 优化现有代码的工作流程

当被要求添加或优化缓存时：

1. **追踪提示组装路径。** 找到 `system`、`tools` 和 `messages` 的构造位置。识别流入它们的每个输入。
2. **按稳定性对每个输入分类：**
   - 从不变化 → 应放在提示早期，任何断点之前
   - 每会话变化 → 应放在全局前缀之后，按会话缓存
   - 每轮次变化 → 应放在末尾，最后一个断点之后
   - 每请求变化（时间戳、UUID、随机 ID）→ **消除或移到最后**
3. **检查渲染顺序是否匹配稳定性顺序。** 稳定内容必须物理上先于易变内容。如果时间戳被插入到系统提示头部，其后所有内容无论标记如何都不可缓存。
4. **在稳定性边界处放置断点。** 参见下文的放置模式。
5. **审计静默失效因子。** 参见反模式表。

---

## 放置模式

### 大型系统提示在多个请求间共享

在最后一个 system 文本块上放置断点。如果有工具，它们在 system 之前渲染，最后一个 system 块上的标记会同时缓存 tools + system。

```json
"system": [
  {"type": "text", "text": "<large shared prompt>", "cache_control": {"type": "ephemeral"}}
]
```

### 多轮对话

在最近追加的轮次的最后一个内容块上放置断点。每个后续请求复用整个先前对话前缀。较早的断点仍然是有效的读取点，因此随着对话增长，命中会增量累积。

```json
// 最后一个用户轮次的最后一个内容块
messages[-1].content[-1].cache_control = {"type": "ephemeral"}
```

### 共享前缀，变化后缀

许多请求共享一个大型固定前言（few-shot 示例、检索到的文档、指令），但最终问题不同。将断点放在**共享**部分的末尾，而不是整个提示的末尾，否则每个请求都写入一个不同的缓存条目，永远不会有读取。

```json
"messages": [{"role": "user", "content": [
  {"type": "text", "text": "<shared context>", "cache_control": {"type": "ephemeral"}},
  {"type": "text", "text": "<varying question>"}  // 无标记 — 每次都不同
]}]
```

### 对话中途的系统消息

**仅限 Claude Opus 4.8；无需 beta 头。** 当操作者指令在对话中途到达，模式切换、更新上下文、动态注入状态，将其作为 `{"role": "system", "content": "..."}` 追加到 `messages[]`，而不是编辑顶层 `system`。编辑顶层 `system` 会改变整个对话历史之前的前缀，因此每个缓存的轮次都会被重新未缓存处理；而 `role: "system"` 消息位于历史之后，保持缓存前缀完整。

```json
// 顶层 system 保持字节一致；新指令放在缓存的历史之后
"system": [{"type": "text", "text": "<stable core>", "cache_control": {"type": "ephemeral"}}],
"messages": [
  ...history,
  {"role": "user", "content": "..."},
  {"role": "system", "content": "Terse mode enabled — keep responses under 40 words."}
]
```

这也是将操作者指令作为文本嵌入用户轮次（`<system-reminder>` 模式）的提示注入安全替代方案：两者具有相同的缓存特性，但 `role: "system"` 是不可伪造的操作者通道，而用户/工具内容中的文本可以被任何写入用户可见输入的内容伪造。

在 Claude Opus 4.8 上可用；无需 beta 头。必须跟随在 `role: "user"` 消息（或以服务端工具使用结束的 `assistant` 消息）之后，且必须是 `messages` 中的最后一个条目或后跟一个 `assistant` 轮次；不能是 `messages[0]`，初始提示请使用顶层 `system`。内容仅为文本。不支持的模型返回 400（`BadRequestError`：`role 'system' is not supported on this model`）；捕获该错误并回退到将指令放在用户轮次的 `<system-reminder>` 块中。

### 每次都从头开始变化的提示

不要缓存。如果前 1K token 每次请求都不同，就没有可复用的前缀。添加 `cache_control` 只会付出缓存写入溢价而零读取。不要加。

---

## 架构指导

这些是比标记放置更重要的事情。先解决这些。

**保持系统提示冻结。** 不要在系统提示中插入"当前日期：X"、"模式：Y"、"用户名：Z"，这些位于前缀最前面，会使下游一切失效。将动态上下文注入到后面的 `messages` 中，在支持的地方作为 `{"role": "system", ...}` 消息（参见上文 § 对话中途的系统消息），否则作为用户消息中的文本。第 5 轮的消息不会使第 5 轮之前的内容失效。

**不要在对话中途更改工具或模型。** 工具在位置 0 渲染；添加、删除或重新排序工具会使整个缓存失效。切换模型也一样（缓存是模型作用域的）。如果需要"模式"，不要交换工具集，给 Claude 一个记录模式转换的工具，或将模式作为消息内容传递。确定性地序列化工具（按名称排序）。

**分叉操作必须复用父级的精确前缀。** 辅助计算（摘要、压缩、子代理）通常会启动一个单独的 API 调用。如果分叉以任何差异重建 `system` / `tools` / `model`，它会完全错过父级的缓存。逐字复制父级的 `system`、`tools` 和 `model`，然后在末尾追加分叉特定的内容。

---

## 静默失效因子

审查代码时，在所有流入提示前缀的内容中 grep 以下模式：

| 模式 | 为什么会破坏缓存 |
|---|---|
| 系统提示中的 `datetime.now()` / `Date.now()` / `time.time()` | 前缀每次请求都变化 |
| 内容早期的 `uuid4()` / `crypto.randomUUID()` / 请求 ID | 同上，每次请求都唯一 |
| 没有 `sort_keys=True` 的 `json.dumps(d)` / 迭代 `set` | 非确定性序列化 → 前缀字节不同 |
| 将会话/用户 ID 插入系统提示的 f-string | 每用户前缀；无法跨用户共享 |
| 条件系统段落（`if flag: system += ...`） | 每种标志组合都是一个不同的前缀 |
| `tools=build_tools(user)`，其中集合因用户而异 | 工具在位置 0 渲染；跨用户无缓存 |

通过将动态部分移到最后一个断点之后、使其确定性化，或在不重要时删除来修复。

---

## API 参考

```json
"cache_control": {"type": "ephemeral"}              // 5 分钟 TTL（默认）
"cache_control": {"type": "ephemeral", "ttl": "1h"} // 1 小时 TTL
```

- 每次请求最多 **4** 个 `cache_control` 断点。
- 可放在任何内容块上：系统文本块、工具定义、消息内容块（`text`、`image`、`tool_use`、`tool_result`、`document`）。
- `messages.create()` 上的顶层 `cache_control` 会自动放在最后一个可缓存块上，不需要精细放置时最简单的选择。
- 最小可缓存前缀取决于模型。更短的前缀即使有标记也静默不缓存，没有错误，只是 `cache_creation_input_tokens: 0`：

| 模型 | 最小值 |
|---|---:|
| Opus 4.8, Opus 4.7, Opus 4.6, Opus 4.5, Haiku 4.5 | 4096 token |
| Fable 5, Sonnet 4.6, Haiku 3.5, Haiku 3 | 2048 token |
| Sonnet 4.5, Sonnet 4.1, Sonnet 4, Sonnet 3.7 | 1024 token |

一个 3K token 的提示在 Sonnet 4.5 和 Fable 5 上会缓存，但在 Opus 4.8 上静默不缓存。

**经济性：** 缓存读取成本约为基础输入价格的 0.1 倍。缓存写入成本为 **5 分钟 TTL 1.25 倍，1 小时 TTL 2 倍**。盈亏平衡取决于 TTL：使用 5 分钟 TTL，两个请求即可盈亏平衡（1.25× + 0.1× = 1.35× vs 未缓存 2×）；使用 1 小时 TTL，你至少需要三个请求（2× + 0.2× = 2.2× vs 未缓存 3×）。1 小时 TTL 在突发流量的间隙中保持条目存活，但翻倍的写入成本意味着需要更多读取才能回本。

---

## 验证缓存命中

响应的 `usage` 对象报告缓存活动：

| 字段 | 含义 |
|---|---|
| `cache_creation_input_tokens` | 本次请求写入缓存的 token（你支付了约 1.25× 写入溢价） |
| `cache_read_input_tokens` | 本次请求从缓存服务的 token（你支付了约 0.1×） |
| `input_tokens` | 以全价处理的 token（未缓存） |

如果 `cache_read_input_tokens` 在具有相同前缀的重复请求中为零，则存在静默失效因子，对比两个请求的渲染提示字节来找到它。

**`input_tokens` 仅为未缓存的剩余部分。** 提示总大小 = `input_tokens + cache_creation_input_tokens + cache_read_input_tokens`。如果你的代理运行了数小时但 `input_tokens` 显示 4K，其余部分是从缓存服务的，检查总和，而不是单个字段。

各语言访问方式：`response.usage.cache_read_input_tokens`（Python/TS/Ruby）、`$message->usage->cacheReadInputTokens`（PHP）、`resp.Usage.CacheReadInputTokens`（Go/C#）、`.usage().cacheReadInputTokens()`（Java）。

---

## 失效层级

并非每个参数更改都会使一切失效。API 有三个缓存层级，更改只会使其自身层级及以下失效：

| 更改 | 工具缓存 | 系统缓存 | 消息缓存 |
|---|:---:|:---:|:---:|
| 工具定义（添加/删除/重新排序） | ❌ | ❌ | ❌ |
| 模型切换 | ❌ | ❌ | ❌ |
| `speed`、网页搜索、引用切换 | ✅ | ❌ | ❌ |
| 系统提示内容 | ✅ | ❌ | ❌ |
| `tool_choice`、图片、`thinking` 启用/禁用 | ✅ | ✅ | ❌ |
| 消息内容 | ✅ | ✅ | ❌ |

含义：你可以按请求更改 `tool_choice` 或切换 `thinking` 而不丢失 tools+system 缓存。不要过度担心这些，只有工具定义和模型更改才会强制完全重建。

---

## 20 块回溯窗口

每个断点最多向后回溯 **20 个内容块** 来查找先前的缓存条目。如果单个轮次添加了超过 20 个块（在有许多 tool_use/tool_result 对的代理循环中很常见），下一个请求的断点将找不到先前的缓存并静默错过。

修复：在长轮次中每约 15 个块放置一个中间断点，或将标记放在距离上一轮次最后缓存块 20 个以内的块上。

---

## 并发请求时序

缓存条目仅在第一个响应**开始流式传输**后变为可读。N 个具有相同前缀的并行请求都支付全价，没有一个能读取其他请求仍在写入的内容。

对于扇出模式：发送 1 个请求，等待第一个流式 token（不是完整响应），然后发送剩余的 N-1 个。它们将读取第一个请求刚写入的缓存。

## 预热缓存

要消除*第一个*真实请求上的缓存未命中延迟，在启动时（或按间隔）发送一个 **`max_tokens: 0`** 请求。API 运行预填充，在你的 `cache_control` 断点写入缓存，并立即返回 `content: []`、`stop_reason: "max_tokens"` 和填充的 `usage` 块（零输出 token 计费；正常的缓存写入费用在 `cache_creation_input_tokens` 上）。

**何时预热** — 预热用*现在*的缓存写入费用换取*下一个*真实请求上更低的 TTFT。当以下三点都成立时值得做：(a) 首请求延迟对用户可见（聊天/语音/交互式，非后台任务），(b) 共享前缀足够大，冷写入明显缓慢，(c) 有一个在流量*之前*的时机来触发它，应用启动、工作进程启动、部署后、计划窗口开始。

| 跳过预热的情况... | 因为 |
|---|---|
| 流量是持续的（请求间隔 ≤ TTL） | 第一个真实请求预热缓存，每个后续请求都命中；单独的预热调用是纯粹的额外写入 |
| 前缀很小或低于可缓存最小值 | 冷写入惩罚可忽略 |
| 前缀因请求/用户而异 | 没有共享内容可预热 |
| 你会推测性地预热许多不同前缀 | 每个都是约 1.25× 写入；成本可能超过你节省的延迟 |

**计划性重新预热：** 仅在流量有长于 TTL 的间隙时需要。如果真实请求到达频率高于每 5 分钟一次，它们会自行保持缓存温暖，不要添加间隔重新预热。对于有长空闲间隙的突发流量，要么在略低于 TTL 时重新预热，要么切换到 `ttl: "1h"` 并降低重新预热频率。

```python
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=0,
    system=[{
        "type": "text",
        "text": SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"},
    }],
    messages=[{"role": "user", "content": "warmup"}],
)
```

**断点放置：** 将 `cache_control` 放在与真实请求**共享的最后一个块**上（系统提示或工具定义），**不是**放在占位用户消息上，也**不是**通过顶层自动缓存（这会将缓存键到占位符）。占位符可以是任何非空白字符串；它在预填充期间被读取但从不被回答。

**被拒绝的组合：** `max_tokens: 0` 与 `stream: true`、`thinking.type: "enabled"`、`output_config.format`、`{"type":"tool"}` 或 `{"type":"any"}` 类型的 `tool_choice`，或在 Message Batches 请求中一起使用会返回 `invalid_request_error`。

**TTL 仍然适用** — 默认缓存至少每 5 分钟重新预热，或使用 1 小时 TTL。这取代了旧的 `max_tokens: 1` 变通方法（无需丢弃的单 token 回复，不计费输出 token，意图明确）。
