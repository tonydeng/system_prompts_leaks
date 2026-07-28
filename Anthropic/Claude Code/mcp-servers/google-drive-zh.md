> **说明**：本文件为英文原文（`google-drive.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

<!-- MCP server: Google Drive | captured 2026-07-16, Claude Code 2.1.211 -->

# Google Drive（MCP 服务器）

## 工具（8 个）

### copy_file

调用此工具复制 Google Drive 中的现有文件。
该工具允许指定新标题和副本的父文件夹。
如果未指定标题，副本标题将为 'Copy of {original title}'。如果未指定父文件夹，副本将在与原始文件相同的文件夹中创建，除非请求用户对该文件夹没有写入权限，此时副本将在用户的根文件夹中创建。成功复制后返回新创建的 File 对象。

```json
{
  "type": "object",
  "properties": {
    "fileId": {
      "description": "Required. The ID of the file to copy.",
      "type": "string"
    },
    "parentId": {
      "description": "The parent id of the newly created file. If empty, the file will be created with the same parent as the original file.",
      "type": "string"
    },
    "title": {
      "description": "The title of the newly created file. If empty, the title will be 'Copy of [original file title]'.",
      "type": "string"
    }
  },
  "required": [
    "fileId"
  ],
  "description": "Request to copy a file."
}
```

### create_file

调用此工具在 Google Drive 中创建或上传文件。

如果要上传内容，文本内容优先使用 "text_content"。对于非 UTF-8 内容，使用 "base64_content" 字段并将数据 base64 编码后设置到该字段。

成功创建后返回单个 File 对象。

以下 Google 第一方 mime 类型可以在不提供内容的情况下创建：

 - `application/vnd.google-apps.document` 
 - `application/vnd.google-apps.spreadsheet` 
 - `application/vnd.google-apps.presentation` 

可以通过将 mime 类型设置为 `application/vnd.google-apps.folder` 来创建文件夹。

上传内容时，`content_mime_type` 字段是必需的，应与上传内容的类型匹配。

默认情况下，支持的内容会转换为 Google 第一方 mime 类型。

要禁用第一方 mime 类型的转换，将 `disable_conversion_to_google_type` 设为 true。

```json
{
  "type": "object",
  "properties": {
    "base64Content": {
      "description": "Optional. The base64 encoded content to upload. It's an error to set this and text_content.",
      "type": "string"
    },
    "content": {
      "description": "The content of the file encoded as base64. The content field should always be base64 encoded regardless of the mime type of the file. DEPRECATED. Use base64_content or text_content instead.",
      "type": "string"
    },
    "contentMimeType": {
      "description": "The mime type of the content being uploaded. Required when any type of content is provided.",
      "type": "string"
    },
    "disableConversionToGoogleType": {
      "description": "Set to true to retain the passed in content mime type and not convert to a Google type. For example, without this a text/plain content mime type will be converted to to an application/vnd.google-apps.document. Has no effect for types that do not have a Google equivalent.",
      "type": "boolean"
    },
    "mimeType": {
      "description": "DEPRECATED. DO NOT USE!! Set content_mime_type instead.",
      "type": "string"
    },
    "parentId": {
      "description": "The parent id of the file.",
      "type": "string"
    },
    "textContent": {
      "description": "Optional. The (UTF-8) text content to upload. It's an error to set this and base64_content.",
      "type": "string"
    },
    "title": {
      "description": "The title of the file.",
      "type": "string"
    }
  },
  "description": "Request to upload a file."
}
```

### download_file_content

调用此工具以 base64 编码字符串形式下载 Drive 文件的内容。

如果文件是 Google Drive 第一方 mime 类型，`exportMimeType` 字段是必需的，将决定下载文件的格式。

如果未找到文件，尝试使用 `search_files` 等其他工具查找用户请求的文件。

如果用户想要其 Drive 内容的自然语言表示，使用 `read_file_content` 工具（`read_file_content` 应该更小且更易解析）。

```json
{
  "type": "object",
  "properties": {
    "exportMimeType": {
      "description": "Optional. For Google native files, the MIME type to export the file to, ignored otherwise. Defaults to text if not specified.",
      "type": "string"
    },
    "fileId": {
      "description": "Required. The ID of the file to retrieve.",
      "type": "string"
    }
  },
  "required": [
    "fileId"
  ],
  "description": "Defines a request to download a file's content."
}
```

### get_file_metadata

调用此工具查找用户 Drive 文件的一般元数据。

如果未找到文件，尝试使用 `search_files` 等其他工具查找用户请求的文件。

```json
{
  "type": "object",
  "properties": {
    "excludeContentSnippets": {
      "description": "If true, the content snippet will be excluded from the response.",
      "type": "boolean"
    },
    "fileId": {
      "description": "Required. The ID of the file to retrieve.",
      "type": "string"
    }
  },
  "required": [
    "fileId"
  ],
  "description": "Request to get the file."
}
```

### get_file_permissions

调用此工具列出 Drive 文件的权限。

```json
{
  "type": "object",
  "properties": {
    "fileId": {
      "description": "Required. The ID of the file to get permissions for.",
      "type": "string"
    }
  },
  "required": [
    "fileId"
  ],
  "description": "Request to get file permissions."
}
```

### list_recent_files

调用此工具查找用户的最近文件，可指定排序顺序。默认排序为 `recency`。

支持的排序方式：

 - `recency`：文件日期时间字段中最近的时间戳。
 - `lastModified`：文件被任何人修改的最后时间。
 - `lastModifiedByMe`：用户修改文件的最后时间。

默认页面大小为 10。使用 `next_page_token` 分页遍历结果。

```json
{
  "type": "object",
  "properties": {
    "excludeContentSnippets": {
      "description": "If true, the content snippet will be excluded from the response.",
      "type": "boolean"
    },
    "orderBy": {
      "description": "The sort order for the files.",
      "type": "string"
    },
    "pageSize": {
      "description": "The maximum number of files to return.",
      "format": "int32",
      "type": "integer"
    },
    "pageToken": {
      "description": "The page token to use for pagination.",
      "type": "string"
    }
  },
  "description": "Request to list files."
}
```

### read_file_content

调用此工具获取 Drive 文件的自然语言表示，以及可选的评论。

对于非常大的文件，文件内容可能不完整。文本表示会随时间变化，因此不要对此工具返回的文本的特定格式做假设。如果支持，评论标签将包含在内容中。

支持的 Mime 类型：

 - `application/vnd.google-apps.document` 
 - `application/vnd.google-apps.presentation` 
 - `application/vnd.google-apps.spreadsheet` 
 - `application/pdf` 
 - `application/msword` 
 - `application/vnd.openxmlformats-officedocument.wordprocessingml.document` 
 - `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` 
 - `application/vnd.openxmlformats-officedocument.presentationml.presentation` 
 - `application/vnd.oasis.opendocument.spreadsheet` 
 - `application/vnd.oasis.opendocument.presentation` 
 - `application/x-vnd.oasis.opendocument.text` 
 - `image/png` 
 - `image/jpeg` 
 - `image/jpg` 

如果未找到文件，尝试使用 `search_files` 等其他工具用关键词查找用户请求的文件。

```json
{
  "type": "object",
  "properties": {
    "fileId": {
      "description": "Required. The ID of the file to retrieve.",
      "type": "string"
    },
    "includeComments": {
      "description": "Whether to include comments in the response. Comments will be inlined in the text content of the file with a mapping to the comment threads.",
      "type": "boolean"
    }
  },
  "required": [
    "fileId"
  ],
  "description": "Request to read file content with support for fetching comments."
}
```

### search_files

使用结构化查询搜索 Drive 文件（语法：`query_term operator values`）。
用 `and`、`or`、`not` 和括号组合子句。字符串值必须用单引号括起，内嵌引号转义为 `\'`。

查询词和操作符：

 - `title`（操作符：contains, =, !=）— 文件标题
 - `fullText`（操作符：contains）— 标题或正文文本
 - `mimeType`（操作符：contains, =, !=）— MIME 类型
 - `modifiedTime`、`viewedByMeTime`、`createdTime`（操作符：`<=`、`<`、`=`、`!=`、`>`、`>=`）。使用 RFC 3339 UTC，例如 `2012-06-04T12:00:00-08:00`。日期类型不可比较。
 - `parentId`（操作符：`=`、`!=`）。用 `'root'` 表示用户的"我的云端硬盘"。
 - `owner`（操作符：`=`、`!=`）。用 `'me'` 表示请求用户。
 - `sharedWithMe`（操作符：`=`、`!=`）。值：`true` 或 `false`。

其他操作符：`and`、`or`、`not`。

示例：

 - `title contains 'hello' and title contains 'goodbye'`
 - `modifiedTime > '2024-01-01T00:00:00Z' and (mimeType contains 'image/' or mimeType contains 'video/')`
 - `parentId = '1234567'`
 - `fullText contains 'hello'`
 - `owner = 'test@example.org'`
 - `sharedWithMe = true`
 - `owner = 'me'`（用户拥有的文件）

使用 `next_page_token` 分页。空响应表示没有更多结果。

```json
{
  "type": "object",
  "properties": {
    "excludeContentSnippets": {
      "description": "If true, the content snippet will be excluded from the response.",
      "type": "boolean"
    },
    "pageSize": {
      "description": "The maximum number of files to return in each page.",
      "format": "int32",
      "type": "integer"
    },
    "pageToken": {
      "description": "The page token to use for pagination.",
      "type": "string"
    },
    "query": {
      "description": "The search query.",
      "type": "string"
    }
  },
  "description": "Request to search files."
}
```
