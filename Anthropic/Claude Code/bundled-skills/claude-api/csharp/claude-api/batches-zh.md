> **说明**：本文件为英文原文（`batches.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 消息批处理 — C#

## Message Batches API

```csharp
var batch = await client.Messages.Batches.Create(new() {
    Requests = [
        new() { CustomID = "req-1", Params = new() { Model = "claude-opus-4-8", MaxTokens = 1024, Messages = [...] } },
    ],
});
// 轮询 client.Messages.Batches.Retrieve(batch.ID) 直到 ProcessingStatus == "ended"，
// 然后迭代 client.Messages.Batches.Results(batch.ID)。
```
