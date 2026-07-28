> **说明**：本文件为英文原文（`files-api.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Files API — Go

## Files API（Beta）

位于 `client.Beta.Files` 下。方法是 **`Upload`**（不是 `New`/`Create`），参数结构体为 `BetaFileUploadParams`。`File` 字段接受 `io.Reader`；使用 `anthropic.File()` 附加文件名 + content-type 用于 multipart 编码。

```go
f, _ := os.Open("./upload_me.txt")
defer f.Close()

meta, err := client.Beta.Files.Upload(ctx, anthropic.BetaFileUploadParams{
    File:  anthropic.File(f, "upload_me.txt", "text/plain"),
    Betas: []anthropic.AnthropicBeta{anthropic.AnthropicBetaFilesAPI2025_04_14},
})
// meta.ID 是在后续消息请求中引用的 file_id
```

其他 `Beta.Files` 方法：`List`、`Delete`、`Download`、`GetMetadata`。

---
