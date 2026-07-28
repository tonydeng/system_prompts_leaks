> **说明**：本文件为英文原文（`tool-use.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 工具使用 — Ruby

概念概述（工具定义、工具选择、技巧）见 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用

Ruby SDK 通过原始 JSON schema 定义支持工具使用，并提供 beta tool runner 用于自动工具执行。

### Tool Runner（Beta）

```ruby
class GetWeatherInput < Anthropic::BaseModel
  required :location, String, doc: "City and state, e.g. San Francisco, CA"
end

class GetWeather < Anthropic::BaseTool
  doc "Get the current weather for a location"

  input_schema GetWeatherInput

  def call(input)
    "The weather in #{input.location} is sunny and 72°F."
  end
end

client.beta.messages.tool_runner(
  model: :"claude-opus-4-8",
  max_tokens: 16000,
  tools: [GetWeather.new],
  messages: [{ role: "user", content: "What's the weather in San Francisco?" }]
).each_message do |message|
  puts message.content
end
```

### 手动循环

工具定义格式和智能体循环模式见 [shared tool use concepts](../../shared/tool-use-concepts.md)。

---
