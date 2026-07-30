> **说明**：本文件为英文原文（`init.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

请分析此代码库并创建一个 CLAUDE.md 文件，该文件将提供给未来的 Claude Code 实例，以便在此仓库中操作。

添加内容：
1. 常用命令，如如何构建、lint 和运行测试。包括在此代码库中开发所需的必要命令，如如何运行单个测试。
2. 高层代码架构和结构，使未来实例能更快上手。聚焦于需要阅读多个文件才能理解的"大局"架构。

使用说明：
- 若已有 CLAUDE.md，提出改进建议。
- 创建初始 CLAUDE.md 时，不要重复自己，也不要包含显而易见的指令，如"向用户提供有用的错误消息"、"为所有新工具编写单元测试"、"切勿在代码或提交中包含敏感信息（API 密钥、令牌）"。
- 避免列出每个可轻松发现的组件或文件结构。
- 不要包含通用开发实践。
- 若有 Cursor 规则（在 .cursor/rules/ 或 .cursorrules 中）或 Copilot 规则（在 .github/copilot-instructions.md 中），确保包含重要部分。
- 若有 README.md，确保包含重要部分。
- 不要编造诸如"常见开发任务"、"开发提示"、"支持与文档"等信息，除非你阅读的其他文件中明确包含这些内容。
- 确保文件以下列文本开头：

```
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.
```
