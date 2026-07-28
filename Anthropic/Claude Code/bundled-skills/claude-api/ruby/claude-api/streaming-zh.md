> **说明**：本文件为英文原文（`streaming.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 流式输出 — Ruby

## 流式输出

```ruby
stream = client.messages.stream(
  model: :"claude-opus-4-8",
  max_tokens: 64000,
  messages: [{ role: "user", content: "Write a haiku" }]
)

stream.text.each { |text| print(text) }
```

---
