# Anthropic —— 哪个文件对应哪个产品？

**本目录下裸文件名 `claude-<model>.md` 即 claude.ai 系统提示词** —— 该模型在 Claude 网页/移动应用（claude.ai）中被服务时使用的提示词。它们不是 API 提示词（Claude API 不注入系统提示词），也不是 Claude Code 提示词。

| 文件名模式 | 产品 |
|---|---|
| `claude-fable-5.md`、`claude-opus-4.8.md`、`claude-sonnet-5.md`、…… | **claude.ai** 应用中该模型的系统提示词 |
| `claude-*-no-tools.md` | claude.ai 禁用工具版本 |
| `Claude Code/` | Claude Code（命令行/智能体框架） |
| `claude-design.md` | Claude Design |
| `claude-cowork.md`、`claude-cowork-dispatch.md` | Claude Cowork |
| `claude-for-excel.md`、`claude-for-word.md`、`claude-in-powerpoint.md` | Microsoft 365 中的 Claude |
| `claude-in-chrome.md` | Claude in Chrome 扩展 |
| `claude-mobile-ios.md` | claude.ai iOS 应用 |
| `anthropic_reminders.md`、`sonnet-4.6-reminders.md`、`research_instructions.md`、`visualize.md` | claude.ai 注入片段（提醒、研究、artifacts） |
| `Official/` | Anthropic 自行发布的提示词（release-notes 版本——比上方实际服务的提示词更短） |
