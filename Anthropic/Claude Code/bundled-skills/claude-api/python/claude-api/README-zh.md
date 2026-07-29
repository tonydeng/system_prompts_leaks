# Claude API — Python

## 安装

```bash
pip install anthropic
```

## 客户端初始化

```python
import anthropic

# 默认 — 从环境中解析凭据：
# ANTHROPIC_API_KEY，或 ANTHROPIC_AUTH_TOKEN，或 `ant auth login` profile。
# 本地开发优先使用此方式；不要硬编码 key。
client = anthropic.Anthropic()

# 显式 API key（仅当你必须注入特定 key 时）
client = anthropic.Anthropic(api_key="your-api-key")

# 异步客户端
async_client = anthropic.AsyncAnthropic()
```

---

## 客户端配置

### 单次请求覆盖

使用 `with_options()` 为单次调用覆盖客户端设置，而不改变客户端：

```python
client.with_options(timeout=5.0, max_retries=5).messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
```

### 超时

默认请求超时为 10 分钟。传入浮点数（秒）或 `httpx.Timeout` 进行精细控制。超时时 SDK 抛出 `anthropic.APITimeoutError`（并按 `max_retries` 重试）。

```python
import httpx

client = anthropic.Anthropic(timeout=20.0)
client = anthropic.Anthropic(
    timeout=httpx.Timeout(60.0, read=5.0, write=10.0, connect=2.0),
)
```

### 重试

SDK 自动重试连接错误、408、409、429 和 ≥500，使用指数退避（默认 2 次重试）。在客户端或通过 `with_options()` 设置 `max_retries`；`max_retries=0` 禁用。

### 异步性能（aiohttp 后端）

对于高并发异步工作负载，安装 `anthropic[aiohttp]` 并传入 `DefaultAioHttpClient` 代替默认的 httpx 后端：

```python
from anthropic import AsyncAnthropic, DefaultAioHttpClient

async with AsyncAnthropic(http_client=DefaultAioHttpClient()) as client:
    ...
```

### 自定义 HTTP 客户端（代理、base URL）

使用 `DefaultHttpxClient` / `DefaultAsyncHttpxClient`，而非原始 `httpx.Client`，以保留 SDK 的默认超时和连接限制：

```python
from anthropic import Anthropic, DefaultHttpxClient

client = Anthropic(
    base_url="http://my.test.server.example.com:8083",  # 或 ANTHROPIC_BASE_URL 环境变量
    http_client=DefaultHttpxClient(proxy="http://my.test.proxy.example.com"),
)
```

### 日志

设置 `ANTHROPIC_LOG=debug`（或 `info`）通过标准 `logging` 模块启用 SDK 日志。

---

## 基本消息请求

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    messages=[
        {"role": "user", "content": "What is the capital of France?"}
    ]
)
# response.content 是内容块对象列表（TextBlock、ThinkingBlock、
# ToolUseBlock、…）。访问 .text 前先检查 .type。
for block in response.content:
    if block.type == "text":
        print(block.text)
```

---

## 系统提示

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    system="You are a helpful coding assistant. Always provide examples in Python.",
    messages=[{"role": "user", "content": "How do I read a JSON file?"}]
)
```

### 对话中系统消息（模型门控）

对于对话中途到达的操作者指令（模式切换、注入状态），将 `{"role": "system", ...}` 追加到 `messages` 而非编辑顶层 `system`，这保留了缓存前缀并携带操作者权威。必须跟在用户消息（或以服务端工具使用结尾的 `assistant` 消息）之后，且必须是 `messages` 中的最后一条或后跟一个 `assistant` 回合；不能是 `messages[0]`。不支持的模型返回 400（`role 'system' is not supported on this model`）。关于何时使用此方式 vs 顶层 `system`，参见 `shared/prompt-caching.md`。

```python
response = client.messages.create(
    model=MODEL_ID,  # must support mid-conversation system messages
    max_tokens=16000,
    system=[{"type": "text", "text": STABLE_SYSTEM, "cache_control": {"type": "ephemeral"}}],
    messages=history + [
        {"role": "user", "content": user_message},
        {"role": "system", "content": "Terse mode enabled — keep responses under 40 words."},
    ],
)  # No beta header needed — use regular client.messages.create
```

---

## 视觉（图像）

### Base64

```python
import base64

with open("image.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/png",
                    "data": image_data
                }
            },
            {"type": "text", "text": "What's in this image?"}
        ]
    }]
)
```

### URL

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "url",
                    "url": "https://example.com/image.png"
                }
            },
            {"type": "text", "text": "Describe this image"}
        ]
    }]
)
```

---

## Prompt Caching

缓存大型上下文以降低成本（最高节省 90%）。**缓存是前缀匹配** — 前缀中任何位置的字节更改都会使其后的所有内容失效。关于放置模式、架构指导（冻结的系统提示、确定性工具顺序、易变内容放在哪里）和静默失效因子审计清单，请阅读 `shared/prompt-caching.md`。

### 自动缓存（推荐）

使用顶层 `cache_control` 自动缓存请求中最后一个可缓存的块，无需标注各个内容块：

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    cache_control={"type": "ephemeral"},  # auto-caches the last cacheable block
    system="You are an expert on this large document...",
    messages=[{"role": "user", "content": "Summarize the key points"}]
)
```

### 手动缓存控制

对于精细控制，在特定内容块上添加 `cache_control`：

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    system=[{
        "type": "text",
        "text": "You are an expert on this large document...",
        "cache_control": {"type": "ephemeral"}  # default TTL is 5 minutes
    }],
    messages=[{"role": "user", "content": "Summarize the key points"}]
)

# With explicit TTL (time-to-live)
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    system=[{
        "type": "text",
        "text": "You are an expert on this large document...",
        "cache_control": {"type": "ephemeral", "ttl": "1h"}  # 1 hour TTL
    }],
    messages=[{"role": "user", "content": "Summarize the key points"}]
)
```

### 验证缓存命中

```python
print(response.usage.cache_creation_input_tokens)  # tokens written to cache (~1.25x cost)
print(response.usage.cache_read_input_tokens)      # tokens served from cache (~0.1x cost)
print(response.usage.input_tokens)                 # uncached tokens (full cost)
```

如果 `cache_read_input_tokens` 在重复的相同前缀请求中为零，说明有静默失效因子在起作用 — 系统提示中的 `datetime.now()` 或 UUID、未排序的 `json.dumps()`、或变化的工具集。完整审计表参见 `shared/prompt-caching.md`。

---

## 扩展思考

> **Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：**使用自适应思考。`budget_tokens` 在 Fable 5、Opus 4.8 和 4.7 上已移除（如果发送返回 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。
> **旧模型：**使用 `thinking: {type: "enabled", budget_tokens: N}`（必须 < `max_tokens`，最小 1024）。

```python
# Fable 5 / Opus 4.8 / 4.7 / 4.6: adaptive thinking (recommended)
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    thinking={"type": "adaptive", "display": "summarized"},  # display opt-in: default is omitted (empty thinking text) on Fable 5 / Mythos 5 / Opus 4.8 / 4.7
    output_config={"effort": "high"},  # low | medium | high | max
    messages=[{"role": "user", "content": "Solve this step by step..."}]
)

# Access thinking and response
for block in response.content:
    if block.type == "thinking":
        print(f"Thinking: {block.thinking}")
    elif block.type == "text":
        print(f"Response: {block.text}")
```

---

## 错误处理

```python
import anthropic

try:
    response = client.messages.create(...)
except anthropic.BadRequestError as e:
    print(f"Bad request: {e.message}")
except anthropic.AuthenticationError:
    print("Invalid API key")
except anthropic.PermissionDeniedError:
    print("API key lacks required permissions")
except anthropic.NotFoundError:
    print("Invalid model or endpoint")
except anthropic.RateLimitError as e:
    retry_after = int(e.response.headers.get("retry-after", "60"))
    print(f"Rate limited. Retry after {retry_after}s.")
except anthropic.APIStatusError as e:
    if e.status_code >= 500:
        print(f"Server error ({e.status_code}). Retry later.")
    else:
        print(f"API error: {e.message}")
except anthropic.APIConnectionError:
    print("Network error. Check internet connection.")
```

---

## 响应辅助方法

每个响应对象都暴露 `_request_id`（从 `request-id` header 填充），向 Anthropic 报告故障时记录它。尽管有下划线前缀，此属性是公开的。

```python
message = client.messages.create(...)
print(message._request_id)       # req_018EeWyXxfu5pfWkrYcMdjWG
print(message.to_json())          # serialize the Pydantic model
print(message.to_dict())          # plain dict
```

要访问原始 header 或其他响应元数据，使用 `.with_raw_response`：

```python
raw = client.messages.with_raw_response.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
print(raw.headers.get("request-id"))
message = raw.parse()  # the Message object messages.create() would have returned
```

---

## 多轮对话

API 是无状态的 — 每次发送完整对话历史。

```python
class ConversationManager:
    """Manage multi-turn conversations with the Claude API."""

    def __init__(self, client: anthropic.Anthropic, model: str, system: str = None):
        self.client = client
        self.model = model
        self.system = system
        self.messages = []

    def send(self, user_message: str, **kwargs) -> str:
        """Send a message and get a response."""
        self.messages.append({"role": "user", "content": user_message})

        response = self.client.messages.create(
            model=self.model,
            max_tokens=kwargs.get("max_tokens", 16000),
            system=self.system,
            messages=self.messages,
            **kwargs
        )

        assistant_message = next(
            (b.text for b in response.content if b.type == "text"), ""
        )
        self.messages.append({"role": "assistant", "content": assistant_message})

        return assistant_message

# Usage
conversation = ConversationManager(
    client=anthropic.Anthropic(),
    model="claude-opus-4-8",
    system="You are a helpful assistant."
)

response1 = conversation.send("My name is Alice.")
response2 = conversation.send("What's my name?")  # Claude remembers "Alice"
```

**规则：**

- 允许连续相同角色的消息 — API 将它们合并为单个回合
- 第一条消息必须是 `user`
- 支持的模型上允许对话中途出现 `role: "system"` 消息（无需 beta header），参见上方 § 对话中系统消息

---

### 压缩（长对话）

> **Beta，Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6。** 当对话接近 200K 上下文窗口时，压缩会在服务端自动摘要早期上下文。API 返回一个 `compaction` 块；你必须在后续请求中传回它 — 追加 `response.content`，而非仅文本。

```python
import anthropic

client = anthropic.Anthropic()
messages = []

def chat(user_message: str) -> str:
    messages.append({"role": "user", "content": user_message})

    response = client.beta.messages.create(
        betas=["compact-2026-01-12"],
        model="claude-opus-4-8",
        max_tokens=16000,
        messages=messages,
        context_management={
            "edits": [{"type": "compact_20260112"}]
        }
    )

    # Append full content — compaction blocks must be preserved
    messages.append({"role": "assistant", "content": response.content})

    return next(block.text for block in response.content if block.type == "text")

# Compaction triggers automatically when context grows large
print(chat("Help me build a Python web scraper"))
print(chat("Add support for JavaScript-rendered pages"))
print(chat("Now add rate limiting and error handling"))
```

---

## 停止原因

响应中的 `stop_reason` 字段指示模型为何停止生成：

| 值 | 含义 |
|-------|---------|
| `end_turn` | Claude 自然完成了响应 |
| `max_tokens` | 达到 `max_tokens` 限制 — 增加它或使用流式 |
| `stop_sequence` | 命中自定义停止序列 |
| `tool_use` | Claude 想调用工具 — 执行它并继续 |
| `pause_turn` | 模型暂停，可以恢复（智能体流程） |
| `refusal` | Claude 因安全原因拒绝 — 检查 `stop_details` |

### 结构化停止详情

当 `stop_reason` 为 `"refusal"` 时，响应包含一个 `stop_details` 对象，带有关于拒绝的结构化信息：

```python
if response.stop_reason == "refusal" and response.stop_details:
    print(f"Category: {response.stop_details.category}")   # e.g. "cyber", "bio", "reasoning_extraction", "frontier_llm", or None — see docs for the full set
    print(f"Explanation: {response.stop_details.explanation}")
```

### 拒绝回退（Claude Fable 5），默认选择加入

回退是**可选的**：不加入则被拒绝的请求直接停止。在 `claude-fable-5` 代码中默认包含服务端 `fallbacks` 参数，在策略拒绝时 API 会在同一调用中在回退模型上重新运行相同请求。输出前的拒绝不计费（流中途的拒绝计费已流式的部分）；救援按回退模型自身的费率计费，缓存重新定价自动应用。

```python
response = client.beta.messages.create(
    model="claude-fable-5",
    max_tokens=16000,
    betas=["server-side-fallback-2026-06-01"],
    fallbacks=[{"model": "claude-opus-4-8"}],
    messages=[{"role": "user", "content": "..."}],
)

# Switch points: one fallback block per model that ran and declined this turn
for block in response.content:
    if block.type == "fallback":
        print(f"{block.from_.model} declined; {block.to.model} continued")

# Served-by signal — covers sticky turns, which carry no fallback block.
# Pair with stop_reason: the fallback model can itself refuse.
fallback_ran = any(
    entry.type == "fallback_message" for entry in response.usage.iterations or []
)
if fallback_ran and response.stop_reason != "refusal":
    print(f"Served by {response.model}")
```

最终响应上的 `stop_reason: "refusal"` 意味着整个链拒绝。header 必须恰好是 `server-side-fallback-2026-06-01`；该参数在 Batches API 上被拒绝，在 Amazon Bedrock、Vertex AI 和 Microsoft Foundry 上不可用，在这些平台上改为在客户端注册 `BetaRefusalFallbackMiddleware`。完整语义（粘性路由、计费、流式、回退回合回显）：`shared/model-migration.md` → Migrating to Claude Fable 5 → `refusal` stop reason。

---

## 成本优化策略

### 1. 对重复上下文使用 Prompt Caching

```python
# Automatic caching (simplest — caches the last cacheable block)
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=16000,
    cache_control={"type": "ephemeral"},
    system=large_document_text,  # e.g., 50KB of context
    messages=[{"role": "user", "content": "Summarize the key points"}]
)

# First request: full cost
# Subsequent requests: ~90% cheaper for cached portion
```

### 2. 选择合适的模型

```python
# Default to Opus for most tasks
response = client.messages.create(
    model="claude-opus-4-8",  # $5.00/$25.00 per 1M tokens
    max_tokens=16000,
    messages=[{"role": "user", "content": "Explain quantum computing"}]
)

# Use Sonnet for high-volume production workloads
standard_response = client.messages.create(
    model="claude-sonnet-5",  # $3.00/$15.00 per 1M tokens
    max_tokens=16000,
    messages=[{"role": "user", "content": "Summarize this document"}]
)

# Use Haiku only for simple, speed-critical tasks
simple_response = client.messages.create(
    model="claude-haiku-4-5",  # $1.00/$5.00 per 1M tokens
    max_tokens=256,
    messages=[{"role": "user", "content": "Classify this as positive or negative"}]
)
```

### 3. 请求前使用 Token 计数

```python
count_response = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=messages,
    system=system
)

estimated_input_cost = count_response.input_tokens * 0.000005  # $5/1M tokens
print(f"Estimated input cost: ${estimated_input_cost:.4f}")
```

---

## 指数退避重试

> **注意：** Anthropic SDK 自动以指数退避重试速率限制（429）和服务器错误（5xx）。你可以通过 `max_retries`（默认：2）配置。仅在你需要 SDK 提供之外的行为时才实现自定义重试逻辑。

```python
import time
import random
import anthropic

def call_with_retry(
    client: anthropic.Anthropic,
    max_retries: int = 5,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    **kwargs
):
    """Call the API with exponential backoff retry."""
    last_exception = None

    for attempt in range(max_retries):
        try:
            return client.messages.create(**kwargs)
        except anthropic.RateLimitError as e:
            last_exception = e
        except anthropic.APIStatusError as e:
            if e.status_code >= 500:
                last_exception = e
            else:
                raise  # Client errors (4xx except 429) should not be retried

        delay = min(base_delay * (2 ** attempt) + random.uniform(0, 1), max_delay)
        print(f"Retry {attempt + 1}/{max_retries} after {delay:.1f}s")
        time.sleep(delay)

    raise last_exception
```
