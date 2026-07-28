> **说明**：本文件为英文原文（`batches.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 消息批处理 — PHP

## Message Batches API

```php
$batch = $client->messages->batches->create(requests: [
    ['customId' => 'req-1', 'params' => ['model' => 'claude-opus-4-8', 'maxTokens' => 1024, 'messages' => [...]]],
    ['customId' => 'req-2', 'params' => [...]],
]);
// 轮询 $client->messages->batches->retrieve($batch->id) 直到 processingStatus === 'ended'，
// 然后迭代 $client->messages->batches->results($batch->id)。
```

---
