# Claude API — TypeScript

| 功能 | 命名空间 | 关键类型 / 调用 |
|---|---|---|
| 用户配置 | beta | `client.beta.userProfiles.create(...)` / `.retrieve(id)` / `.list()`。将返回的 profile id 传递给 `client.beta.messages.create`。需要 beta 头——查阅 SDK 的 beta-headers 参考获取当前标志。 |

## 安装

```bash
npm install @anthropic-ai/sdk
```

> **读取本地文件（ESM）：** `__dirname` 和 `__filename` 在 ES 模块中是 **undefined** 的——在运行时使用两者都会抛出 `ReferenceError: __dirname is not defined`。对于相对于当前工作目录的读取，传递裸相对路径（`fs.readFileSync("./sample.png")`）。对于相对于脚本的路径，从 `import.meta.url` 派生目录：`const here = path.dirname(fileURLToPath(import.meta.url))`。绝不要在 ESM `.ts` 文件中写 `path.join(__dirname, …)`。

## 客户端初始化

```typescript
import Anthropic from "@anthropic-ai/sdk";

// 默认——从环境中解析凭证：
// ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN 或 `ant auth login` 配置。
// 本地开发优先使用此方式；不要硬编码密钥。
const client = new Anthropic();

// 显式 API 密钥（仅当必须注入特定密钥时）
const client = new Anthropic({ apiKey: "your-api-key" });
```

---

## 基本消息请求

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [{ role: "user", content: "What is the capital of France?" }],
});
// response.content 是 ContentBlock[]——一个可辨识联合体。在访问 .text 之前
// 先通过 .type 收窄类型（不这样做 TypeScript 会在 content[0].text 上报错）。
for (const block of response.content) {
  if (block.type === "text") {
    console.log(block.text);
  }
}
```

---

## 系统提示

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  system:
    "You are a helpful coding assistant. Always provide examples in Python.",
  messages: [{ role: "user", content: "How do I read a JSON file?" }],
});
```

### 对话中途的系统消息（模型受限）

对于在对话中途到达的操作者指令（模式切换、注入状态），将 `{role: "system", ...}` 追加到 `messages` 而不是编辑顶层 `system`——这保留了缓存前缀并携带操作者权限。必须跟在用户消息（或以服务端工具使用结束的 `assistant` 消息）之后，且必须是 `messages` 中的最后一个条目或后跟一个 `assistant` 轮次；不能是 `messages[0]`。不支持的模型返回 400（`role 'system' is not supported on this model`）。参见 `shared/prompt-caching.md` 了解何时使用此方式 vs. 顶层 `system`。

```typescript
// 无需 beta 头——使用常规 client.messages.create。
const response = await client.messages.create({
  model: MODEL_ID, // 必须支持对话中途系统消息
  max_tokens: 16000,
  system: [
    { type: "text", text: STABLE_SYSTEM, cache_control: { type: "ephemeral" } },
  ],
  messages: [
    ...history,
    { role: "user", content: userMessage },
    { role: "system", content: "Terse mode enabled — keep responses under 40 words." },
  ],
});
```

---

## 视觉（图片）

### URL

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: [
        {
          type: "image",
          source: { type: "url", url: "https://example.com/image.png" },
        },
        { type: "text", text: "Describe this image" },
      ],
    },
  ],
});
```

### Base64

```typescript
import fs from "fs";

const imageData = fs.readFileSync("image.png").toString("base64");

const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: [
        {
          type: "image",
          source: { type: "base64", media_type: "image/png", data: imageData },
        },
        { type: "text", text: "What's in this image?" },
      ],
    },
  ],
});
```

---

## 提示缓存

**缓存是前缀匹配**——前缀中任何位置的任何字节更改都会使其后的一切失效。关于放置模式、架构指导（冻结的系统提示、确定性的工具顺序、易变内容的放置位置）和静默失效因子审计清单，请阅读 `shared/prompt-caching.md`。

### 自动缓存（推荐）

使用顶层 `cache_control` 自动缓存请求中最后一个可缓存的块：

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  cache_control: { type: "ephemeral" }, // 自动缓存最后一个可缓存块
  system: "You are an expert on this large document...",
  messages: [{ role: "user", content: "Summarize the key points" }],
});
```

### 手动缓存控制

对于细粒度控制，将 `cache_control` 添加到特定内容块：

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  system: [
    {
      type: "text",
      text: "You are an expert on this large document...",
      cache_control: { type: "ephemeral" }, // 默认 TTL 为 5 分钟
    },
  ],
  messages: [{ role: "user", content: "Summarize the key points" }],
});

// 显式 TTL（存活时间）
const response2 = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  system: [
    {
      type: "text",
      text: "You are an expert on this large document...",
      cache_control: { type: "ephemeral", ttl: "1h" }, // 1 小时 TTL
    },
  ],
  messages: [{ role: "user", content: "Summarize the key points" }],
});
```

### 验证缓存命中

```typescript
console.log(response.usage.cache_creation_input_tokens); // 写入缓存的 token（约 1.25x 成本）
console.log(response.usage.cache_read_input_tokens);     // 从缓存服务的 token（约 0.1x 成本）
console.log(response.usage.input_tokens);                // 未缓存的 token（全价）
```

如果在重复的相同前缀请求中 `cache_read_input_tokens` 为零，则存在静默失效因子——系统提示中的 `Date.now()` 或 UUID、非确定性的键排序，或变化的工具集。参见 `shared/prompt-caching.md` 获取完整的审计表。

---

## 扩展思考

> **Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考。`budget_tokens` 在 Fable 5、Opus 4.8 和 4.7 上已移除（发送则返回 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。
> **旧模型：** 使用 `thinking: {type: "enabled", budget_tokens: N}`（必须 < `max_tokens`，最小 1024）。

```typescript
// Fable 5 / Opus 4.8 / 4.7 / 4.6：自适应思考（推荐）
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  thinking: { type: "adaptive", display: "summarized" }, // display 为可选：在 Fable 5 / Mythos 5 / Opus 4.8 / 4.7 上默认省略（空思考文本）
  output_config: { effort: "high" }, // low | medium | high | max
  messages: [
    { role: "user", content: "Solve this math problem step by step..." },
  ],
});

for (const block of response.content) {
  if (block.type === "thinking") {
    console.log("Thinking:", block.thinking);
  } else if (block.type === "text") {
    console.log("Response:", block.text);
  }
}
```

---

## 错误处理

使用 SDK 的类型化异常类——绝不要用字符串匹配检查错误消息：

```typescript
import Anthropic from "@anthropic-ai/sdk";

try {
  const response = await client.messages.create({...});
} catch (error) {
  if (error instanceof Anthropic.BadRequestError) {
    console.error("Bad request:", error.message);
  } else if (error instanceof Anthropic.AuthenticationError) {
    console.error("Invalid API key");
  } else if (error instanceof Anthropic.RateLimitError) {
    console.error("Rate limited - retry later");
  } else if (error instanceof Anthropic.APIError) {
    console.error(`API error ${error.status}:`, error.message);
  }
}
```

所有类都继承自 `Anthropic.APIError`，带有类型化的 `status` 字段。从最具体到最不具体的顺序检查。参见 [shared/error-codes.md](../../shared/error-codes.md) 获取完整的错误代码参考。

---

## 多轮对话

API 是无状态的——每次发送完整的对话历史。使用 `Anthropic.MessageParam[]` 来类型化消息数组：

```typescript
const messages: Anthropic.MessageParam[] = [
  { role: "user", content: "My name is Alice." },
  { role: "assistant", content: "Hello Alice! Nice to meet you." },
  { role: "user", content: "What's my name?" },
];

const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: messages,
});
```

**规则：**

- 允许连续的相同角色消息——API 会将它们合并为单个轮次
- 第一条消息必须是 `user`
- 对所有 API 数据结构使用 SDK 类型（`Anthropic.MessageParam`、`Anthropic.Message`、`Anthropic.Tool` 等）——不要重新定义等价的接口

---

### 压缩（长对话）

> **Beta，Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6。** 当对话接近 200K 上下文窗口时，压缩会自动在服务端摘要较早的上下文。API 返回一个 `compaction` 块；你必须在后续请求中将其传回——追加 `response.content`，而不仅仅是文本。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const messages: Anthropic.Beta.BetaMessageParam[] = [];

async function chat(userMessage: string): Promise<string> {
  messages.push({ role: "user", content: userMessage });

  const response = await client.beta.messages.create({
    betas: ["compact-2026-01-12"],
    model: "claude-opus-4-8",
    max_tokens: 16000,
    messages,
    context_management: {
      edits: [{ type: "compact_20260112" }],
    },
  });

  // 追加完整内容——压缩块必须保留
  messages.push({ role: "assistant", content: response.content });

  const textBlock = response.content.find(
    (b): b is Anthropic.Beta.BetaTextBlock => b.type === "text",
  );
  return textBlock?.text ?? "";
}

// 上下文增长时压缩自动触发
console.log(await chat("Help me build a Python web scraper"));
console.log(await chat("Add support for JavaScript-rendered pages"));
console.log(await chat("Now add rate limiting and error handling"));
```

---

## 停止原因

响应中的 `stop_reason` 字段指示模型为何停止生成：

| 值              | 含义                                                           |
| --------------- | --------------------------------------------------------------- |
| `end_turn`      | Claude 自然完成了回复                                           |
| `max_tokens`    | 达到 `max_tokens` 限制——增加它或使用流式传输                   |
| `stop_sequence` | 命中自定义停止序列                                              |
| `tool_use`      | Claude 想要调用工具——执行它并继续                              |
| `pause_turn`    | 模型暂停并可恢复（代理流程）                                    |
| `refusal`       | Claude 因安全原因拒绝——检查 `stop_details`                     |

### 结构化停止详情

当 `stop_reason` 为 `"refusal"` 时，响应包含一个 `stop_details` 对象，带有关于拒绝的结构化信息：

```typescript
if (response.stop_reason === "refusal" && response.stop_details) {
  console.log(`Category: ${response.stop_details.category}`); // 例如 "cyber"、"bio"、"reasoning_extraction"、"frontier_llm" 或 null——参见文档获取完整集合
  console.log(`Explanation: ${response.stop_details.explanation}`);
}
```

### 拒绝回退（Claude Fable 5）——默认选择启用

回退是**可选的**：没有它们，被拒绝的请求简单地停止。在 `claude-fable-5` 代码中默认包含服务端 `fallbacks` 参数——在策略拒绝时，API 在同一次调用中用回退模型重新运行相同请求。在任何输出之前的拒绝不计费（流中拒绝计费已流式的部分）；救援按回退模型自身的费率计费，缓存重新定价自动应用。

```typescript
const response = await client.beta.messages.create({
  model: "claude-fable-5",
  max_tokens: 16000,
  betas: ["server-side-fallback-2026-06-01"],
  fallbacks: [{ model: "claude-opus-4-8" }],
  messages: [{ role: "user", content: "..." }],
});

// 切换点：每个运行并拒绝此轮的模型对应一个回退块
for (const block of response.content) {
  if (block.type === "fallback") {
    console.log(`${block.from.model} declined; ${block.to.model} continued`);
  }
}

// 服务方信号——覆盖粘性轮次，后者不携带回退块。
// 与 stop_reason 配对：回退模型本身也可能拒绝。
const fallbackRan = (response.usage.iterations ?? []).some(
  (entry) => entry.type === "fallback_message",
);
if (fallbackRan && response.stop_reason !== "refusal") {
  console.log(`Served by ${response.model}`);
}
```

最终响应上的 `stop_reason: "refusal"` 意味着整条链都拒绝了。头必须精确为 `server-side-fallback-2026-06-01`；该参数在 Batches API 上被拒绝，在 Amazon Bedrock、Vertex AI 和 Microsoft Foundry 上不可用——在这些平台上改为在客户端注册客户端侧 `betaRefusalFallbackMiddleware`。完整语义（粘性路由、计费、流式传输、回传回退轮次）：`shared/model-migration.md` → 迁移到 Claude Fable 5 → `refusal` 停止原因。

---

## 成本优化策略

### 1. 对重复上下文使用提示缓存

```typescript
// 自动缓存（最简单——缓存最后一个可缓存块）
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  cache_control: { type: "ephemeral" },
  system: largeDocumentText, // 例如 50KB 的上下文
  messages: [{ role: "user", content: "Summarize the key points" }],
});

// 第一次请求：全价
// 后续请求：缓存部分便宜约 90%
```

### 2. 在请求前使用 Token 计数

```typescript
const countResponse = await client.messages.countTokens({
  model: "claude-opus-4-8",
  messages: messages,
  system: system,
});

const estimatedInputCost = countResponse.input_tokens * 0.000005; // $5/1M tokens
console.log(`Estimated input cost: $${estimatedInputCost.toFixed(4)}`);
```
