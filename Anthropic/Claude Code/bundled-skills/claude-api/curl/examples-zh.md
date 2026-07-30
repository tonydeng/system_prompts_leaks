> **说明**：本文件为英文原文（`examples.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Claude API — cURL / 原始 HTTP

当用户需要原始 HTTP 请求或使用没有官方 SDK 的语言时，使用这些示例。

## 设置

```bash
export ANTHROPIC_API_KEY="your-api-key"
```

---

## 基本消息请求

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 16000,
    "messages": [
      {"role": "user", "content": "What is the capital of France?"}
    ]
  }'
```

### 解析响应

使用 `jq` 从 JSON 响应中提取字段。不要使用 `grep`/`sed`，JSON 字符串可以包含任何字符，正则解析会在引号、转义或多行内容上出错。

```bash
# Capture the response, then extract fields
response=$(curl -s https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"claude-opus-4-8","max_tokens":16000,"messages":[{"role":"user","content":"Hello"}]}')

# Print the first text block (-r strips the JSON quotes)
echo "$response" | jq -r '.content[0].text'

# Read usage fields
input_tokens=$(echo "$response" | jq -r '.usage.input_tokens')
output_tokens=$(echo "$response" | jq -r '.usage.output_tokens')

# Read stop reason (for tool-use loops)
stop_reason=$(echo "$response" | jq -r '.stop_reason')

# Extract all text blocks (content is an array; filter to type=="text")
echo "$response" | jq -r '.content[] | select(.type == "text") | .text'
```


---

## 流式传输（SSE）

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 64000,
    "stream": true,
    "messages": [{"role": "user", "content": "Write a haiku"}]
  }'
```

响应是 Server-Sent Events 流：

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_...","type":"message",...}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":12}}

event: message_stop
data: {"type":"message_stop"}
```

---

## 工具使用

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 16000,
    "tools": [{
      "name": "get_weather",
      "description": "Get current weather for a location",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": {"type": "string", "description": "City name"}
        },
        "required": ["location"]
      }
    }],
    "messages": [{"role": "user", "content": "What is the weather in Paris?"}]
  }'
```

当 Claude 响应一个 `tool_use` 块时，将结果发回：

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 16000,
    "tools": [{
      "name": "get_weather",
      "description": "Get current weather for a location",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": {"type": "string", "description": "City name"}
        },
        "required": ["location"]
      }
    }],
    "messages": [
      {"role": "user", "content": "What is the weather in Paris?"},
      {"role": "assistant", "content": [
        {"type": "text", "text": "Let me check the weather."},
        {"type": "tool_use", "id": "toolu_abc123", "name": "get_weather", "input": {"location": "Paris"}}
      ]},
      {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_abc123", "content": "72°F and sunny"}
      ]}
    ]
  }'
```

---

## 提示词缓存

将 `cache_control` 放在稳定前缀的最后一个块上。放置模式和静默失效审计清单请参见 `shared/prompt-caching.md`。

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 16000,
    "system": [
      {"type": "text", "text": "<large shared prompt...>", "cache_control": {"type": "ephemeral"}}
    ],
    "messages": [{"role": "user", "content": "Summarize the key points"}]
  }'
```

1 小时 TTL：`"cache_control": {"type": "ephemeral", "ttl": "1h"}`。请求体上的顶层 `"cache_control"` 自动放置在最后一个可缓存块上。通过响应的 `usage.cache_creation_input_tokens` / `usage.cache_read_input_tokens` 字段验证命中。

---

## 扩展思考

> **Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考。`budget_tokens` 在 Fable 5、Opus 4.8 和 4.7 上已移除（发送则返回 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。
> **较旧模型：** 使用 `"type": "enabled"` 加 `"budget_tokens": N`（必须 < `max_tokens`，最小 1024）。

```bash
# Fable 5 / Opus 4.8 / 4.7 / 4.6: adaptive thinking (recommended)
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 16000,
    "thinking": {
      "type": "adaptive",
      "display": "summarized"
    },
    "output_config": {
      "effort": "high"
    },
    "messages": [{"role": "user", "content": "Solve this step by step..."}]
  }'
```

---

## 拒绝回退（Claude Fable 5）— 默认 opt in

在 `claude-fable-5` 上，安全分类器可能拒绝请求（HTTP 200 且 `stop_reason: "refusal"`）。回退是**可选的**：不启用时请求直接停止。默认包含 `fallbacks` 参数及其 beta 请求头，在策略拒绝时 API 在同一调用内用回退模型重新运行同一请求。输出前的拒绝不计费（流中途的拒绝计费已流式的部分）；救援按回退模型自身费率计费。

```bash
response=$(curl -s https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: server-side-fallback-2026-06-01" \
  -d '{
    "model": "claude-fable-5",
    "max_tokens": 16000,
    "fallbacks": [{"model": "claude-opus-4-8"}],
    "messages": [{"role": "user", "content": "Hello"}]
  }')

# Which model produced the message
echo "$response" | jq -r '.model'

# Refusal on the final response means the whole chain refused
echo "$response" | jq -r '.stop_reason'

# Switch points: one fallback block per model that ran and declined this turn
echo "$response" | jq -r '.content[] | select(.type == "fallback") | "\(.from.model) declined; \(.to.model) continued"'

# Served-by signal — covers sticky turns, which carry no fallback block.
# Pair with stop_reason: the fallback model can itself refuse.
if [ "$(echo "$response" | jq -r '.stop_reason')" != "refusal" ] && \
   echo "$response" | jq -e '[.usage.iterations[]? | select(.type == "fallback_message")] | length > 0' > /dev/null; then
  echo "fallback model served this turn"
fi
```

请求头必须恰好是 `server-side-fallback-2026-06-01`。该参数在 Batches API 上被拒绝，且在 Amazon Bedrock、Vertex AI 和 Microsoft Foundry 上不可用。完整语义（粘性路由、计费、流式传输、回退轮次回显）：`shared/model-migration.md` → 迁移到 Claude Fable 5 → `refusal` 停止原因。

---

## 必需请求头

| 请求头             | 值                 | 说明                       |
| ------------------- | ------------------ | -------------------------- |
| `Content-Type`      | `application/json` | 必需                       |
| `x-api-key`         | 你的 API 密钥       | 身份认证                   |
| `anthropic-version` | `2023-06-01`       | API 版本                   |
| `anthropic-beta`    | Beta 功能 ID       | Beta 功能必需              |
