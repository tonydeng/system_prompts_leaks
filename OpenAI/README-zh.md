# OpenAI — 各文件对应哪个产品？

**`gpt-<version>-thinking/instant.md` 文件是 ChatGPT 应用的系统提示词**，即 chatgpt.com 为该模型提供的内容。以 `-api.md` 结尾的文件是 OpenAI 在原始 API 调用中注入的隐藏系统消息（未公开文档）。`Codex/` 是 Codex CLI/智能体。

| 文件模式 | 产品 |
|---|---|
| `gpt-5.6-sol-extra-high.md`, `gpt-5.5-thinking.md`, `gpt-5.5-instant.md`, … | 该模型的 **ChatGPT** 应用系统提示词 |
| `chatgpt-4.5.md`, `chatgpt-atlas.md`, `chatgpt-gpt-5-agent-mode.md` | ChatGPT 应用（旧版捕获 / Atlas 浏览器 / 智能体模式） |
| `gpt-*-api.md` | **API** 调用时注入的隐藏系统消息 |
| `Codex/` | Codex CLI / 编码智能体 |
| `gpt-4o.md` | ChatGPT 4o（包含弃用自我葬礼协议，L226+） |
| `gpt-5-*-personality.md`, `gpt-5.1-*.md` | ChatGPT 人格变体 |
| `tool-*.md` | ChatGPT 工具特定片段 |
| `Old/` | 已被取代的版本 · `deprecated/` — 已下线的功能 |
