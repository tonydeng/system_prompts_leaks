> **说明**：本文件为英文原文（`managed-agents-client-patterns.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Managed Agents — 常用客户端模式

以下是在驱动 Managed Agent 会话时客户端需要编写的模式，基于可用的 SDK 示例。

代码示例为 TypeScript，其他语言遵循相同的结构；等效代码请参见 `{lang}/managed-agents/README.md`（cURL 和 C# 参见 `curl/managed-agents.md`）。

---

## 1. 无损流重连

**问题：** SSE 没有重放机制。如果连接在会话中途断开，简单的重连会从"当前时刻"重新打开流，你会静默丢失中间发出的所有事件。

**解决方案：** 重连时，先通过 `events.list()` 获取完整事件历史，再消费实时流，并在实时流追上时按事件 ID 去重。

```ts
const seenEventIds = new Set<string>()
const stream = await client.beta.sessions.events.stream(session.id)

// Stream is now open and buffering server-side. Read history first.
for await (const event of client.beta.sessions.events.list(session.id)) {
  seenEventIds.add(event.id)
  handle(event)
}

// Tail the live stream. Dedupe only gates handle() — terminal checks must run
// even for already-seen events, or a terminal event that was in the history
// response gets skipped by `continue` and the loop never exits.
for await (const event of stream) {
  if (!seenEventIds.has(event.id)) {
    seenEventIds.add(event.id)
    handle(event)
  }
  if (event.type === 'session.status_terminated') break
  if (event.type === 'session.status_idle' && event.stop_reason.type !== 'requires_action') break
}
```

---

## 2. `processed_at` — 排队 vs 已处理

流上的每个事件都携带 `processed_at`（ISO 8601 格式）。对于客户端发送的事件（`user.message`、`user.interrupt`、`user.tool_confirmation`、`user.custom_tool_result`），当事件已排队但尚未被 agent 处理时该字段为 `null`，agent 处理后填充时间戳。同一事件会在流上出现两次：一次 `processed_at: null`，一次带时间戳。

```ts
for await (const event of stream) {
  if (event.type === 'user.message') {
    if (event.processed_at == null) onQueued(event.id)
    else onProcessed(event.id, event.processed_at)
  }
}
```

用这个机制驱动你发送的任何内容的"待处理 → 已确认"UI 状态。你如何将本地渲染的乐观消息映射到服务端分配的 `event.id` 取决于具体应用（通常通过 `events.send()` 的返回值或 FIFO 顺序）。

---

## 3. 中断运行中的会话

将 `user.interrupt` 作为普通事件发送。会话会继续运行直到到达安全边界，然后转为空闲状态。

```ts
await client.beta.sessions.events.send(session.id, {
  events: [{ type: 'user.interrupt' }],
})

// Drain until the session is truly done — see Pattern 5 for the full gate.
for await (const event of stream) {
  if (event.type === 'session.status_terminated') break
  if (
    event.type === 'session.status_idle' &&
    event.stop_reason.type !== 'requires_action'
  ) break
}
```

参考：`interrupt.ts`，在看到 `span.model_request_start` 时发送中断，排空到空闲状态，然后通过 `sessions.retrieve()` 验证。

---

## 4. `tool_confirmation` 往返

当 agent 的 `permission_policy` 为 `{ type: 'always_ask' }` 时，对该工具的任何调用都会触发一个 `evaluated_permission === 'ask'` 的 `agent.tool_use` 事件，会话转为空闲等待决策。用 `user.tool_confirmation` 响应。

```ts
for await (const event of stream) {
  if (event.type === 'agent.tool_use' && event.evaluated_permission === 'ask') {
    await client.beta.sessions.events.send(session.id, {
      events: [{
        type: 'user.tool_confirmation',
        tool_use_id: event.id,         // not a toolu_ id — use event.id
        result: 'allow',               // or 'deny'
        // deny_message: '...',        // optional, only with result: 'deny'
      }],
    })
  }
}
```

关键点：
- `tool_use_id` 是 `event.id`（通常为 `sevt_...`），**不是** `toolu_...` ID。
- `result` 为 `'allow' | 'deny'`。使用 `deny_message` 告诉模型你*为什么*拒绝，该消息会回传给 agent。
- 多个待处理工具：每个 `evaluated_permission === 'ask'` 的 `agent.tool_use` 事件分别响应一次。

参考：`tool-permissions.ts`。

---

## 5. 正确的空闲退出门控

不要仅凭 `session.status_idle` 就退出循环。会话会短暂进入空闲状态，例如在并行工具执行之间、等待 `user.tool_confirmation` 时、或等待 `user.custom_tool_result` 时。应在空闲且 `stop_reason` 为终止状态时退出，或在 `session.status_terminated` 时退出。

```ts
for await (const event of stream) {
  handle(event)
  if (event.type === 'session.status_terminated') break
  if (event.type === 'session.status_idle') {
    if (event.stop_reason.type === 'requires_action') continue // waiting on you — handle it
    break // end_turn or retries_exhausted — both terminal
  }
}
```

`session.status_idle` 上的 `stop_reason.type` 值：
- `requires_action` — agent 正在等待客户端事件（工具确认、自定义工具结果）。处理它，不要退出。
- `retries_exhausted` — 终止失败。退出，然后通过 `sessions.retrieve()` 检查错误状态。
- `end_turn` — 正常完成。

---

## 6. 空闲后状态写入竞态

SSE 流在会话的可查询状态反映空闲状态之前就发出 `session.status_idle`。在空闲时退出并立即调用 `sessions.delete()` 或 `sessions.archive()` 的客户端会间歇性收到 400 错误"cannot delete/archive while running"。

清理前先轮询：

```ts
let s
for (let i = 0; i < 10; i++) {
  s = await client.beta.sessions.retrieve(session.id)
  if (s.status !== 'running') break
  await new Promise(r => setTimeout(r, 200))
}
if (s?.status !== 'running') {
  await client.beta.sessions.archive(session.id)
} // else: still running after 2s — don't archive, let it settle or escalate
```

---

## 7. 先开流，再发送

始终在发送启动事件**之前**打开流。否则 agent 可能在你的消费者连接之前就处理了事件并发出首批事件，导致你错过它们。

```ts
const stream = await client.beta.sessions.events.stream(session.id)
await client.beta.sessions.events.send(session.id, {
  events: [{ type: 'user.message', content: [{ type: 'text', text: 'Hello' }] }],
})
for await (const event of stream) { /* ... */ }
```

`Promise.all([stream, send])` 的写法也可以，但先开流更简单且效果相同，流在打开的那一刻就开始缓冲。

---

## 8. 文件挂载注意事项

**挂载的资源有不同的 `file_id`，与你上传的文件不同。** 会话创建时会生成一份会话作用域的副本。

```ts
const uploaded = await client.beta.files.upload({ file, purpose: 'agent_resource' })
// uploaded.id         → the original file
const session = await client.beta.sessions.create({
  /* ... */
  resources: [{ type: 'file', file_id: uploaded.id, mount_path: '/workspace/data.csv' }],
})
// session.resources[0].file_id !== uploaded.id  ← different IDs
```

通过 `files.delete(uploaded.id)` 删除原始文件；会话作用域的副本随会话一起被垃圾回收。`mount_path` 必须为绝对路径，参见 `shared/managed-agents-environments.md`。

---

## 9. 非 MCP API 和 CLI 的密钥 — 通过自定义工具保持在宿主侧

**问题：** 你希望 agent 调用第三方 API 或运行需要密钥（API key、token、服务账号凭证）的 CLI，但你不能或不想将密钥交给 vault。

**首先检查：** 对于云环境，首选方案是 vault `environment_variable` 凭证，agent 的 shell 看到的是不透明占位符，真实密钥在出口处替换。参见 `shared/managed-agents-tools.md` → Vaults。当该方案不适用时使用以下模式：**自托管沙箱**（尚不支持环境变量凭证）、客户端因本地格式校验拒绝占位符、密钥绝不能离开你的基础设施、或调用需要宿主侧二进制文件。

**解决方案：** 将需要认证的调用移到你的侧。在 agent 上声明一个自定义工具；当 agent 发出 `agent.custom_tool_use` 时，你的编排器（读取 SSE 流的进程）用自己的凭证执行调用，并用 `user.custom_tool_result` 响应。容器永远不会看到密钥。

```ts
// Agent template: declare the tool, no credentials
tools: [{ type: 'custom', name: 'linear_graphql', input_schema: { /* query, vars */ } }]

// Orchestrator: handle the call with host-side creds
for await (const event of stream) {
  if (event.type === 'agent.custom_tool_use' && event.name === 'linear_graphql') {
    const result = await linear.request(event.input.query, event.input.vars) // host's key
    await client.beta.sessions.events.send(session.id, {
      events: [{ type: 'user.custom_tool_result', tool_use_id: event.id, result }],
    })
  }
}
```

同样的结构适用于 `gh` CLI、本地评估脚本或任何需要宿主侧认证或二进制文件的场景。

**安全说明：** 这不会暴露公共端点。`agent.custom_tool_use` 到达你的编排器已经用你的 Anthropic API key 保持打开的 SSE 流上，`user.custom_tool_result` 通过同一 key 下的 `events.send()` 返回。你的编排器是客户端，不是服务端，没有未认证的监听。

**不要将 API key 嵌入系统提示或用户消息作为变通方案。** 提示和消息存储在会话的事件历史中，通过 `events.list()` 返回，并包含在压缩摘要中。放在那里的密钥会被持久化存储，在会话生命周期内可通过 API 读取。
