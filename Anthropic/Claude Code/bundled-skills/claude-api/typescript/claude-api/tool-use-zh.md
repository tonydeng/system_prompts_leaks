# 工具使用 — TypeScript

概念概述（工具定义、工具选择、技巧）请参见 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具运行器（推荐）

**Beta：** 工具运行器在 TypeScript SDK 中处于 beta 阶段。

使用 `betaZodTool` 配合 Zod schema 定义带有 `run` 函数的工具，然后传递给 `client.beta.messages.toolRunner()`：

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

const client = new Anthropic();

const getWeather = betaZodTool({
  name: "get_weather",
  description: "Get current weather for a location",
  inputSchema: z.object({
    location: z.string().describe("City and state, e.g., San Francisco, CA"),
    unit: z.enum(["celsius", "fahrenheit"]).optional(),
  }),
  run: async (input) => {
    // Your implementation here
    return `72°F and sunny in ${input.location}`;
  },
});

// 工具运行器处理 agentic 循环并返回最终消息
const finalMessage = await client.beta.messages.toolRunner({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools: [getWeather],
  messages: [{ role: "user", content: "What's the weather in Paris?" }],
});

console.log(finalMessage.content);
```

Zod 是可选的——如果你不想依赖 Zod，可以使用 `@anthropic-ai/sdk/helpers/beta/json-schema` 中的 `betaTool()`，它接受原始 JSON Schema `inputSchema` 加上 `run` 函数。

**工具运行器的主要优势：**

- 无需手动循环——SDK 处理工具调用和结果反馈
- 通过 Zod schema（或通过 `betaTool()` 使用原始 JSON Schema）实现类型安全的工具输入
- 工具 schema 从 Zod 定义自动生成
- 当 Claude 没有更多工具调用时迭代自动停止

### 使用工具运行器的服务器工具

运行器的 `tools` 数组接受原始服务器工具定义（`web_search_20260209`、`web_fetch_20260209`、代码执行）以及可运行工具——传递字面量工具对象即可；服务器工具在 Anthropic 的服务器上运行，因此没有 `run` 函数。

**注意——运行器不会自动恢复 `pause_turn`（截至 `@anthropic-ai/sdk` 0.110.0）。** 长时间运行的服务器工具轮次可能以 `stop_reason: "pause_turn"` 停止。运行器仅在客户端工具产生结果后继续，因此暂停的轮次会结束循环并作为最终消息返回——没有错误、没有警告，只是静默截断的答案。如果在运行器中混用服务器工具，请在每次迭代时检查 `stop_reason`，并通过将暂停的助手轮次推回来恢复：

```typescript
const params = {
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools: [getWeather, { type: "web_search_20260209", name: "web_search", max_uses: 5 }],
  messages: [{ role: "user", content: "Compare this week's forecasts for Paris across two sources" }],
};

const runner = client.beta.messages.toolRunner(params);

// 非流式：每次迭代产生一条完整消息
for await (const message of runner) {
  if (message.stop_reason === "pause_turn") {
    runner.pushMessages({ role: "assistant", content: message.content });
  }
}

// 流式替代方案——使用 `stream: true` 构建运行器（参数同上）。
// 每次迭代产生的是流而非消息——简单的 `message.stop_reason` 检查永远不会触发。
// 先解析流：
const streamingRunner = client.beta.messages.toolRunner({ ...params, stream: true });
for await (const stream of streamingRunner) {
  const message = await stream.finalMessage();
  if (message.stop_reason === "pause_turn") {
    streamingRunner.pushMessages({ role: "assistant", content: message.content });
  }
}
```

每次暂停-恢复消耗一个 `max_iterations` 计数，因此设置了上限的运行仍可能以暂停结束——在信任结果之前检查最终消息的 `stop_reason`（循环结束后对你迭代过的运行器调用 `.done()` 以获取最终消息）。或者使用下方的手动循环，它显式处理 `pause_turn`。

---

## 手动 Agentic 循环

优先使用上方的工具运行器。仅在需要运行器未暴露的控制时才使用手动循环（例如自定义传输、SDK 无法构建的请求形式，或避免 beta 依赖——运行器是 beta 的，它通过 `stream: true` 支持逐 token 流式传输）。人在环中审批*不需要*手动循环——在工具的 `run()` 函数内设门（返回"用户拒绝"结果），或在迭代之间检查待处理的 `tool_use` 块并调用 `setMessagesParams()`。

如果你确实需要手动循环：

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const tools: Anthropic.Tool[] = [...]; // 你的工具定义
let messages: Anthropic.MessageParam[] = [{ role: "user", content: userInput }];

while (true) {
  const response = await client.messages.create({
    model: "claude-opus-4-8",
    max_tokens: 16000,
    tools: tools,
    messages: messages,
  });

  if (response.stop_reason === "end_turn") break;

  // 服务器端工具命中迭代限制；追加助手轮次并重新发送以继续
  if (response.stop_reason === "pause_turn") {
    messages.push({ role: "assistant", content: response.content });
    continue;
  }

  const toolUseBlocks = response.content.filter(
    (b): b is Anthropic.ToolUseBlock => b.type === "tool_use",
  );

  messages.push({ role: "assistant", content: response.content });

  const toolResults: Anthropic.ToolResultBlockParam[] = [];
  for (const tool of toolUseBlocks) {
    const result = await executeTool(tool.name, tool.input);
    toolResults.push({
      type: "tool_result",
      tool_use_id: tool.id,
      content: result,
    });
  }

  messages.push({ role: "user", content: toolResults });
}
```

### 流式手动循环

在手动循环中需要流式传输时，使用 `client.messages.stream()` + `finalMessage()` 而非 `.create()`。每次迭代中文本增量会被流式传输；`finalMessage()` 收集完整的 `Message`，以便你检查 `stop_reason` 并提取工具使用块：

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const tools: Anthropic.Tool[] = [...];
let messages: Anthropic.MessageParam[] = [{ role: "user", content: userInput }];

while (true) {
  const stream = client.messages.stream({
    model: "claude-opus-4-8",
    max_tokens: 64000,
    tools,
    messages,
  });

  // 每次迭代流式传输文本增量
  stream.on("text", (delta) => {
    process.stdout.write(delta);
  });

  // finalMessage() 解析为完整的 Message——无需手动连接
  // .on("message") / .on("error") / .on("abort")
  const message = await stream.finalMessage();

  if (message.stop_reason === "end_turn") break;

  // 服务器端工具命中迭代限制；追加助手轮次并重新发送以继续
  if (message.stop_reason === "pause_turn") {
    messages.push({ role: "assistant", content: message.content });
    continue;
  }

  const toolUseBlocks = message.content.filter(
    (b): b is Anthropic.ToolUseBlock => b.type === "tool_use",
  );

  messages.push({ role: "assistant", content: message.content });

  const toolResults: Anthropic.ToolResultBlockParam[] = [];
  for (const tool of toolUseBlocks) {
    const result = await executeTool(tool.name, tool.input);
    toolResults.push({
      type: "tool_result",
      tool_use_id: tool.id,
      content: result,
    });
  }

  messages.push({ role: "user", content: toolResults });
}
```

> **重要：** 不要将 `.on()` 事件包装在 `new Promise()` 中来收集最终消息——使用 `stream.finalMessage()` 代替。SDK 内部处理所有错误/中止/完成状态。

> **循环中的错误处理：** 使用 SDK 的类型化异常（例如 `Anthropic.RateLimitError`、`Anthropic.APIError`）——参见 [错误处理](./README.md#error-handling) 中的示例。不要用字符串匹配检查错误消息。

> **SDK 类型：** 使用 `Anthropic.MessageParam`、`Anthropic.Tool`、`Anthropic.ToolUseBlock`、`Anthropic.ToolResultBlockParam`、`Anthropic.Message` 等来表示所有 API 相关的数据结构。不要重新定义等价的接口。

---

## 处理工具结果

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools: tools,
  messages: [{ role: "user", content: "What's the weather in Paris?" }],
});

for (const block of response.content) {
  if (block.type === "tool_use") {
    const result = await executeTool(block.name, block.input);

    const followup = await client.messages.create({
      model: "claude-opus-4-8",
      max_tokens: 16000,
      tools: tools,
      messages: [
        { role: "user", content: "What's the weather in Paris?" },
        { role: "assistant", content: response.content },
        {
          role: "user",
          content: [
            { type: "tool_result", tool_use_id: block.id, content: result },
          ],
        },
      ],
    });
  }
}
```

---

## 工具选择

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools: tools,
  tool_choice: { type: "tool", name: "get_weather" },
  messages: [{ role: "user", content: "What's the weather in Paris?" }],
});
```

---

## Anthropic 定义的工具

带版本后缀的 `type` 字面量；`name` 按接口固定。Web 搜索和代码执行在服务器端执行；bash 和文本编辑器在客户端执行（你在本地处理 `tool_use`——参见 `shared/tool-use-concepts.md`）。传递普通对象字面量——`ToolUnion` 类型通过结构化方式满足。**`name`/`type` 对必须与接口匹配**：将 `str_replace_based_edit_tool`（20250728 的 name）与 `text_editor_20250124`（期望 `str_replace_editor`）混用会导致 TS2322。

**不要标注为 `Tool[]` 类型**——`Tool` 只是自定义工具变体。让结构化类型从 `tools` 参数推断，或者如果必须标注则使用 `Anthropic.Messages.ToolUnion[]`：

```typescript
// ✓ 让推断工作——不标注
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools: [
    { type: "text_editor_20250728", name: "str_replace_based_edit_tool" },
    { type: "bash_20250124", name: "bash" },
    { type: "web_search_20260209", name: "web_search" },
    { type: "code_execution_20260120", name: "code_execution" },
  ],
  messages: [{ role: "user", content: "..." }],
});

// ✗ 这是 TS2352——Tool 仅为自定义工具变体
// const tools: Anthropic.Tool[] = [{ type: "text_editor_20250728", ... }]
```

| 接口 | `name` | `type` |
|---|---|---|
| `ToolTextEditor20250124` | `str_replace_editor` | `text_editor_20250124` |
| `ToolTextEditor20250429` | `str_replace_based_edit_tool` | `text_editor_20250429` |
| `ToolTextEditor20250728` | `str_replace_based_edit_tool` | `text_editor_20250728` |
| `ToolBash20250124` | `bash` | `bash_20250124` |
| `WebSearchTool20260209` | `web_search` | `web_search_20260209` |
| `WebFetchTool20260209` | `web_fetch` | `web_fetch_20260209` |
| `CodeExecutionTool20260120` | `code_execution` | `code_execution_20260120` |

**不要混用 beta 和非 beta 类型**：如果你调用 `client.beta.messages.create()`，响应的 `content` 是 `BetaContentBlock[]`——你不能将其传递给非 beta 的 `ContentBlockParam[]` 而不逐个元素收窄。

---


## 代码执行

### 基本用法

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content:
        "Calculate the mean and standard deviation of [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]",
    },
  ],
  tools: [{ type: "code_execution_20260120", name: "code_execution" }],
});
```

### 读取本地文件（ESM 注意事项）

`__dirname` 在 ES 模块中不存在。对于脚本相对路径，使用 `import.meta.url`：

```typescript
import { readFileSync } from "fs";
import { fileURLToPath } from "url";
import { dirname, join } from "path";

const __dirname = dirname(fileURLToPath(import.meta.url));
const pdfBytes = readFileSync(join(__dirname, "sample.pdf"));
```

或者如果脚本从已知目录运行，使用 CWD 相对路径：`readFileSync("./sample.pdf")`。

### 上传文件进行分析

```typescript
import Anthropic, { toFile } from "@anthropic-ai/sdk";
import { createReadStream } from "fs";

const client = new Anthropic();

// 1. 上传文件
const uploaded = await client.beta.files.upload({
  file: await toFile(createReadStream("sales_data.csv"), undefined, {
    type: "text/csv",
  }),
  betas: ["files-api-2025-04-14"],
});

// 2. 传递给代码执行
// 代码执行已 GA；Files API 仍为 beta（通过 RequestOptions 传递）
const response = await client.messages.create(
  {
    model: "claude-opus-4-8",
    max_tokens: 16000,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: "Analyze this sales data. Show trends and create a visualization.",
          },
          { type: "container_upload", file_id: uploaded.id },
        ],
      },
    ],
    tools: [{ type: "code_execution_20260120", name: "code_execution" }],
  },
  { headers: { "anthropic-beta": "files-api-2025-04-14" } },
);
```

### 检索生成的文件

```typescript
import path from "path";
import fs from "fs";

const OUTPUT_DIR = "./claude_outputs";
await fs.promises.mkdir(OUTPUT_DIR, { recursive: true });

for (const block of response.content) {
  if (block.type === "bash_code_execution_tool_result") {
    const result = block.content;
    if (result.type === "bash_code_execution_result" && result.content) {
      for (const fileRef of result.content) {
        if (fileRef.type === "bash_code_execution_output") {
          const metadata = await client.beta.files.retrieveMetadata(
            fileRef.file_id,
          );
          const downloadResponse = await client.beta.files.download(fileRef.file_id);
          const fileBytes = Buffer.from(await downloadResponse.arrayBuffer());
          const safeName = path.basename(metadata.filename);
          if (!safeName || safeName === "." || safeName === "..") {
            console.warn(`Skipping invalid filename: ${metadata.filename}`);
            continue;
          }
          const outputPath = path.join(OUTPUT_DIR, safeName);
          await fs.promises.writeFile(outputPath, fileBytes);
          console.log(`Saved: ${outputPath}`);
        }
      }
    }
  }
}
```

### 容器复用

```typescript
// 第一次请求：设置环境
const response1 = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: "Install tabulate and create data.json with sample user data",
    },
  ],
  tools: [{ type: "code_execution_20260120", name: "code_execution" }],
});

// 复用容器
// container 可为 null——仅在使用服务器端代码执行时设置
const containerId = response1.container!.id;

const response2 = await client.messages.create({
  container: containerId,
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: "Read data.json and display as a formatted table",
    },
  ],
  tools: [{ type: "code_execution_20260120", name: "code_execution" }],
});
```

---

## 记忆工具

### 基本用法

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: "Remember that my preferred language is TypeScript.",
    },
  ],
  tools: [{ type: "memory_20250818", name: "memory" }],
});
```

### SDK 记忆助手

使用 `betaMemoryTool` 配合 `MemoryToolHandlers` 实现：

```typescript
import {
  betaMemoryTool,
  type MemoryToolHandlers,
} from "@anthropic-ai/sdk/helpers/beta/memory";

const handlers: MemoryToolHandlers = {
  async view(command) { ... },
  async create(command) { ... },
  async str_replace(command) { ... },
  async insert(command) { ... },
  async delete(command) { ... },
  async rename(command) { ... },
};

const memory = betaMemoryTool(handlers);

const runner = client.beta.messages.toolRunner({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  tools: [memory],
  messages: [{ role: "user", content: "Remember my preferences" }],
});

for await (const message of runner) {
  console.log(message);
}
```

完整实现示例，请通过 WebFetch 获取：

- `https://github.com/anthropics/anthropic-sdk-typescript/blob/main/examples/tools-helpers-memory.ts`

---

## 结构化输出

### JSON 输出（Zod — 推荐）

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { z } from "zod";
import { zodOutputFormat } from "@anthropic-ai/sdk/helpers/zod";

const ContactInfoSchema = z.object({
  name: z.string(),
  email: z.string(),
  plan: z.string(),
  interests: z.array(z.string()),
  demo_requested: z.boolean(),
});

const client = new Anthropic();

const response = await client.messages.parse({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content:
        "Extract: Jane Doe (jane@co.com) wants Enterprise, interested in API and SDKs, wants a demo.",
    },
  ],
  output_config: {
    format: zodOutputFormat(ContactInfoSchema),
  },
});

// 如果解析失败，parsed_output 为 null——断言或守卫
console.log(response.parsed_output!.name); // "Jane Doe"
```

### 严格工具使用

```typescript
const response = await client.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: "Book a flight to Tokyo for 2 passengers on March 15",
    },
  ],
  tools: [
    {
      name: "book_flight",
      description: "Book a flight to a destination",
      strict: true,
      input_schema: {
        type: "object",
        properties: {
          destination: { type: "string" },
          date: { type: "string", format: "date" },
          passengers: {
            type: "integer",
            enum: [1, 2, 3, 4, 5, 6, 7, 8],
          },
        },
        required: ["destination", "date", "passengers"],
        additionalProperties: false,
      },
    },
  ],
});
```

---

## Agent Skills

通过 `container.skills` + beta 路径上的 `code_execution` 工具启用 Anthropic 管理的 skill（例如 `pptx`）。两个 beta header 都是必需的。输出以文件形式出现在响应内容中——通过 Files API 按 file ID 下载。

```typescript
const response = await client.beta.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  container: {
    skills: [{ type: "anthropic", skill_id: "pptx", version: "latest" }],
  },
  tools: [{ type: "code_execution_20260521", name: "code_execution" }],
  betas: ["code-execution-2025-08-25", "skills-2025-10-02"],
  messages: [{ role: "user", content: "Create a 3-slide deck about X." }],
});
// 在 response.content 中找到 file_id，然后：
// await client.beta.files.download(fileId)
```
