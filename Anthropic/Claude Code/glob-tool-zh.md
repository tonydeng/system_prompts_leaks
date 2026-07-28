> **说明**：本文件为英文原文（`glob-tool.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

<!-- Glob 自 Claude Code 2.1.211 起不在原生构建的默认主智能体工具集中（自约 2026 年 4 月 ~2.1.117 起被通过 Bash 嵌入的 bfs 取代）。搜索子智能体（Explore 等）仍接收它，可通过在完整 `--tools` 白名单中显式列出将其恢复到主智能体。以下定义在两种情况下完全相同——已于 2026-07-16 对 2.1.211 主智能体 `--tools` 捕获和 Explore 子智能体提取进行验证。 -->

# Glob

快速文件模式匹配。支持 glob 模式，如 "**/*.js" 或 "src/**/*.ts"。返回按修改时间排序的匹配文件路径。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "pattern": {
      "description": "The glob pattern to match files against",
      "type": "string"
    },
    "path": {
      "description": "The directory to search in. If not specified, the current working directory will be used. IMPORTANT: Omit this field to use the default directory. DO NOT enter \"undefined\" or \"null\" - simply omit it for the default behavior. Must be a valid directory path if provided.",
      "type": "string"
    }
  },
  "required": [
    "pattern"
  ],
  "additionalProperties": false
}
```
