> **说明**：本文件为英文原文（`claude-api-in-prototypes.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 原型中的 Claude API

你的 HTML artifact 可以通过内置助手调用 Claude。无需 SDK 或 API 密钥。

```html
<script>
(async () => {
  const text = await window.claude.complete("Summarize this: ...");
  // 或使用 messages 数组：
  const text2 = await window.claude.complete({
    messages: [{ role: 'user', content: '...' }],
  });
})();
</script>
```

调用默认使用 `claude-haiku-4-5`，输出上限 1024 token。body 还可设置 `model`（仅 haiku/sonnet 系列）、`max_tokens`（最高 32000）、`system`、`tool_choice` 和客户端 `tools`——标准 Messages API 形状，区别是每个 tool 还携带 `run: async (input) => string`，助手在页面内执行工具调用并循环（最多 8 次模型调用），以最终文本解析。处理程序抛出的异常会变成 is_error 的 tool_results。服务端工具（web 搜索等）被拒绝；不支持流式输出；每用户限速 15 次/分钟（含循环迭代）。共享 artifact 在查看者的配额下运行。
