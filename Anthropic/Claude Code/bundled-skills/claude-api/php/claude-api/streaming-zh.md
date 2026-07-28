> **说明**：本文件为英文原文（`streaming.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 流式输出 — PHP

## 流式输出

> **需要 SDK v0.5.0+。** v0.4.0 及更早版本使用单个 `$params` 数组；以命名参数调用会抛出 `Unknown named parameter $model`。升级：`composer require "anthropic-ai/sdk:^0.7"`

```php
use Anthropic\Messages\RawContentBlockDeltaEvent;
use Anthropic\Messages\TextDelta;

$stream = $client->messages->createStream(
    model: 'claude-opus-4-8',
    maxTokens: 64000,
    messages: [
        ['role' => 'user', 'content' => 'Write a haiku'],
    ],
);

foreach ($stream as $event) {
    if ($event instanceof RawContentBlockDeltaEvent && $event->delta instanceof TextDelta) {
        echo $event->delta->text;
    }
}
```

---
