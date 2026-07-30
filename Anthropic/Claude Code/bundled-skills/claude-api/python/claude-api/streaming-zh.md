> **说明**：本文件为英文原文（`streaming.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 流式传输 — Python

## 快速开始

```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Write a story"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### 异步

```python
async with async_client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Write a story"}]
) as stream:
    async for text in stream.text_stream:
        print(text, end="", flush=True)
```

### 低层级：`stream=True`

`messages.stream()`（上文）是推荐的辅助方法——它累积状态并暴露 `text_stream` / `get_final_message()`。如果你只需要原始事件迭代器且想要更低的内存占用，改为向 `messages.create()` 传 `stream=True`：

```python
for event in client.messages.create(
    model="claude-opus-4-8",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Write a story"}],
    stream=True,
):
    print(event.type)
```

这种形式不会为你做最终消息累积。

---

## 处理不同内容类型

Claude 可能返回文本、思考块或工具使用。分别妥善处理：

> **Fable 5 / Opus 4.8 / Opus 4.7 / Opus 4.6：** 使用 `thinking: {type: "adaptive"}`。旧模型上改用 `thinking: {type: "enabled", budget_tokens: N}`。

```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=64000,
    thinking={"type": "adaptive", "display": "summarized"},  # display 需显式开启：默认在 Fable 5 / Mythos 5 / Opus 4.8 / 4.7 上省略（空思考文本）
    messages=[{"role": "user", "content": "Analyze this problem"}]
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            if event.content_block.type == "thinking":
                print("\n[Thinking...]")
            elif event.content_block.type == "text":
                print("\n[Response:]")

        elif event.type == "content_block_delta":
            if event.delta.type == "thinking_delta":
                print(event.delta.thinking, end="", flush=True)
            elif event.delta.type == "text_delta":
                print(event.delta.text, end="", flush=True)
```

---

## 工具使用中的流式传输

Python 工具运行器支持流式传输：向 `client.beta.messages.tool_runner(...)` 传 `stream=True`，每次迭代产出一个流，你逐事件消费，用 `get_final_message()` 获取每轮累积的消息（参见 `shared/tool-use-concepts.md` → Tool Runner vs Manual Loop）。仅当你不使用工具运行器且需要带工具的逐 token 流式传输时，才使用下面的手动循环模式：

```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=64000,
    tools=tools,
    messages=messages
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

    response = stream.get_final_message()
    # 如果 response.stop_reason == "tool_use" 则继续工具执行
```

---

## 获取最终消息

```python
with client.messages.stream(
    model="claude-opus-4-8",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Hello"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

    # 流式传输后获取完整消息
    final_message = stream.get_final_message()
    print(f"\n\nTokens used: {final_message.usage.output_tokens}")
```

---

## 带进度更新的流式传输

```python
def stream_with_progress(client, **kwargs):
    """带进度更新的流式响应。"""
    total_tokens = 0
    content_parts = []

    with client.messages.stream(**kwargs) as stream:
        for event in stream:
            if event.type == "content_block_delta":
                if event.delta.type == "text_delta":
                    text = event.delta.text
                    content_parts.append(text)
                    print(text, end="", flush=True)

            elif event.type == "message_delta":
                if event.usage and event.usage.output_tokens is not None:
                    total_tokens = event.usage.output_tokens

        final_message = stream.get_final_message()

    print(f"\n\n[Tokens used: {total_tokens}]")
    return "".join(content_parts)
```

---

## 流式传输中的错误处理

```python
try:
    with client.messages.stream(
        model="claude-opus-4-8",
        max_tokens=64000,
        messages=[{"role": "user", "content": "Write a story"}]
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
except anthropic.APIConnectionError:
    print("\nConnection lost. Please retry.")
except anthropic.RateLimitError:
    print("\nRate limited. Please wait and retry.")
except anthropic.APIStatusError as e:
    print(f"\nAPI error: {e.status_code}")
```

---

## 流事件类型

| 事件类型               | 描述                       | 触发时机                          |
| --------------------- | ------------------------- | --------------------------------- |
| `message_start`       | 包含消息元数据              | 开头一次                           |
| `content_block_start` | 新内容块开始                | 文本/tool_use 块开始时             |
| `content_block_delta` | 增量内容更新                | 每个 token/块                       |
| `content_block_stop`  | 内容块完成                  | 块结束时                            |
| `message_delta`       | 消息级更新                  | 包含 `stop_reason`、usage          |
| `message_stop`        | 消息完成                   | 结尾一次                            |

## 最佳实践

1. **始终刷新输出** —— 使用 `flush=True` 立即显示 token
2. **处理部分响应** —— 如果流被中断，你可能有不完整的内容
3. **跟踪 token 用量** —— `message_delta` 事件包含用量信息
4. **使用超时** —— 为你的应用设置适当的超时
5. **默认流式传输** —— 使用 `.get_final_message()` 即使在流式传输时也能获取完整响应，在无需处理单个事件的情况下提供超时保护
6. **大 `max_tokens` 不流式传输会抛 `ValueError`** —— SDK 拒绝其估计会超过约 10 分钟的非流式请求（空闲连接会断开）。传 `stream=True` / 使用 `messages.stream()`，或显式覆盖 `timeout` 来抑制此保护。
