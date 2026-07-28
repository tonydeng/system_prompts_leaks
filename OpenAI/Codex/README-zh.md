# Codex — 哪个文件对应哪个产品？

**`gpt-<version>.md` 文件是精简的 Codex 系统提示词**——每个模型在 Codex 客户端模型缓存中自带的基础指令。`codex-full.md` 是完整的运行时捕获（精简提示词 + Codex harness 周边的所有补充内容）。自 GPT-5.6 起变体可能不同：Sol 有自己的提示词，而 Terra 和 Luna 共享同一个。

| 文件 | 内容 |
|---|---|
| `gpt-5.6.md` | GPT-5.6 **Terra + Luna** 系统提示词（共享，字节级相同） |
| `gpt-5.6-sol.md` | GPT-5.6 **Sol** 系统提示词（2026 年 7 月从 Terra/Luna 分叉） |
| `gpt-5.5.md`、`gpt-5.4.md`、`gpt-5.4-mini.md`、`gpt-5.3-codex-spark.md` | 该模型的精简系统提示词 |
| `codex-full.md` | 完整运行时捕获（精简提示词 + harness 补充内容） |
| `codex-auto-review.md` | 隐藏的自动审批评审模型 |
| `personality_*.md` | `{{ personality }}` 槽位变体（friendly / pragmatic；该槽位在 5.6 中已移除） |
| `plan_mode.md`、`computer-use.md`、`control-chrome.md`、`control-in-app-browser.md` | 模式专属提示词 |
| `codex-desktop-realtime-voice-agent.md` | Codex 桌面实时语音助手 |
| `old/` | 已被取代的版本 |
