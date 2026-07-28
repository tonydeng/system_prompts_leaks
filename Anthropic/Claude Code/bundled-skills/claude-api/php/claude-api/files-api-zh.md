> **说明**：本文件为英文原文（`files-api.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Files API — PHP

## Files API

```php
$file = $client->beta->files->upload(
    file: fopen('upload_me.txt', 'r'),
    betas: ['files-api-2025-04-14'],
);
// 将 $file->id 作为文件内容块在 ->beta->messages->create() 上引用。
```
