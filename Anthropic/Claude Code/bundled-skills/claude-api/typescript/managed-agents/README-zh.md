# Managed Agents — TypeScript

> **未展示的绑定：** 本 README 涵盖 TypeScript 最常见的 managed-agents 流程。如果需要未展示的类、方法、命名空间、字段或行为，请从 `shared/live-sources.md` 中 WebFetch TypeScript SDK 仓库**或相关文档页面**，而不是猜测。不要从 cURL 形状或其他语言的 SDK 推断。

> **Agent 是持久的，创建一次，按 ID 引用。** 存储 `agents.create` 返回的 agent ID，传给后续每次 `sessions.create`；不要在请求路径中调用 `agents.create`。**推荐做法：** 将 agent 和 environment 定义为版本控制的 YAML，通过 `ant` CLI 应用，参见 `shared/anthropic-cli.md`（其在线文档 URL 在 `shared/live-sources.md` 中）。CLI 拥有控制面（创建/更新），你的代码拥有数据面（用存储的 ID 创建会话）。以下示例展示的是必须编程式配置时的代码内创建；在生产环境中，创建调用应放在设置阶段，不在请求路径中。

## 安装

```bash
npm install @anthropic-ai/sdk
```

## 客户端初始化

```typescript
import Anthropic from "@anthropic-ai/sdk";

// 默认方式 — 从环境变量解析凭证：
// ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN 或 `ant auth login` 配置。
// 本地开发优先使用此方式；不要硬编码 key。
const client = new Anthropic();

// 显式 API key（仅在需要注入特定 key 时使用）
const client = new Anthropic({ apiKey: "your-api-key" });
```

---

## 创建环境

```typescript
const environment = await client.beta.environments.create(
  {
    name: "my-dev-env",
    config: {
      type: "cloud",
      networking: { type: "unrestricted" },
    },
  },
);
console.log(environment.id); // env_...
```

---

## 创建 Agent（必需的第一步）

> ⚠️ **没有内联 agent 配置。** `model`/`system`/`tools` 位于 agent 对象上，不在 session 上。始终从 `agents.create()` 开始，session 只接受 `agent: { type: "agent", id: agent.id }`。

### 最小配置

```typescript
// 1. 创建 agent（可复用、有版本）
const agent = await client.beta.agents.create(
  {
    name: "Coding Assistant",
    model: "claude-opus-4-8",
    tools: [{ type: "agent_toolset_20260401", default_config: { enabled: true } }],
  },
);

// 2. 启动会话
const session = await client.beta.sessions.create(
  {
    agent: { type: "agent", id: agent.id, version: agent.version },
    environment_id: environment.id,
  },
);
console.log(session.id, session.status);
console.log(`Trace: https://platform.claude.com/workspaces/default/sessions/${session.id}`); // 如果 API key 不在 Default workspace 中，将 'default' 替换为你的 workspace ID
```

### 带系统提示和自定义工具

```typescript
const agent = await client.beta.agents.create(
  {
    name: "Code Reviewer",
    model: "claude-opus-4-8",
    system: "You are a senior code reviewer.",
    tools: [
      { type: "agent_toolset_20260401", default_config: { enabled: true } },
      {
        type: "custom",
        name: "run_tests",
        description: "Run the test suite",
        input_schema: {
          type: "object",
          properties: {
            test_path: { type: "string", description: "Path to test file" },
          },
          required: ["test_path"],
        },
      },
    ],
  },
);

const session = await client.beta.sessions.create(
  {
    agent: { type: "agent", id: agent.id, version: agent.version },
    environment_id: environment.id,
    title: "Code review session",
    resources: [
      {
        type: "github_repository",
        url: "https://github.com/owner/repo",
        mount_path: "/workspace/repo",
        authorization_token: process.env.GITHUB_TOKEN,
        branch: "main",
      },
    ],
  },
);
```

---

## 发送用户消息

```typescript
await client.beta.sessions.events.send(
  session.id,
  {
    events: [
      {
        type: "user.message",
        content: [{ type: "text", text: "Review the auth module" }],
      },
    ],
  },
);
```

> 💡 **先开流：** 在发送消息*之前*（或同时）打开流。流只传递打开后发生的事件，先发后开会导致早期事件以一批缓冲数据到达。参见 [Steering Patterns](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式事件（SSE）

```typescript
// 先开流：同时打开流和发送
const [events] = await Promise.all([
  collectStream(session.id),
  client.beta.sessions.events.send(
    session.id,
    { events: [{ type: "user.message", content: [{ type: "text", text: "..." }] }] },
  ),
]);

// 独立流迭代：
const stream = await client.beta.sessions.events.stream(
  session.id,
);

for await (const event of stream) {
  switch (event.type) {
    case "agent.message":
      for (const block of event.content) {
        if (block.type === "text") {
          process.stdout.write(block.text);
        }
      }
      break;
    case "agent.custom_tool_use":
      // 自定义工具调用 — 会话现在空闲
      console.log(`\nCustom tool call: ${event.name}`);
      console.log(`Input: ${JSON.stringify(event.input)}`);
      break;
    case "session.status_idle":
      console.log("\n--- Agent idle ---");
      break;
    case "session.status_terminated":
      console.log("\n--- Session terminated ---");
      break;
  }
}
```

---

## 提供自定义工具结果

```typescript
await client.beta.sessions.events.send(
  session.id,
  {
    events: [
      {
        type: "user.custom_tool_result",
        custom_tool_use_id: "sevt_abc123",
        content: [{ type: "text", text: "All 42 tests passed." }],
      },
    ],
  },
);
```

---

## 轮询事件

```typescript
const events = await client.beta.sessions.events.list(
  session.id,
);
for (const event of events.data) {
  console.log(`${event.type}: ${event.id}`);
}
```

---

## 带自定义工具的完整流式循环

```typescript
function runCustomTool(toolName: string, toolInput: unknown): string {
  if (toolName === "run_tests") {
    // 你的工具实现
    return "All tests passed.";
  }
  return `Unknown tool: ${toolName}`;
}

async function runSession(client: Anthropic, sessionId: string) {
  while (true) {
    const stream = await client.beta.sessions.events.stream(
      sessionId,
    );

    const toolCalls: Anthropic.Beta.Sessions.BetaManagedAgentsAgentCustomToolUseEvent[] = [];

    for await (const event of stream) {
      if (event.type === "agent.message") {
        for (const block of event.content) {
          if (block.type === "text") {
            process.stdout.write(block.text);
          }
        }
      } else if (event.type === "agent.custom_tool_use") {
        toolCalls.push(event);
      } else if (event.type === "session.status_idle") {
        break;
      } else if (event.type === "session.status_terminated") {
        return;
      }
    }

    if (toolCalls.length === 0) break;

    // 处理自定义工具调用
    const results = toolCalls.map((call) => ({
      type: "user.custom_tool_result" as const,
      custom_tool_use_id: call.id,
      content: [{ type: "text" as const, text: runCustomTool(call.name, call.input) }],
    }));

    await client.beta.sessions.events.send(
      sessionId,
      { events: results },
    );
  }
}
```

---

## 上传文件

```typescript
import fs from "fs";

const file = await client.beta.files.upload({
  file: fs.createReadStream("data.csv"),
  purpose: "agent",
});

// 在会话中使用
const session = await client.beta.sessions.create(
  {
    agent: { type: "agent", id: agent.id, version: agent.version },
    environment_id: environment.id,
    resources: [{ type: "file", file_id: file.id, mount_path: "/workspace/data.csv" }],
  },
);
```

---

## 列出和下载会话文件

列出 agent 在会话期间写入 `/mnt/session/outputs/` 的文件，然后下载。

```typescript
import fs from "fs";

// 列出与会话关联的文件
const files = await client.beta.files.list({
  scope_id: session.id,
  betas: ["managed-agents-2026-04-01"],
});
for (const f of files.data) {
  console.log(f.filename, f.size_bytes);

  // 下载并保存到磁盘
  const resp = await client.beta.files.download(f.id);
  const buffer = Buffer.from(await resp.arrayBuffer());
  fs.writeFileSync(f.filename, buffer);
}
```

> 💡 在 `session.status_idle` 和输出文件出现在 `files.list` 之间有短暂的索引延迟（约 1-3 秒）。如果列表为空，重试一两次。

---

## 会话管理

```typescript
// 获取会话详情
const session = await client.beta.sessions.retrieve("sesn_011CZxAbc123Def456");
console.log(session.status, session.usage);

// 列出会话
const sessions = await client.beta.sessions.list();

// 删除会话
await client.beta.sessions.delete("sesn_011CZxAbc123Def456");

// 归档会话
await client.beta.sessions.archive("sesn_011CZxAbc123Def456");
```

---

## MCP 服务器集成

```typescript
// Agent 声明 MCP 服务器（此处无认证 — 认证放在 vault 中）
const agent = await client.beta.agents.create({
  name: "MCP Agent",
  model: "claude-opus-4-8",
  mcp_servers: [
    { type: "url", name: "my-tools", url: "https://my-mcp-server.example.com/sse" },
  ],
  tools: [
    { type: "agent_toolset_20260401", default_config: { enabled: true } },
    { type: "mcp_toolset", mcp_server_name: "my-tools" },
  ],
});

// 会话挂载包含这些 MCP 服务器 URL 凭证的 vault
const session = await client.beta.sessions.create({
  agent: agent.id,
  environment_id: environment.id,
  vault_ids: [vault.id],
});
```

创建 vault 和添加凭证请参见 `shared/managed-agents-tools.md` §Vaults。
