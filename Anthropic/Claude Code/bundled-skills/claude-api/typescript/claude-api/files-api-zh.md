> **说明**：本文件为英文原文（`files-api.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Files API — TypeScript

Files API 上传文件供 Messages API 请求使用。通过内容块中的 `file_id` 引用文件，避免跨多次 API 调用重新上传。

**Beta：** 在 API 调用中传入 `betas: ["files-api-2025-04-14"]`（SDK 自动设置所需 header）。

## 关键事实

- 最大文件大小：500 MB
- 总存储：每组织 100 GB
- 文件在删除前持久存在
- 文件操作（上传、列表、删除）免费；消息中使用的内容按输入 token 计费
- 在 Amazon Bedrock 或 Google Vertex AI 上不可用

---

## 上传文件

```typescript
import Anthropic, { toFile } from "@anthropic-ai/sdk";
import fs from "fs";

const client = new Anthropic();

const uploaded = await client.beta.files.upload({
  file: await toFile(fs.createReadStream("report.pdf"), undefined, {
    type: "application/pdf",
  }),
  betas: ["files-api-2025-04-14"],
});

console.log(`File ID: ${uploaded.id}`);
console.log(`Size: ${uploaded.size_bytes} bytes`);
```

---

## 在消息中使用文件

### PDF / 文本文档

```typescript
const response = await client.beta.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Summarize the key findings in this report." },
        {
          type: "document",
          source: { type: "file", file_id: uploaded.id },
          title: "Q4 Report",
          citations: { enabled: true },
        },
      ],
    },
  ],
  betas: ["files-api-2025-04-14"],
});

console.log(response.content[0].text);
```

---

## 管理文件

### 列出文件

```typescript
const files = await client.beta.files.list({
  betas: ["files-api-2025-04-14"],
});
for (const f of files.data) {
  console.log(`${f.id}: ${f.filename} (${f.size_bytes} bytes)`);
}
```

### 删除文件

```typescript
await client.beta.files.delete("file_011CNha8iCJcU1wXNR6q4V8w", {
  betas: ["files-api-2025-04-14"],
});
```

### 下载文件

```typescript
const response = await client.beta.files.download(
  "file_011CNha8iCJcU1wXNR6q4V8w",
  { betas: ["files-api-2025-04-14"] },
);
const content = Buffer.from(await response.arrayBuffer());
await fs.promises.writeFile("output.txt", content);
```
