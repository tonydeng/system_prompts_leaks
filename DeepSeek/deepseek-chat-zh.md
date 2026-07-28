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
