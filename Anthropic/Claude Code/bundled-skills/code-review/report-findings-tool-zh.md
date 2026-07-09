> **说明**：本文件为英文原文（`report-findings-tool.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# ReportFindings 工具

将代码审查发现作为类型化列表报告，以便主机 UI 可以呈现它们。仅当活动的代码审查指令告诉你使用此工具报告发现时才使用它；否则遵循这些指令指定的任何输出格式。报告审查结果时，调用一次并按最严重排序的已验证发现（如果没有通过验证，则为空数组），并且不要也将发现作为文本打印。在应用修复后重新报告时（仅当应用指令要求时），将每个发现的 `outcome` 设置为实际发生的情况。

## input_schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "level": {
      "description": "审查运行的投入级别",
      "type": "string",
      "enum": [
        "low",
        "medium",
        "high",
        "xhigh",
        "max"
      ]
    },
    "findings": {
      "description": "已验证的发现，最严重优先；如果没有存活则为空",
      "maxItems": 32,
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "file": {
            "description": "发现所在文件的仓库相对路径",
            "type": "string"
          },
          "line": {
            "description": "发现锚定的 1 索引行",
            "type": "integer",
            "minimum": -9007199254740991,
            "maximum": 9007199254740991
          },
          "summary": {
            "description": "缺陷的单句陈述",
            "type": "string"
          },
          "failure_scenario": {
            "description": "具体输入/状态 → 错误输出/崩溃",
            "type": "string"
          },
          "verdict": {
            "description": "当运行验证传递时设置；在仅内联审查中不存在",
            "type": "string",
            "enum": [
              "CONFIRMED",
              "PLAUSIBLE"
            ]
          },
          "outcome": {
            "description": "仅在应用修复后重新报告时设置：此发现发生了什么",
            "type": "string",
            "enum": [
              "fixed",
              "skipped",
              "no_change_needed"
            ]
          }
        },
        "required": [
          "file",
          "summary",
          "failure_scenario"
        ],
        "additionalProperties": false
      }
    }
  },
  "required": [
    "findings"
  ],
  "additionalProperties": false
}
```
