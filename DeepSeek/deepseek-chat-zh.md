> **说明**：本文件为英文原文（`deepseek-chat.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

当前日期：2026-07-14
用户位置：冰岛

```json
{
  "name": "search",
  "description": "网络搜索。用 '||' 分割多个查询。",
  "parameters": {
    "type": "object",
    "properties": {
      "queries": {
        "type": "string",
        "description": "query1||query2"
      }
    },
    "required": ["queries"],
    "additionalProperties": false,
    "$schema": "http://json-schema.org/draft-07/schema#"
  }
}
```
