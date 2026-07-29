# 工具使用概念

本文件涵盖 Claude API 工具使用的概念基础。如需特定语言的代码示例，请参见 `python/`、`typescript/` 或其他语言文件夹。如需关于暴露哪些工具、如何在长时间运行的智能体中管理上下文以及缓存策略的决策启发式方法，请参见 `agent-design.md`。

## 用户定义工具

### 工具定义结构

> **注意：** 使用 Tool Runner（测试版）时，工具 schema 会从你的函数签名（Python）、Zod schema（TypeScript）、带注解的类（Java）、`jsonschema` struct 标签（Go）或 `BaseTool` 子类（Ruby）自动生成。下方的原始 JSON schema 格式适用于手动方式，包括 PHP 的 `BetaRunnableTool`（在手写 schema 外包一层 run 闭包），或不支持 tool runner 的 SDK。

每个工具需要一个名称、描述，以及输入的 JSON Schema：

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City and state, e.g., San Francisco, CA"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "Temperature unit"
      }
    },
    "required": ["location"]
  }
}
```

**工具定义最佳实践：**

- 使用清晰、描述性的名称（如 `get_weather`、`search_database`、`send_email`）
- 编写详细的描述，Claude 会根据这些描述来决定何时使用工具。要**明确规定*何时*调用它**，而不仅仅是它做什么（例如"当用户询问当前价格或近期事件时调用此工具"）。在较新的 Opus 模型上，模型会更保守地使用工具，描述中的触发条件能显著提升应该调用的比率。
- 为每个属性包含描述
- 对有固定值集合的参数使用 `enum`
- 将真正必需的参数标记在 `required` 中；其他参数设为可选并带默认值

---

### 工具选择选项

控制 Claude 何时使用工具：

| 值                                | 行为                                            |
| --------------------------------- | ----------------------------------------------- |
| `{"type": "auto"}`                | Claude 自行决定是否使用工具（默认）              |
| `{"type": "any"}`                 | Claude 必须使用至少一个工具                      |
| `{"type": "tool", "name": "..."}` | Claude 必须使用指定的工具                        |
| `{"type": "none"}`                | Claude 不能使用工具                              |

任何 `tool_choice` 值都可以包含 `"disable_parallel_tool_use": true`，强制 Claude 在每次响应中最多使用一个工具。默认情况下，Claude 可能在单次响应中请求多个工具调用。

---

### Tool Runner 与手动循环

**Tool Runner（推荐）：** SDK 的 tool runner 自动处理智能体循环，它会调用 API、检测工具使用请求、执行你的工具函数、将结果反馈给 Claude，然后重复，直到 Claude 停止调用工具。在 Python、TypeScript、Java、Go、Ruby、PHP 和 C# SDK 中可用（测试版）。Python SDK 还提供 MCP 转换辅助函数（`anthropic.lib.tools.mcp`），用于转换 MCP 工具、提示和资源以配合 tool runner 使用，详见 `python/claude-api/tool-use.md`。对于任何自定义工具智能体，**默认使用 tool runner**。

**Tool runner 不是黑盒，"我需要控制"很少是退回手动循环的理由。** 每次迭代会在工具运行*之前*产出助手消息，并允许你介入，因此大多数"细粒度控制"需求无需手写循环即可满足：

- **人工审批门控：** 在工具的 run 函数中设门（返回"用户拒绝"结果而不执行），或在产出的消息中检查工具调用，并用 `set_messages_params()` / `setMessagesParams()` / `append_messages()` / `pushMessages()` 覆盖待处理请求，在工具执行*之前*允许或拒绝。runner 仅在你未介入时自动运行你的函数。
- **错误拦截：** 在工具结果返回给 Claude 之前检查它（`generate_tool_call_response()` / `generateToolResponse()`）；提前停止或自行处理。
- **结果修改：** 在工具结果返回之前修改它（例如为 prompt caching 添加 `cache_control`，或转换输出）。
- **逐轮重试/参数变更：** 例如增大 `max_tokens` 并重新运行被截断的轮次；用 `max_iterations` 限制整个循环。
- **流式传输和自动压缩** 均受支持。

这些钩子是 SDK 的辅助功能，不是独立的 API 参数。如需确切的方法名和示例，请用 WebFetch 查阅 `shared/live-sources.md` → *Claude API SDK Repositories* 中列出的各语言 SDK 仓库（tool runner 辅助函数位于各仓库的 `tools.md` / `helpers.md` 中）。随附的 `python/claude-api/tool-use.md` 和 `typescript/claude-api/tool-use.md` 展示了基本的 tool runner 设置。

**不要因为以下误解而退回手动循环：**

- Tool runner 不要求使用 Zod/Pydantic，`betaTool()`（TS）和 `@beta_tool`（Python）接受原始 JSON Schema；其他 SDK 使用普通结构体/映射/类。
- Runner 让检测最终轮次*更容易*而非更难，当 Claude 停止调用工具时迭代结束，最后产出的消息即最终响应。大多数 SDK 还提供一次性变体（`runner.until_done()` / `runner.runUntilDone()` / `RunToCompletion()`）。
- 确认/审批门控可与 runner 配合使用（参见下文安全部分）。

**手动智能体循环：** 仅在你想要拥有*整个*循环时才使用此方式，即你需要 runner 未暴露的控制（例如自定义传输、SDK 无法构建的请求形状、在不支持 runner 的 SDK 上进行逐 token 流式传输），或者你不想引入测试版依赖，或者你的控制流不符合 runner 的逐轮钩子（例如在循环中间插入了不相关的工作）。审批门控、日志记录、拦截、结果修改和条件执行**不**需要手动循环，tool runner 已涵盖这些（见上文）。循环直到 `stop_reason == "end_turn"`，始终追加完整的 `response.content` 以保留 tool_use 块，并确保每个 `tool_result` 包含匹配的 `tool_use_id`。

**服务端工具的停止原因：** 使用服务端工具（代码执行、网页搜索等）时，API 会运行服务端采样循环。如果此循环达到默认的 10 次迭代限制，响应将带有 `stop_reason: "pause_turn"`。要继续，重新发送用户消息和助手响应并再发一次 API 请求即可，服务端会从中断处恢复。**不要**添加类似"继续"的额外用户消息，API 会检测尾部的 `server_tool_use` 块并自动知道要恢复。

```python
# Handle pause_turn in your agentic loop
if response.stop_reason == "pause_turn":
    messages = [
        {"role": "user", "content": user_query},
        {"role": "assistant", "content": response.content},
    ]
    # Make another API request — server resumes automatically
    response = client.messages.create(
        model="claude-opus-4-8", messages=messages, tools=tools
    )
```

**注意：** SDK tool runner 不会自动恢复 `pause_turn`（截至 `@anthropic-ai/sdk` 0.110.0 / `anthropic` 0.116.0），暂停的轮次会结束 runner 并作为最终消息返回，不会报错。在 TypeScript 中你可以在迭代体内恢复（将暂停的助手轮次推回 runner）；在 Python 中 runner 无法在循环中间恢复，需要启动一个新 runner 并追加暂停的轮次，或在手动循环中处理 `pause_turn`。请参阅各语言的 `tool-use.md` 了解具体模式。

设置 `max_continuations` 限制（如 5）以防止无限循环。完整指南请参见：`https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons`

> **安全：** Tool runner 会在 Claude 请求时自动执行你的工具函数。对于有副作用的工具（发送邮件、修改数据库、金融交易），请验证输入并将破坏性操作置于人工审批之后。Tool runner 和手动循环**都**支持这一点。使用 tool runner 时，在工具的 run 函数中设门（提示用户并返回"用户拒绝"结果而不执行），或在每次产出的消息中检查工具调用，用 `set_messages_params()` / `setMessagesParams()` 在工具运行*之前*接管消息历史以允许或拒绝（仅在你未介入时才自动执行你的函数）；使用手动循环时，在调用函数之前内联设门。

---

### 处理工具结果

当 Claude 使用工具时，响应包含一个 `tool_use` 块。你必须：

1. 使用提供的输入执行工具
2. 在 `tool_result` 消息中发送结果
3. 继续对话

**工具结果中的错误处理：** 当工具执行失败时，设置 `"is_error": true` 并提供有意义的错误消息。Claude 通常会确认错误，然后尝试不同的方法或请求澄清。

**多个工具调用：** Claude 可以在单次响应中请求多个工具。在继续之前全部处理完毕，在单条 `user` 消息中发回所有结果。

---

## 服务端工具：代码执行

代码执行工具让 Claude 在安全的沙盒容器中运行代码。与用户定义工具不同，服务端工具在 Anthropic 的基础设施上运行，你无需在客户端执行任何操作。只需包含工具定义，Claude 会处理其余部分。

### 关键信息

- 在隔离容器中运行（1 CPU、5 GiB RAM、5 GiB 磁盘）
- 无互联网访问（完全沙盒化）
- Python 3.11，预装数据科学库
- 容器保留 30 天，可跨请求重用
- 与 web search/web fetch 工具一起使用时免费；否则在每月每组织 1,550 小时免费额度后按 $0.05/小时计费

### 工具定义

该工具不需要 schema，只需在 `tools` 数组中声明：

```json
{
  "type": "code_execution_20260120",
  "name": "code_execution"
}
```

Claude 自动获得 `bash_code_execution`（运行 shell 命令）和 `text_editor_code_execution`（创建/查看/编辑文件）的访问权限。

### 预装 Python 库

- **数据科学**：pandas、numpy、scipy、scikit-learn、statsmodels
- **可视化**：matplotlib、seaborn
- **文件处理**：openpyxl、xlsxwriter、pillow、pypdf、pdfplumber、python-docx、python-pptx
- **数学**：sympy、mpmath
- **实用工具**：tqdm、python-dateutil、pytz、sqlite3

可在运行时通过 `pip install` 安装额外的包。

### 支持上传的文件类型

| 类型     | 扩展名                              |
| -------- | ----------------------------------- |
| 数据     | CSV、Excel (.xlsx/.xls)、JSON、XML  |
| 图像     | JPEG、PNG、GIF、WebP                |
| 文本     | .txt、.md、.py、.js 等              |

### 容器重用

跨请求重用容器以保持状态（文件、已安装的包、变量）。从第一次响应中提取 `container_id`，并在后续请求中传入。

### 响应结构

响应包含交替的文本和工具结果块：

- `text` — Claude 的解释
- `server_tool_use` — Claude 正在做什么
- `bash_code_execution_tool_result` — 代码执行输出（检查 `return_code` 判断成功/失败）
- `text_editor_code_execution_tool_result` — 文件操作结果

> **安全：** 在将下载的文件写入磁盘之前，始终用 `os.path.basename()` / `path.basename()` 对文件名进行净化，以防止路径遍历攻击。将文件写入专用输出目录。

---

## 服务端工具：Web Search 和 Web Fetch

Web search 和 web fetch 让 Claude 搜索网页并获取页面内容。它们在服务端运行，只需包含工具定义，Claude 会自动处理查询、获取和结果处理。

### 工具定义

```json
[
  { "type": "web_search_20260209", "name": "web_search" },
  { "type": "web_fetch_20260209", "name": "web_fetch" }
]
```

### 动态过滤（Fable 5 / Opus 4.8 / Opus 4.7 / Opus 4.6 / Sonnet 4.6）

`web_search_20260209` 和 `web_fetch_20260209` 版本支持**动态过滤**，Claude 会编写并执行代码来过滤搜索结果，防止它们进入上下文窗口，从而提高准确性和 token 效率。动态过滤内置于这些工具版本中并自动激活，你无需单独声明 `code_execution` 工具或传递任何 beta 头。

```json
{
  "tools": [
    { "type": "web_search_20260209", "name": "web_search" },
    { "type": "web_fetch_20260209", "name": "web_fetch" }
  ]
}
```

如不使用动态过滤，也可使用之前的 `web_search_20250305` 版本。

> **注意：** 仅当你的应用出于自身目的（数据分析、文件处理、可视化）需要独立于网页搜索的代码执行时，才包含独立的 `code_execution` 工具。将其与 `_20260209` web 工具一起包含会创建第二个执行环境，可能使模型产生混淆。

---

## 服务端工具：程序化工具调用

在标准工具使用中，每次工具调用都是一次往返：Claude 调用、结果进入 Claude 的上下文、Claude 推理、然后调用下一个工具。链式调用会累积延迟和 token，而大部分中间数据后续不再需要。

程序化工具调用让 Claude 将这些调用组合成脚本。脚本在代码执行容器中运行；当它调用工具时，容器暂停，调用执行，结果返回给运行中的代码（而非 Claude 的上下文）。脚本用正常的控制流处理结果，只有最终输出返回给 Claude。当链接大量工具调用或中间结果较大、需要在进入上下文窗口之前过滤时使用。

完整文档请用 WebFetch：

- URL: `https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling`

---

## 服务端工具：工具搜索

工具搜索工具让 Claude 从大型库中动态发现工具，而无需将所有定义加载到上下文窗口中。当你有很多工具但只有少数与任何给定请求相关时使用。发现的工具 schema 会追加到请求中，而非替换，这保留了 prompt 缓存（参见 `agent-design.md` §Caching for Agents）。

完整文档请用 WebFetch：

- URL: `https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool`

---

## 智能体技能（Messages API）

智能体技能打包了特定任务的指令和文件，Claude 会在相关时加载它们（例如 Anthropic 预构建的 `pptx`、`xlsx`、`pdf`、`docx` 技能）。在 **Messages API** 上，技能通过 `container` 参数与代码执行工具一起启用，这**不是** Managed Agents 界面，也**不**使用 `client.beta.agents` / `sessions` / `environments`。可用性：参见 `shared/platform-availability.md`。

每次请求需要：

1. `client.beta.messages.create(...)` 带有**两个** beta 标志：`code-execution-2025-08-25` **和** `skills-2025-10-02`。
2. `container={"skills": [{"type": "anthropic", "skill_id": "<id>", "version": "latest"}]}`，skills 列表选择在执行容器中可用的技能。
3. `tools=[{"type": "code_execution_20260521", "name": "code_execution"}]`，技能通过容器中的代码执行来运行。

```python
response = client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=16000,
    betas=["code-execution-2025-08-25", "skills-2025-10-02"],
    container={"skills": [{"type": "anthropic", "skill_id": "pptx", "version": "latest"}]},
    tools=[{"type": "code_execution_20260521", "name": "code_execution"}],
    messages=[{"role": "user", "content": "Create a 3-slide presentation on X"}],
)
```

生成的文件（`.pptx`、`.xlsx` 等）写入容器内部，响应为每个文件携带一个文件 ID。通过将该 ID 传给 Files API 下载（`client.beta.files.download(file_id)` / `GET /v1/files/{id}/content`，带 `anthropic-beta: files-api-2025-04-14`）。

通过 `GET /v1/skills` 列出可用技能（需要 `anthropic-beta: skills-2025-10-02`）。

---

## MCP Connector（测试版）

MCP connector 让 Claude 直接从 Messages API 调用托管在远程 MCP 服务器上的工具，Anthropic 在服务端建立 MCP 连接。需要在 `client.beta.messages.create(...)` 上使用 beta 标志 `mcp-client-2025-11-20`。可用性：参见 `shared/platform-availability.md`。

**两个参数必须一起使用：**

- `mcp_servers` — 服务器连接定义数组：`[{"type": "url", "url": "<server URL>", "name": "<server-name>", "authorization_token": "<optional>"}]`
- `tools` — 必须包含一个通过名称引用服务器的 `mcp_toolset` 条目：`[{"type": "mcp_toolset", "mcp_server_name": "<server-name>"}]`

toolset 中的 `mcp_server_name` 必须与 `mcp_servers` 中的某个 `name` 匹配。省略 `mcp_toolset` 条目会被作为验证错误拒绝，`mcp_servers` 中的每个服务器必须恰好被一个 toolset 引用。

```python
client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=1024,
    betas=["mcp-client-2025-11-20"],
    mcp_servers=[{"type": "url", "url": "https://example/sse", "name": "example-mcp"}],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "example-mcp"}],
    messages=[...],
)
```

Go 使用类型化常量 `anthropic.AnthropicBetaMCPClient2025_11_20`，旧的 `…2025_04_04` 常量已弃用。

可选 toolset 字段：`default_config`（所有工具的默认值，如 `{"enabled": false}` 用于白名单模式）和 `configs`（按工具名称键入的逐工具覆盖）。

---

## 工具使用示例

你可以在工具定义中直接提供示例工具调用，以演示使用模式并减少参数错误。这有助于 Claude 理解如何正确格式化工具输入，特别是对于具有复杂 schema 的工具。

完整文档请用 WebFetch：

- URL: `https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use`

---

## 客户端工具：Computer Use

Computer use 让 Claude 与桌面环境交互（截图、鼠标、键盘）。它是客户端工具，你的应用提供环境并执行 Claude 请求的操作，Anthropic 实时处理截图和操作请求，但不托管环境或保留数据。

完整文档请用 WebFetch：

- URL: `https://platform.claude.com/docs/en/agents-and-tools/computer-use/overview`

---

## 上下文编辑

上下文编辑会在长时间运行的智能体累积轮次时，从对话记录中清除过时的工具结果和 thinking 块。与压缩（会生成摘要）不同，上下文编辑是修剪，被清除的内容会被移除而非替换。当旧的工具输出不再相关、你想在不丢失对话结构的情况下保持对话记录精简时使用。

**测试版。** 使用 `client.beta.messages.*` 和 beta `context-management-2025-06-27`。通过 `context_management.edits` 配置，策略类型为 `clear_tool_uses_20250919`（清除旧工具结果，可选 `clear_tool_inputs: true` 同时清除 tool_use 参数）或 `clear_thinking_20251015`（清除 thinking 块）。这些**不是**压缩类型，`compact_20260112` 配合 beta `compact-2026-01-12` 是独立的压缩功能。

完整文档请用 WebFetch：

- URL: `https://platform.claude.com/docs/en/build-with-claude/context-editing`

---

## 服务端工具：Advisor（测试版）

Advisor 工具将一个更快、成本更低的**执行器**模型（请求中的顶层 `model`）与一个更高智能的**顾问**模型（工具定义中的 `model` 字段）配对，后者在生成过程中提供策略指导。执行器负责大部分 token 生成，顾问用于规划咨询。可用性：参见 `shared/platform-availability.md`。

### 工具定义

```json
{
  "type": "advisor_20260301",
  "name": "advisor",
  "model": "claude-opus-4-8"
}
```

**顾问模型必须至少与执行器同等能力。** 无效配对会返回 `400 invalid_request_error`。有效配对：

| 执行器（请求 `model`） | 有效顾问（工具 `model`） |
|---|---|
| `claude-haiku-4-5` / `claude-sonnet-4-6` / `claude-sonnet-5` / `claude-opus-4-6` / `claude-opus-4-7` | `claude-opus-4-8` 或 `claude-opus-4-7` |
| `claude-opus-4-8` | 仅 `claude-opus-4-8` |

通过 `client.beta.messages.create(...)` 调用，带 `betas=["advisor-tool-2026-03-01"]`（或 `anthropic-beta: advisor-tool-2026-03-01` 头）。在多轮对话中，将完整的 `response.content`（包括任何 `advisor_tool_result` 块）追加到下一轮的 `messages` 中。如果在后续轮次中从 `tools` 中移除了 advisor 工具，而历史记录中仍包含 `advisor_tool_result` 块，API 会返回 400。

---

## 客户端工具：Memory

Memory 工具让 Claude 通过 memory 文件目录跨对话存储和检索信息。Claude 可以创建、读取、更新和删除在会话间持久存在的文件。

### 关键信息

- 客户端工具，你通过实现来控制存储
- 支持命令：`view`、`create`、`str_replace`、`insert`、`delete`、`rename`
- 操作 `/memories` 目录中的文件
- Python、TypeScript 和 Java SDK 提供辅助类/函数来实现 memory 后端

> **安全：** 切勿在 memory 文件中存储 API 密钥、密码、令牌或其他机密信息。谨慎处理个人身份信息（PII），在持久化用户数据之前检查数据隐私法规（GDPR、CCPA）。参考实现没有内置访问控制，在多用户系统中，请在你的工具处理程序中实现每用户 memory 目录和身份验证。

完整实现示例请用 WebFetch：

- 文档: `https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool.md`

---

## 客户端工具：Bash 和 Text Editor

Bash 和 text editor 工具是 **Anthropic 定义的、无 schema** 工具。仅通过 `type` 和 `name` 声明，输入 schema 内置于模型中且无法修改。**不要传 `input_schema`**，也不要定义恰好名为 `"bash"` 的自定义工具，那会创建一个没有内置行为的用户定义工具。

两者都是**客户端执行**的：Claude 返回 `tool_use` 块，你的代码在本地执行操作，然后你发回 `tool_result`。API 是无状态的，你的应用在轮次之间维护 shell 会话或文件系统。

### Bash 工具声明

```json
{"type": "bash_20250124", "name": "bash"}
```

| 语言 | 声明 |
|---|---|
| Python / TypeScript / Ruby / cURL | 普通对象 `{"type": "bash_20250124", "name": "bash"}` |
| Go | `anthropic.ToolUnionParam{OfBashTool20250124: &anthropic.ToolBash20250124Param{}}` |
| Java | `.addTool(ToolBash20250124.builder().build())` from `com.anthropic.models.messages` |
| C# | `Tools = [new ToolBash20250124()]` from `Anthropic.Models.Messages` |
| PHP | `tools: [new \Anthropic\Messages\ToolBash20250124()]` |

Claude 的 `tool_use.input` 包含 `{"command": "<string>"}` 或 `{"restart": true}`。先检查 `restart`（重置会话，返回确认字符串），否则运行 `command` 并返回合并的 stdout + stderr。

> **安全：命令是不可信的模型输出。** 在隔离环境中运行（容器、VM 或受限用户），应用可执行文件**白名单**并拒绝 shell 操作符（`&&`、`|`、`;`、`` ` ``、`$()`），设置超时和资源限制，记录每个命令。仅用黑名单是不够的。

### Text editor 工具声明

```json
{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}
```

可选字段：`max_characters` 用于限制 `view` 输出。Java 提供类型化的 `ToolTextEditor20250728` builder（`com.anthropic.models.messages`），其他静态类型 SDK 遵循相同的命名模式，请参见 `{lang}/claude-api/tool-use.md` 中的 Anthropic-Defined Tools 部分获取确切类名。

> **安全：`path` 是不可信的模型输出。将每个文件操作限制在固定项目根目录内。** 在执行任何命令之前，将模型提供的 `path` 解析为其规范形式，并验证它仍位于项目根目录内；如果逃逸则拒绝请求（`..`、符号链接、根目录外的绝对路径、URL 编码的遍历如 `%2e%2e%2f`）。使用你语言内置的路径工具（如 Python `pathlib.Path.resolve()` 然后检查 `.is_relative_to(root)`）。切勿直接对原始 `path` 值调用 `open()` / `writeFile` / `unlink`。

`tool_use.input.command` 为以下之一：

| `command` | 其他输入 | 操作 |
|---|---|---|
| `view` | `path`、可选 `view_range` | 返回文件内容或目录列表 |
| `create` | `path`、`file_text` | 用 `file_text` 创建/覆盖文件。如文件已存在则创建备份。 |
| `str_replace` | `path`、`old_str`、`new_str` | 替换恰好一处匹配，0 或 >1 匹配时报错 |
| `insert` | `path`、`insert_line`、`insert_text` | 在第 `insert_line` 行之后插入 `insert_text`（0 = 文件开头） |

对于两个工具，出错时返回 `{"type": "tool_result", "tool_use_id": "…", "content": "<error text>", "is_error": true}` 以便 Claude 恢复。

---

## 结构化输出

结构化输出约束 Claude 的响应遵循特定的 JSON schema，保证有效、可解析的输出。这不是一个单独的工具，它增强了 Messages API 的响应格式和/或工具参数验证。

有两个功能可用：

- **JSON 输出**（`output_config.format`）：控制 Claude 的响应格式
- **严格工具使用**（`strict: true`）：保证有效的工具参数 schema

**支持的模型：** Claude Fable 5、Claude Opus 4.8、Claude Sonnet 5 和 Claude Haiku 4.5。旧模型（Claude Opus 4.5、Claude Opus 4.1）也支持结构化输出。

> **推荐：** 使用 `client.messages.parse()`，它会自动根据你的 schema 验证响应。直接使用 `messages.create()` 时，用 `output_config: {format: {...}}`。某些 SDK 方法（如 `.parse()`）也接受 `output_format` 便捷参数，但 `output_config.format` 是规范的 API 级参数。

### JSON Schema 限制

**支持：**

- 基本类型：object、array、string、integer、number、boolean、null
- `enum`、`const`、`anyOf`、`allOf`、`$ref`/`$def`
- 字符串格式：`date-time`、`time`、`date`、`duration`、`email`、`hostname`、`uri`、`ipv4`、`ipv6`、`uuid`
- `additionalProperties: false`（所有对象必需）

**不支持：**

- 递归 schema
- 数值约束（`minimum`、`maximum`、`multipleOf`）
- 字符串约束（`minLength`、`maxLength`）
- 复杂数组约束
- `additionalProperties` 设为 `false` 以外的任何值

Python 和 TypeScript SDK 会自动处理不支持的约束，从发送给 API 的 schema 中移除它们并在客户端验证。

### 重要说明

- **首次请求延迟**：新 schema 会产生一次性编译开销。使用相同 schema 的后续请求使用 24 小时缓存。
- **拒绝**：如果 Claude 因安全原因拒绝（`stop_reason: "refusal"`），输出可能不符合你的 schema。
- **Token 限制**：如果 `stop_reason: "max_tokens"`，输出可能不完整。增大 `max_tokens`。
- **不兼容**：Citations（返回 400 错误）、消息预填充。
- **兼容**：Batches API、流式传输、token 计数、扩展思考。

---

## 有效工具使用技巧

1. **提供详细描述**：Claude 严重依赖描述来理解何时以及如何使用工具
2. **使用具体的工具名称**：`get_current_weather` 比 `weather` 更好
3. **验证输入**：在执行前始终验证工具输入
4. **优雅处理错误**：返回有意义的错误消息以便 Claude 适应
5. **限制工具数量**：太多工具会使模型混淆，保持工具集聚焦
6. **测试工具交互**：验证 Claude 在各种场景下正确使用工具

详细工具使用文档请用 WebFetch：

- URL: `https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview`
