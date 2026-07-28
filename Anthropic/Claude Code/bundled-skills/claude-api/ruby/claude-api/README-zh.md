# Claude API — Ruby

> **注意：** Ruby SDK 支持 Claude API。工具运行器可通过 `client.beta.messages.tool_runner()` 在 beta 中使用。Agent SDK 尚未支持 Ruby。

## 安装

```bash
gem install anthropic
```

## 客户端初始化

```ruby
require "anthropic"

# 默认（使用 ANTHROPIC_API_KEY 环境变量）
client = Anthropic::Client.new

# 显式 API 密钥
client = Anthropic::Client.new(api_key: "your-api-key")
```

---

## 基本消息请求

```ruby
message = client.messages.create(
  model: :"claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    { role: "user", content: "What is the capital of France?" }
  ]
)
# content 是一个多态块对象数组（TextBlock、ThinkingBlock、
# ToolUseBlock、...）。.type 是一个 Symbol — 用 :text 比较，而非 "text"。
# .text 在非 TextBlock 条目上会引发 NoMethodError。
message.content.each do |block|
  puts block.text if block.type == :text
end
```

---

## 扩展思考

> **Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考。`budget_tokens` 在 Fable 5、Opus 4.8 和 4.7 上已移除（如果发送会返回 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。
> **较旧的模型：** 使用 `thinking: { type: "enabled", budget_tokens: N }`（必须 < `max_tokens`，最小 1024）。

```ruby
message = client.messages.create(
  model: :"claude-opus-4-8",
  max_tokens: 16000,
  thinking: { type: "adaptive" },
  messages: [{ role: "user", content: "Solve: 27 * 453" }]
)

message.content.each do |block|
  case block.type
  when :thinking then puts "Thinking: #{block.thinking}"
  when :text then puts "Response: #{block.text}"
  end
end
```

---

## 提示词缓存

`system_:`（末尾下划线 — 避免遮蔽 `Kernel#system`）接受一个文本块数组；在最后一个块上设置 `cache_control`。普通哈希通过 `OrHash` 类型别名工作。关于放置模式和静默失效项审计清单，参见 `shared/prompt-caching.md`。

```ruby
message = client.messages.create(
  model: :"claude-opus-4-8",
  max_tokens: 16000,
  system_: [
    { type: "text", text: long_system_prompt, cache_control: { type: "ephemeral" } }
  ],
  messages: [{ role: "user", content: "Summarize the key points" }]
)
```

1 小时 TTL：`cache_control: { type: "ephemeral", ttl: "1h" }`。`messages.create` 还有一个顶层 `cache_control:` 会自动放置在最后一个可缓存块上。

通过 `message.usage.cache_creation_input_tokens` / `message.usage.cache_read_input_tokens` 验证命中。

---

## 停止详情

当 `stop_reason` 为 `:refusal` 时，响应包含结构化的 `stop_details`：

```ruby
if message.stop_reason == :refusal && message.stop_details
  puts "Category: #{message.stop_details.category}"     # 例如 :cyber、:bio、:reasoning_extraction、:frontier_llm 或 nil — 参见文档获取完整集合
  puts "Explanation: #{message.stop_details.explanation}"
end
```

**拒绝回退（Claude Fable 5）— 默认选择启用。** 回退是选择启用的：没有它们，被拒绝的请求会直接停止。新的 `claude-fable-5` 代码应默认包含服务端 `fallbacks` 参数（beta 头 `server-side-fallback-2026-06-01`，在 beta messages 调用上 `fallbacks: [{model: "claude-opus-4-8"}]`）。确切的 Ruby 绑定（以及用于不支持服务端回退的提供商的客户端中间件）此处不予记录 — 从 `shared/live-sources.md` WebFetch Ruby SDK 仓库的 `examples/`；完整语义见 `shared/model-migration.md` → 迁移到 Claude Fable 5 → `refusal` 停止原因。

---

## Beta 功能

`betas:` 仅在 `client.beta.messages.create` 上有效，不在非 beta 路径上。

### 任务预算

```ruby
response = client.beta.messages.create(
  model: :"claude-opus-4-8",
  max_tokens: 16000,
  output_config: { task_budget: { type: :tokens, total: 64_000 } },
  tools: [...],
  messages: [...],
  betas: ["task-budgets-2026-03-13"]
)
```

---

## 错误类型

`APIStatusError` 暴露了一个 `.type` 字段用于程序化错误分类：

```ruby
begin
  client.messages.create(...)
rescue Anthropic::Errors::APIStatusError => e
  puts e.type  # :rate_limit_error、:overloaded_error 等
end
```
