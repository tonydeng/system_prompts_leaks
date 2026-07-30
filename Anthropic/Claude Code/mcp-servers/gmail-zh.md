> **说明**：本文件为英文原文（`gmail.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

<!-- MCP server: Gmail | captured 2026-07-16, Claude Code 2.1.211 -->

# Gmail（MCP 服务器）

## 工具（13 个）

### apply_sensitive_message_label

向已认证用户的 Gmail 帐户中的特定邮件添加敏感标签（垃圾箱或垃圾邮件）。

使用此工具将邮件移至垃圾箱或标记为垃圾邮件。要查找邮件 ID，请使用 `search_threads` 或 `get_thread` 等工具。

```json
{
  "type": "object",
  "properties": {
    "labelOption": {
      "description": "Required. The sensitive label option to add.",
      "enum": [
        "LABEL_OPTION_UNSPECIFIED",
        "TRASH",
        "SPAM"
      ],
      "type": "string",
      "x-google-enum-descriptions": [
        "Unspecified label option.",
        "Trash label.",
        "Spam label."
      ]
    },
    "messageId": {
      "description": "Required. The ID of the message to add the label to.",
      "type": "string"
    }
  },
  "required": [
    "messageId",
    "labelOption"
  ],
  "description": "Request message for ApplySensitiveMessageLabel RPC."
}
```

### apply_sensitive_thread_label

向已认证用户的 Gmail 帐户中的整个会话添加敏感标签（垃圾箱或垃圾邮件）。此操作影响会话中当前的所有邮件以及未来添加到其中的任何邮件。

使用此工具将会话移至垃圾箱或标记为垃圾邮件。如果不确定会话 ID，请先使用 `search_threads` 工具。

```json
{
  "type": "object",
  "properties": {
    "labelOption": {
      "description": "Required. The sensitive label option to add.",
      "enum": [
        "LABEL_OPTION_UNSPECIFIED",
        "TRASH",
        "SPAM"
      ],
      "type": "string",
      "x-google-enum-descriptions": [
        "Unspecified label option.",
        "Trash label.",
        "Spam label."
      ]
    },
    "threadId": {
      "description": "Required. The ID of the thread to add the label to.",
      "type": "string"
    }
  },
  "required": [
    "threadId",
    "labelOption"
  ],
  "description": "Request message for ApplySensitiveThreadLabel RPC."
}
```

### create_draft

在已认证用户的 Gmail 帐户中创建新的草稿邮件。

此工具接收收件人地址、主题和正文内容作为输入。返回已创建 Gmail 草稿的 ID。如果草稿是作为对现有邮件的回复创建的，应将原始邮件的 ID 通过 replyToMessageId 字段传递给工具。目前不支持创建带附件的草稿。

```json
{
  "type": "object",
  "properties": {
    "attachments": {
      "description": "Optional. The attachments to include in the email. The combined size of attachments in the message cannot exceed 25MB. If you need to send files larger than 25MB, upload the file to Drive first and then insert the Drive link into body or html_body.",
      "items": {
        "$ref": "#/$defs/Attachment"
      },
      "type": "array"
    },
    "bcc": {
      "description": "Optional. The blind carbon copy recipients of the email draft. Each string MUST be a valid plain email address (e.g., \"user@example.com\"). The \"Name \" format is NOT supported by this tool.",
      "items": {
        "type": "string"
      },
      "type": "array"
    },
    "body": {
      "description": "Optional. The main body content of the email draft. If html_body is also provided, this field is treated as the plain-text alternative.",
      "type": "string"
    },
    "cc": {
      "description": "Optional. The carbon copy recipients of the email draft. Each string MUST be a valid plain email address (e.g., \"user@example.com\"). The \"Name \" format is NOT supported by this tool.",
      "items": {
        "type": "string"
      },
      "type": "array"
    },
    "htmlBody": {
      "description": "The HTML content of the email draft. If provided, this will be used as the rich-text version of the email.",
      "type": "string"
    },
    "replyToMessageId": {
      "description": "Optional. The ID of the message to reply to. If provided, this will be used as the reply-to message ID for the email draft, and the `body` and `html_body` will be appended to the original message body.",
      "type": "string"
    },
    "subject": {
      "description": "Optional. The subject line of the email. Defaults to empty if not provided.",
      "type": "string"
    },
    "to": {
      "description": "Optional. The primary recipients of the email draft. Each string MUST be a valid plain email address (e.g., \"user@example.com\"). The \"Name \" format is NOT supported by this tool.",
      "items": {
        "type": "string"
      },
      "type": "array"
    }
  },
  "$defs": {
    "Attachment": {
      "description": "Represents an attachment to be included in an email.",
      "properties": {
        "content": {
          "description": "Required. The base64-encoded content of the attachment.",
          "format": "byte",
          "type": "string"
        },
        "filename": {
          "description": "Optional. The name of the file to be attached, e.g. \"invoice.pdf\". For inline attachments, this is used for Content-ID generation. For regular attachments, filename is used to specify the filename to email clients. If not provided, the attachment may be received with no name.",
          "type": "string"
        },
        "id": {
          "description": "Optional. Output only. When present, contains the ID of an external attachment that can be retrieved in a separate `GetMessageAttachment` request.",
          "readOnly": true,
          "type": "string"
        },
        "inline": {
          "description": "Optional. If true, this attachment is handled as inline. An inline attachment is a content that is intended to be displayed within the body of an HTML email, as opposed to being listed as a separate file for download. If false or absent, defaults to false, and it's treated as a regular attachment.",
          "type": "boolean"
        },
        "mimeType": {
          "description": "Optional. The field representing a content or media type must use IANA MIME type, https://www.iana.org/assignments/media-types/media-types.xhtml. If not provided, defaults to \"application/octet-stream\".",
          "type": "string"
        }
      },
      "required": [
        "content"
      ],
      "type": "object"
    }
  },
  "description": "Request message for CreateDraft RPC."
}
```

### create_label

在已认证用户的 Gmail 帐户中创建新标签。
支持使用正斜杠创建嵌套标签（子标签）（例如 'Projects/Alpha/Sprint-1'）。
默认情况下，如果父标签不存在，将自动创建。

```json
{
  "type": "object",
  "properties": {
    "autoCreateParentLabels": {
      "description": "Optional. Whether to automatically create parent labels for nested labels (separated by '/'). Defaults to true.",
      "type": "boolean"
    },
    "color": {
      "$ref": "#/$defs/LabelColor",
      "description": "Optional. The color of the label."
    },
    "displayName": {
      "description": "Required. The display name of the label to create.",
      "type": "string"
    }
  },
  "required": [
    "displayName"
  ],
  "$defs": {
    "LabelColor": {
      "description": "The color of the label.",
      "properties": {
        "backgroundColor": {
          "description": "The background color of the label, represented as a hex string (e.g., \"#ffffff\"). Only the following predefined set of color values are allowed: # 000000, #434343, #666666, #999999, #cccccc, #efefef, #f3f3f3, #ffffff, # fb4c2f, #ffad47, #fad165, #16a766, #43d692, #4a86e8, #a479e2, #f691b3, # f6c5be, #ffe6c7, #fef1d1, #b9e4d0, #c6f3de, #c9daf8, #e4d7f5, #fcdee8, # efa093, #ffd6a2, #fce8b3, #89d3b2, #a0eac9, #a4c2f4, #d0bcf1, #fbc8d9, # e66550, #ffbc6b, #fcda83, #44b984, #68dfa9, #6d9eeb, #b694e8, #f7a7c0, # cc3a21, #eaa041, #f2c960, #149e60, #3dc789, #3c78d8, #8e63ce, #e07798, # ac2b16, #cf8933, #d5ae49, #0b804b, #2a9c68, #285bac, #653e9b, #b65775, # 822111, #a46a21, #aa8831, #076239, #1a764d, #1c4587, #41236d, #83334c, # 464646, #e7e7e7, #0d3472, #b6cff5, #0d3b44, #98d7e4, #3d188e, #e3d7ff, # 711a36, #fbd3e0, #8a1c0a, #f2b2a8, #7a2e0b, #ffc8af, #7a4706, #ffdeb5, # 594c05, #fbe983, #684e07, #fdedc1, #0b4f30, #b3efd3, #04502e, #a2dcc1, # c2c2c2, #4986e7, #2da2bb, #b99aff, #994a64, #f691b2, #ff7537, #ffad46, # 662e37, #ebdbde, #cca6ac, #094228, #42d692, #16a765",
          "type": "string"
        },
        "textColor": {
          "description": "The text color of the label, represented as a hex string (e.g., \"#000000\"). Only the following predefined set of color values are allowed: # 000000, #434343, #666666, #999999, #cccccc, #efefef, #f3f3f3, #ffffff, # fb4c2f, #ffad47, #fad165, #16a766, #43d692, #4a86e8, #a479e2, #f691b3, # f6c5be, #ffe6c7, #fef1d1, #b9e4d0, #c6f3de, #c9daf8, #e4d7f5, #fcdee8, # efa093, #ffd6a2, #fce8b3, #89d3b2, #a0eac9, #a4c2f4, #d0bcf1, #fbc8d9, # e66550, #ffbc6b, #fcda83, #44b984, #68dfa9, #6d9eeb, #b694e8, #f7a7c0, # cc3a21, #eaa041, #f2c960, #149e60, #3dc789, #3c78d8, #8e63ce, #e07798, # ac2b16, #cf8933, #d5ae49, #0b804b, #2a9c68, #285bac, #653e9b, #b65775, # 822111, #a46a21, #aa8831, #076239, #1a764d, #1c4587, #41236d, #83334c, # 464646, #e7e7e7, #0d3472, #b6cff5, #0d3b44, #98d7e4, #3d188e, #e3d7ff, # 711a36, #fbd3e0, #8a1c0a, #f2b2a8, #7a2e0b, #ffc8af, #7a4706, #ffdeb5, # 594c05, #fbe983, #684e07, #fdedc1, #0b4f30, #b3efd3, #04502e, #a2dcc1, # c2c2c2, #4986e7, #2da2bb, #b99aff, #994a64, #f691b2, #ff7537, #ffad46, # 662e37, #ebdbde, #cca6ac, #094228, #42d692, #16a765",
          "type": "string"
        }
      },
      "type": "object"
    }
  },
  "description": "Request message for CreateLabel RPC."
}
```

### get_message

通过唯一的邮件 ID 从已认证用户的 Gmail 帐户中检索特定邮件。

当你已知邮件 ID 时，使用此工具检查单封邮件。如果用户想详细阅读特定邮件、检查邮件的确切措辞或检查单封邮件的附件元数据，此工具是正确的选择。它不适合检索整个对话或查看来回讨论的会话；请改用 'get_thread' 工具。
关键指标包括：用户要求获取先前搜索返回的特定邮件 ID 的完整内容，或查询要求检查特定单封邮件而非整个会话。
用户提示示例："Get the full text of message ID 18f123456789abcd."、"Read the latest message in that thread from Alice." 和 "What are the attachment names in the email I just received from HR?"

可选的 `messageFormat` 参数控制返回邮件的格式。默认情况下（或使用 `FULL_CONTENT`），返回邮件的完整内容。使用 `MINIMAL` 仅包含主题和摘要（不含正文）。使用 `METADATA_ONLY` 仅包含基本元数据（邮件 ID、会话 ID、标签、时间戳和大小估算）。

```json
{
  "type": "object",
  "properties": {
    "messageFormat": {
      "description": "Optional. Specifies the format of the message returned. Defaults to FULL_CONTENT.",
      "enum": [
        "MESSAGE_FORMAT_UNSPECIFIED",
        "MINIMAL",
        "FULL_CONTENT",
        "METADATA_ONLY"
      ],
      "type": "string",
      "x-google-enum-descriptions": [
        "Defaults to FULL_CONTENT.",
        "Returns message snippets and key headers (Subject, From, To, Cc, Date).",
        "Returns all information in \"MINIMAL\" plus the full body content of each message.",
        "Metadata only: does not include subject, snippet, body, attachment filenames."
      ]
    },
    "messageId": {
      "description": "Required. The unique identifier of the message to fetch.",
      "type": "string"
    }
  },
  "required": [
    "messageId"
  ],
  "description": "Request message for GetMessage RPC."
}
```

### get_thread

从已认证用户的 Gmail 帐户中检索特定邮件会话，包括其邮件列表。

可选的 `message_format` 参数控制返回邮件的格式。默认情况下（或使用 `FULL_CONTENT`），返回邮件的完整内容。使用 `MINIMAL` 仅包含主题和摘要（不含正文）。使用 `METADATA_ONLY` 仅包含基本元数据（邮件 ID、会话 ID、标签、时间戳和大小估算）。

```json
{
  "type": "object",
  "properties": {
    "messageFormat": {
      "description": "Optional. Specifies the format of the messages returned within the thread. Defaults to FULL_CONTENT. Note: If you need body content or attachments, use FULL_CONTENT. When using MINIMAL, the plaintext_body and attachment_ids fields will not be populated. If you are unsure which format to use, rely on the default behavior by using FULL_CONTENT.",
      "enum": [
        "MESSAGE_FORMAT_UNSPECIFIED",
        "MINIMAL",
        "FULL_CONTENT",
        "METADATA_ONLY"
      ],
      "type": "string",
      "x-google-enum-descriptions": [
        "Defaults to FULL_CONTENT.",
        "Returns message snippets and key headers (Subject, From, To, Cc, Date).",
        "Returns all information in \"MINIMAL\" plus the full body content of each message.",
        "Metadata only: does not include subject, snippet, body, attachment filenames."
      ]
    },
    "threadId": {
      "description": "Required. The unique identifier of the thread to fetch.",
      "type": "string"
    }
  },
  "required": [
    "threadId"
  ],
  "description": "Request message for GetThread RPC."
}
```

### label_message

向已认证用户的 Gmail 帐户中的特定邮件添加一个或多个标签。

要查找邮件 ID，请使用 `search_threads` 或 `get_thread` 等工具。如果不确定用户标签的 ID，请先使用 `list_labels` 工具发现可用标签及其 ID。要为邮件添加垃圾箱标签或垃圾邮件标签，请改用 `apply_sensitive_message_label` 工具。

```json
{
  "type": "object",
  "properties": {
    "labelIds": {
      "description": "Required. The IDs of the labels to add. Can be a system label ID (e.g., 'INBOX', 'TRASH', 'SPAM', 'STARRED', 'UNREAD', 'IMPORTANT') or a user-defined label ID. The tool accepts `label_ids` and not label names. Use the list_labels tool to get the corresponding label id to a display name for user-defined labels.",
      "items": {
        "type": "string"
      },
      "type": "array"
    },
    "messageId": {
      "description": "Required. The ID of the message to add the labels to.",
      "type": "string"
    }
  },
  "required": [
    "messageId",
    "labelIds"
  ],
  "description": "Request message for LabelMessage RPC."
}
```

### label_thread

向已认证用户的 Gmail 帐户中的整个会话添加标签。此操作影响会话中当前的所有邮件以及未来添加到其中的任何邮件。

如果不确定会话 ID，请先使用 `search_threads` 工具。

如果不确定用户标签的 ID，请先使用 `list_labels` 工具发现可用标签及其 ID。要为会话添加垃圾箱标签或垃圾邮件标签，请改用 `apply_sensitive_thread_label` 工具。

```json
{
  "type": "object",
  "properties": {
    "labelIds": {
      "description": "Required. The unique identifiers of the labels to add. Can be a system label ID (e.g., 'INBOX', 'TRASH', 'SPAM', 'STARRED', 'UNREAD', 'IMPORTANT') or a user-defined label ID. The tool accepts `label_ids` and not label names. Use the list_labels tool to get the corresponding label id to a display name for user-defined labels.",
      "items": {
        "type": "string"
      },
      "type": "array"
    },
    "threadId": {
      "description": "Required. The unique identifier of the thread to add labels to.",
      "type": "string"
    }
  },
  "required": [
    "threadId",
    "labelIds"
  ],
  "description": "Request message for LabelThread RPC."
}
```

### list_drafts

列出已认证用户的 Gmail 帐户中的草稿邮件。

此工具可以根据查询字符串筛选草稿，并支持分页。它返回草稿列表，包括其 ID 和主题（除非 `view` 设置为 `DRAFT_VIEW_METADATA_ONLY`）。`page_token` 可用于对结果进行分页。要检索后续结果页，请使用先前响应中返回的 `page_token`。

`view` 参数控制响应中填充哪些字段。默认情况下（或使用 `DRAFT_VIEW_FULL`），返回完整内容。使用 `DRAFT_VIEW_METADATA_ONLY` 可排除主题和正文等敏感内容。

```json
{
  "type": "object",
  "properties": {
    "pageSize": {
      "description": "Optional. The maximum number of drafts to return. If unspecified, defaults to 20. The maximum allowed value is 50.",
      "format": "int32",
      "type": "integer"
    },
    "pageToken": {
      "description": "Optional. A token received from a previous list_drafts call to retrieve the next page of results. Leave empty to fetch the first page. This is primarily used for pagination to continue fetching results from where the previous `ListDraft` call left off, especially when the number of drafts matching the query exceeds the page_size limit.",
      "type": "string"
    },
    "query": {
      "description": "Examples: \"subject:OneMCP Update\" \"from:gduser1@workspacesamples.dev\" \"to:gduser2@workspacesamples.dev AND newer_than:7d\" \"project proposal has:attachment\" \"is:unread\" A space or a dash (`-`) will separate a number while a dot (`.`) will be a decimal. For example, `01.2047-100` is considered two numbers: `01.2047` and `100`. Note: If we want to ensure all drafts for the query are returned, we can paginate the results by making repeated calls to the tool until the response contains an empty list of drafts.",
      "type": "string"
    },
    "view": {
      "description": "Optional. Controls the fields populated for drafts in the draft list.",
      "enum": [
        "DRAFT_VIEW_UNSPECIFIED",
        "DRAFT_VIEW_METADATA_ONLY",
        "DRAFT_VIEW_FULL"
      ],
      "type": "string",
      "x-google-enum-descriptions": [
        "Maps to DRAFT_VIEW_FULL for backward compatibility.",
        "Metadata only: does not include subject, plaintext_body, html_body.",
        "Metadata + UGC (Default behavior)."
      ]
    }
  },
  "description": "Request message for ListDrafts RPC."
}
```

### list_labels

列出已认证用户的 Gmail 帐户中所有可用的用户自定义标签。在调用 `label_thread`、`unlabel_thread`、`label_message` 或 `unlabel_message` 之前，使用此工具发现用户标签的 `id`。系统标签不会由此工具返回，但可以使用其已知 ID：'INBOX'、'TRASH'、'SPAM'、'STARRED'、'UNREAD'、'IMPORTANT'、'CHAT'、'DRAFT'、'SENT'。

```json
{
  "type": "object",
  "properties": {
    "pageSize": {
      "description": "Optional. The maximum number of labels to return.",
      "format": "int32",
      "type": "integer"
    },
    "pageToken": {
      "description": "Optional. Page token to retrieve a specific page of results in the list.",
      "type": "string"
    }
  },
  "description": "Request message for ListLabels RPC."
}
```

### search_threads

列出已认证用户的 Gmail 帐户中的邮件会话。

此工具可以根据查询字符串筛选会话，并支持分页。它返回会话列表，包括其 ID 和相关邮件。每封相关邮件包含邮件正文摘要、主题、发件人、收件人等详细信息。`view` 参数控制相关邮件中填充哪些字段。默认情况下（或使用 `THREAD_VIEW_MINIMAL`），包含主题和摘要。使用 `THREAD_VIEW_METADATA_ONLY` 可排除主题和摘要。注意，此工具不返回完整邮件正文；如需获取完整邮件正文，请使用 'get_thread' 工具配合会话 ID。被排除条件的会话可能仍出现在结果中。这是因为 Gmail 会先识别匹配的邮件。例如，如果你搜索 -is:starred，只要会话中至少有一封未加星标的邮件，Gmail 就会找到整个会话，即使同一对话中的其他邮件已加星标。

```json
{
  "type": "object",
  "properties": {
    "includeTrash": {
      "description": "Optional. Include drafts from TRASH in the results. Defaults to false.",
      "type": "boolean"
    },
    "pageSize": {
      "description": "Optional. The maximum number of threads to return. If unspecified, defaults to 20. The maximum allowed value is 50.",
      "format": "int32",
      "type": "integer"
    },
    "pageToken": {
      "description": "Optional. Page token to retrieve a specific page of results in the list. Leave empty to fetch the first page. This is primarily used for pagination to continue fetching results from where the previous `SearchThreads` call left off, especially when the number of threads matching the query exceeds the page_size limit.",
      "type": "string"
    },
    "query": {
      "description": "Optional. A query string to filter the threads. Natural language queries must be pre-converted into Gmail syntax queries to use this tool. If omitted, all threads (excluding spam and trash by default) are listed. Supported Operators by Category: Sender & Recipient: from: - Sent from a specific person. to: - Sent to a specific person. cc: - Specific people in Cc. bcc: - Specific people in Bcc. deliveredto: - Delivered to a specific address. list: - From a specific mailing list. Time & Date: after:YYYY/MM/DD / newer:YYYY/MM/DD - Received after a date. before:YYYY/MM/DD / older:YYYY/MM/DD - Received before a date. older_than: - Older than a duration (e.g., 1y, 2d). newer_than: - Newer than a duration. Content: subject: - Words in the subject line. has: - Has specific content types (attachment, drive, youtube, document). filename: - Attachment with a specific name or type. \"\" - Search for an exact word or phrase. (e.g., \"holiday\", \"holiday vacation\"). + - Match a word exactly. (e.g., +holiday, +unicorn) rfc822msgid: - Specific message ID header. AROUND - Find words near each other (e.g., holiday AROUND 10 vacation). Labels & Categories: label: - Under a specific label. The tool accepts label IDs, not display names. Use the list_labels tool to get the ID. category: - In a category (primary, social, promotions, updates, forums, reservations, purchases). in: - Search in specific labels (archive, snoozed, trash, sent, inbox). E.g., `in:trash`, `in:inbox`. Archived and sent messages are included by default; use `-in:archive` and `-in:sent` to exclude them. Drafts are explicitly excluded by default by the tool. Use `in:inbox` to restrict search to the inbox only. has:userlabels - Has any user labels. has:nouserlabels - Does not have any user labels. has:*-star - Specific star colors (if enabled, e.g., has:yellow-star). in:draft - Search in drafts. -in:draft means exclude drafts from the search results. in:sent - Search in sent messages. in:anywhere... (line truncated to 2000 chars)",
      "type": "string"
    },
    "view": {
      "description": "Optional. Controls the fields populated for threads in the thread list.",
      "enum": [
        "THREAD_VIEW_UNSPECIFIED",
        "THREAD_VIEW_METADATA_ONLY",
        "THREAD_VIEW_MINIMAL"
      ],
      "type": "string",
      "x-google-enum-descriptions": [
        "Maps to DRAFT_VIEW_FULL for backward compatibility.",
        "Metadata only: does not include subject, plaintext_body, html_body.",
        "Minimal: includes subject and snippet, but no body."
      ]
    }
  },
  "description": "Request message for SearchThreads RPC."
}
```

### unlabel_message

从已认证用户的 Gmail 帐户中的特定邮件移除一个或多个标签。要查找邮件 ID，请使用 `search_threads` 或 `get_thread` 等工具。如果不确定用户标签的 ID，请先使用 `list_labels` 工具发现可用标签及其 ID。

```json
{
  "type": "object",
  "properties": {
    "labelIds": {
      "description": "Required. The IDs of the labels to remove. Can be a system label ID (e.g., 'INBOX', 'TRASH', 'SPAM', 'STARRED', 'UNREAD', 'IMPORTANT') or a user-defined label ID. The tool accepts `label_ids` and not label names. Use the list_labels tool to get the corresponding label id to a display name for user-defined labels.",
      "items": {
        "type": "string"
      },
      "type": "array"
    },
    "messageId": {
      "description": "Required. The ID of the message to remove the labels from.",
      "type": "string"
    }
  },
  "required": [
    "messageId",
    "labelIds"
  ],
  "description": "Request message for UnlabelMessage RPC."
}
```

### unlabel_thread

从已认证用户的 Gmail 帐户中的整个会话移除标签。如果不确定会话 ID，请先使用 `search_threads` 工具。如果不确定用户标签的 ID，请先使用 `list_labels` 工具。

```json
{
  "type": "object",
  "properties": {
    "labelIds": {
      "description": "Required. The unique identifiers of the labels to remove. Can be a system label ID (e.g., 'INBOX', 'TRASH', 'SPAM', 'STARRED', 'UNREAD', 'IMPORTANT') or a user-defined label ID. The tool accepts `label_ids` and not label names. Use the list_labels tool to get the corresponding label id to a display name for user-defined labels.",
      "items": {
        "type": "string"
      },
      "type": "array"
    },
    "threadId": {
      "description": "Required. The unique identifier of the thread to remove labels from.",
      "type": "string"
    }
  },
  "required": [
    "threadId",
    "labelIds"
  ],
  "description": "Request message for UnlabelThread RPC."
}
```
