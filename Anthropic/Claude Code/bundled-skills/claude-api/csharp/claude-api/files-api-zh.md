> **说明**：本文件为英文原文（`files-api.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Files API — C#

## Files API（Beta）

文件位于 `client.Beta.Files` 下（命名空间 `Anthropic.Models.Beta.Files`）。`BinaryContent` 可从 `Stream` 和 `byte[]` 隐式转换。

```csharp
using Anthropic.Models.Beta.Files;
using Anthropic.Models.Beta.Messages;

FileMetadata meta = await client.Beta.Files.Upload(
    new FileUploadParams { File = File.OpenRead("doc.pdf") });

// 引用上传的文件需要 Beta 消息类型：
new BetaRequestDocumentBlock {
    Source = new BetaFileDocumentSource { FileID = meta.ID },
}
```

非 beta 的 `DocumentBlockParamSource` 联合类型没有 file-ID 变体——文件引用需要 `client.Beta.Messages.Create()`。

---
