# Claude API — C#

> **注意：** C# SDK 是 Anthropic 官方 C# SDK。工具使用通过 Messages API 支持，并提供 beta 版 `BetaToolRunner` 用于自动工具执行循环。该 SDK 还支持 Microsoft.Extensions.AI IChatClient 集成，提供函数调用和 Managed Agents（beta）功能。

## 命名空间参考

类型按命名空间组织。如果你需要的类型未在下方示例中出现，请先通过此表定位，不要因为通过网络获取 SDK 源码而阻塞。

| `using` | 包含 |
|---|---|
| `Anthropic` | `AnthropicClient`，顶层选项 |
| `Anthropic.Models.Messages` | 非 beta 请求/响应类型 — `MessageCreateParams`、`Model`、`Role`、`ContentBlock`、`TextBlock`、`ToolUseBlock`、`ToolResultBlockParam`、`Tool*`（工具定义类） |
| `Anthropic.Models.Beta.Messages` | beta 端点等价类型 — `MessageCreateParams`、`BetaMessage`、`BetaTool*`、`Speed`、`BetaRequestMcpServerUrlDefinition`，上下文编辑/压缩配置 |
| `Anthropic.Models.Beta` | 共享 beta 常量 |
| `Anthropic.Models.Beta.Files` | Files API 类型 |
| `Anthropic.Models.Messages.Batches` | Batch API 类型 |
| `Anthropic.Helpers.Beta` | `BetaToolRunner`，beta 辅助工具 |
| `Anthropic.Exceptions` | `AnthropicApiException`、`AnthropicRateLimitException`、`Anthropic5xxException` 等 — 参见 `shared/error-codes.md` |
| `Anthropic.Bedrock` / `Anthropic.Vertex` / `Anthropic.Foundry` / `Anthropic.Aws` | 平台客户端（独立 NuGet 包）：`AnthropicBedrockMantleClient`、`AnthropicFoundryClient`、`AnthropicAwsClient` |

`client.Messages.*` 使用非 beta 类型；`client.Beta.Messages.*` 使用 `Anthropic.Models.Beta.Messages` 类型。两个命名空间都定义了 `MessageCreateParams`，请选择与你调用的客户端路径匹配的那个。

### 各功能关键类型

请从此表中编写代码，而非反射 SDK 程序集。Endpoint 列告诉你使用 `client.Messages.*` 还是 `client.Beta.Messages.*`。

| 功能 | Endpoint | 关键 C# 类型（命名空间见上表） |
|---|---|---|
| 用户画像 | beta | `client.Beta.UserProfiles.Create(...)` / `.Retrieve(id)` / `.List()`。将返回的 profile id 传入 beta messages 调用。需要 beta header，请查看 SDK 的 beta-headers 参考以获取当前标志。 |
| Agent Skills | beta | `BetaContainerParams`（带 `Skills = [new BetaSkillParams { ... }]`）、`BetaCodeExecutionTool20250825`。`Betas = ["code-execution-2025-08-25", "skills-2025-10-02"]`。通过 `client.Beta.Files.Download(fileId)` 下载输出。 |
| Advisor 工具 | beta | `BetaAdvisorTool20260301`，可能尚未在所有 SDK 版本中提供 |
| 缓存诊断 | beta | `Diagnostics = new() { PreviousMessageID = … }`、`BetaCacheControlEphemeral`、`BetaContentBlockParam` |
| 上下文编辑 | beta | `ContextManagement = new BetaContextManagementConfig { Edits = [new BetaClearToolUses20250919Edit()] }`。`Betas = ["context-management-2025-06-27"]`（不是 `compact-2026-01-12`，那是用于 `BetaCompact20260112Edit` 的）。 |
| Memory 工具 | non-beta | `Tools = [new ToolUnion(new MemoryTool20250818())]` |
| 编程式工具调用 | non-beta | `CodeExecutionTool20260120`、`ToolResultBlockParam`、`ContentBlockParam` |
| 任务预算 | beta | `BetaOutputConfig` 配合 `TaskBudget = new BetaTokenTaskBudget { ... }` |
| 工具搜索 | non-beta | `new ToolUnion(new ToolSearchToolRegex20251119 { Type = ToolSearchToolRegex20251119Type.ToolSearchToolRegex20251119 })`，`Type` 必须显式设置。 |
| 网页搜索 | non-beta | `new ToolUnion(new WebSearchTool20260209())`，最新变体，支持动态过滤（Opus 4.8/4.7/4.6 + Sonnet 4.6）。对于旧模型或 Vertex，使用 `WebSearchTool20250305()` |

### 发现类型和成员名称

如果你需要的类型或成员不在上表中，`strings ~/.nuget/packages/anthropic/*/lib/*/Anthropic.dll | grep -i <term>` 足够快速地定位类和属性名称。**不要升级到 `dotnet run` 反射探针**来精确导出成员，因为首次编译在很多环境中足够慢以至于需要后台执行，会让你陷入轮询循环。相反，使用 `strings | grep` 找到的名称编写 `Program.cs`；如果成员名称错误，编译器错误（`error CS1061: 'X' does not contain a definition for 'Y'`）会在几秒内指出问题，比任何反射探针都快。

注意 `strings` 不会显示传输格式的 snake_case 字段名（`output_tokens`、`stop_reason`），它们在 DLL 中以不同方式存储。**C# 属性是传输字段的 PascalCase 等价物**（`response.Usage.OutputTokens`、`response.StopReason`）。如果你从文档中知道传输字段名，直接写 PascalCase 属性并编译，不要探查 snake_case 字符串。

### 最小可用骨架

**编写普通的 `Program.cs` 主体**，即 `using` 语句后跟顶层语句，如下所示。**不要**添加 `#!/usr/bin/env dotnet` shebang 或 `#:package Anthropic@*` 指令：这些是 .NET 基于文件的应用语法，当文件通过现有的 `.csproj` 编译时会因 `CS1024: Preprocessor directive expected` 而失败。标准项目设置（参见 [C# 快速入门](https://platform.claude.com/docs/en/get-started)：`dotnet new console` → `dotnet add package Anthropic` → 编辑 `Program.cs` → `dotnet run`）提供了 `.csproj` 和包引用。

从此模板开始，它可以直接编译。填入特定功能的字段，不要花时间运行反射或 XML 文档检查来先发现类型名称。

```csharp
using System;
using Anthropic;
using Anthropic.Models.Messages;       // or Anthropic.Models.Beta.Messages for beta endpoints

AnthropicClient client = new();

var message = await client.Messages.Create(new MessageCreateParams
{
    Model = Model.ClaudeOpus4_8,
    MaxTokens = 1024,
    Messages = [ new() { Role = Role.User, Content = "Hello, Claude" } ],
});

Console.WriteLine(message);
```

对于 beta 功能（任何在 `anthropic-beta` header 之后的功能），使用 beta 客户端路径和命名空间，整体结构相同：

```csharp
using System;
using Anthropic;
using Anthropic.Models.Beta.Messages;

AnthropicClient client = new();

var response = await client.Beta.Messages.Create(new MessageCreateParams
{
    Model = "claude-opus-4-8",
    MaxTokens = 4096,
    Betas = ["<beta-flag>"],
    Messages = [ new() { Role = Role.User, Content = "…" } ],
    // Tools = new BetaToolUnion[] { new BetaSomeTool { … } },   // for tool features
});

Console.WriteLine(response);
```

如果功能所需的类型名称不在此文件中，请按照上方命名空间参考中的命名模式编写，并根据编译器输出修正。编写 `Program.cs` 并迭代比研究更高效。

### 常见 C# 编译错误

- **CS8803（顶层语句必须先于类型声明）：**将任何 `record`/`class`/`struct` 定义放在最后一条顶层语句**之后**，即文件末尾。在 `var client = new AnthropicClient()` 之上定义 record 将无法编译。
- **对 `Task<…Page>` 使用 `await foreach`：**`client.Models.List()` 返回 `Task<ModelListPage>`，不能直接异步枚举。先 await 再迭代：`var page = await client.Models.List(); foreach (var m in page.Items) {…}`。如需自动分页，在尝试 `await foreach` 之前先检查页面类型是否暴露 `AutoPagingEachAsync()` 或类似方法。

## 安装

```bash
dotnet add package Anthropic
```

## 客户端初始化

```csharp
using Anthropic;

// 默认（使用 ANTHROPIC_API_KEY 环境变量）
AnthropicClient client = new();

// 显式 API key（使用环境变量，绝不硬编码 key）
AnthropicClient client = new() {
    ApiKey = Environment.GetEnvironmentVariable("ANTHROPIC_API_KEY")
};
```

---

## 基本消息请求

```csharp
using Anthropic.Models.Messages;

var parameters = new MessageCreateParams
{
    Model = Model.ClaudeOpus4_8,
    MaxTokens = 16000,
    Messages = [new() { Role = Role.User, Content = "What is the capital of France?" }]
};
var response = await client.Messages.Create(parameters);

// ContentBlock 是联合类型包装器。.Value 解包为变体对象，
// 然后 OfType<T> 过滤到你想要的类型。或使用下方 Thinking 章节中
// 展示的 TryPick* 惯用法。
foreach (var text in response.Content.Select(b => b.Value).OfType<TextBlock>())
{
    Console.WriteLine(text.Text);
}
```

---

## Thinking

**自适应思考是 Claude 4.6+ 模型的推荐模式。** Claude 动态决定何时思考以及思考多少。

> **Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：**使用自适应思考（见下文）。`new ThinkingConfigEnabled { BudgetTokens = N }` 在 Fable 5、Opus 4.8 和 4.7 上已移除（如果发送返回 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。
> **旧模型：**使用 `new ThinkingConfigEnabled { BudgetTokens = N }`（预算必须 < `MaxTokens`，最小 1024）。

```csharp
using Anthropic.Models.Messages;

var response = await client.Messages.Create(new MessageCreateParams
{
    Model = Model.ClaudeOpus4_8,
    MaxTokens = 16000,
    // ThinkingConfigParam? 从具体变体类隐式转换，
    // 不需要包装器。
    // 显示选项：Fable 5 / Mythos 5 / Opus 4.8 / 4.7 上默认省略（空 thinking 文本）
    Thinking = new ThinkingConfigAdaptive { Display = Display.Summarized },
    Messages =
    [
        new() { Role = Role.User, Content = "Solve: 27 * 453" },
    ],
});

// ThinkingBlock(s) 在 Content 中先于 TextBlock。TryPick* 缩小联合类型。
foreach (var block in response.Content)
{
    if (block.TryPickThinking(out ThinkingBlock? t))
    {
        Console.WriteLine($"[thinking] {t.Thinking}");
    }
    else if (block.TryPickText(out TextBlock? text))
    {
        Console.WriteLine(text.Text);
    }
}
```

`TryPick*` 的替代方案：`.Select(b => b.Value).OfType<ThinkingBlock>()`（与基本消息示例相同的 LINQ 模式）。

---

## 上下文编辑 / 压缩（Beta）

**Beta 命名空间前缀不一致**（已根据 `src/Anthropic/Models/Beta/Messages/*.cs` @ 12.9.0 源码验证）。无前缀：`MessageCreateParams`、`MessageCountTokensParams`、`Role`、`Speed`。**其他所有类型都有 `Beta` 前缀**：`BetaMessageParam`、`BetaMessage`、`BetaContentBlock`、`BetaToolUseBlock`，所有 block param 类型。无前缀的 `Role` 在你同时导入两个命名空间时会与 `Anthropic.Models.Messages.Role` 冲突（CS0104）。最安全的做法：只导入 Beta；如果混合使用，给 beta 的 `Role` 起别名：

```csharp
using Anthropic.Models.Beta.Messages;
using NonBeta = Anthropic.Models.Messages;  // only if you also need non-beta types
// Now: MessageCreateParams, BetaMessageParam, Role (beta's), NonBeta.Role (if needed)
```


`BetaMessage.Content` 是 `IReadOnlyList<BetaContentBlock>`，一个 15 变体的判别联合。用 `TryPick*` 缩小。**响应 `BetaContentBlock` 不能赋值给参数 `BetaContentBlockParam`**，C# 中没有 `.ToParam()`。通过逐个转换每个 block 来完成往返：

```csharp
using Anthropic.Models.Beta.Messages;

var betaParams = new MessageCreateParams   // no Beta prefix — see unprefixed list above
{
    Model = Model.ClaudeOpus4_8,
    MaxTokens = 16000,
    Betas = ["compact-2026-01-12"],
    ContextManagement = new BetaContextManagementConfig
    {
        Edits = [new BetaCompact20260112Edit()],
    },
    Messages = messages,
};
BetaMessage resp = await client.Beta.Messages.Create(betaParams);

foreach (BetaContentBlock block in resp.Content)
{
    if (block.TryPickCompaction(out BetaCompactionBlock? compaction))
    {
        // Content is nullable — compaction can fail server-side
        Console.WriteLine($"compaction summary: {compaction.Content}");
    }
}

// Context-edit metadata lives on a separate nullable field
if (resp.ContextManagement is { } ctx)
{
    foreach (var edit in ctx.AppliedEdits)
        Console.WriteLine($"cleared {edit.ClearedInputTokens} tokens");
}

// ROUND-TRIP: BetaMessageParam.Content is BetaMessageParamContent (a string|list
// union). It implicit-converts from List<BetaContentBlockParam>, NOT from the
// response's IReadOnlyList<BetaContentBlock>. Convert each block:
List<BetaContentBlockParam> paramBlocks = [];
foreach (var b in resp.Content)
{
    if (b.TryPickText(out var t)) paramBlocks.Add(new BetaTextBlockParam { Text = t.Text });
    else if (b.TryPickCompaction(out var c)) paramBlocks.Add(new BetaCompactionBlockParam { Content = c.Content });
    // ... other variants as needed
}
messages.Add(new BetaMessageParam { Role = Role.Assistant, Content = paramBlocks });
```

所有 15 个 `BetaContentBlock.TryPick*` 变体：`Text`、`Thinking`、`RedactedThinking`、`ToolUse`、`ServerToolUse`、`WebSearchToolResult`、`WebFetchToolResult`、`CodeExecutionToolResult`、`BashCodeExecutionToolResult`、`TextEditorCodeExecutionToolResult`、`ToolSearchToolResult`、`McpToolUse`、`McpToolResult`、`ContainerUpload`、`Compaction`。

**`BetaToolUseBlock.Input` 是 `IReadOnlyDictionary<string, JsonElement>`**，按键索引然后调用 `JsonElement` 提取器：

```csharp
if (block.TryPickToolUse(out BetaToolUseBlock? tu))
{
    int a = tu.Input["a"].GetInt32();
    string s = tu.Input["name"].GetString()!;
}
```

---

## Effort 参数

Effort 嵌套在 `OutputConfig` 下，不是顶层属性。`ApiEnum<string, Effort>` 有从枚举的隐式转换，因此直接赋值 `Effort.High`。

```csharp
OutputConfig = new OutputConfig { Effort = Effort.High },
```

值：`Effort.Low`、`Effort.Medium`、`Effort.High`、`Effort.Max`。与 `Thinking = new ThinkingConfigAdaptive()` 结合使用以控制成本和质量。

---

## Prompt Caching

`System` 接受 `MessageCreateParamsSystem?`，即 `string` 或 `List<TextBlockParam>` 的联合。没有 `SystemTextBlockParam`，使用普通的 `TextBlockParam`。隐式转换需要具体的 `List<TextBlockParam>` 类型（数组字面量不会转换）。关于放置模式和静默失效因子审计清单，参见 `shared/prompt-caching.md`。

```csharp
System = new List<TextBlockParam> {
    new() {
        Text = longSystemPrompt,
        CacheControl = new CacheControlEphemeral(),  // auto-sets Type = "ephemeral"
    },
},
```

`CacheControlEphemeral` 上的可选 `Ttl`：`new() { Ttl = Ttl.Ttl1h }` 或 `Ttl.Ttl5m`。`CacheControl` 也存在于 `Tool.CacheControl` 和顶层 `MessageCreateParams.CacheControl` 上。

通过 `response.Usage.CacheCreationInputTokens` / `response.Usage.CacheReadInputTokens` 验证命中。

---

## Token 计数

```csharp
MessageTokensCount result = await client.Messages.CountTokens(new MessageCountTokensParams {
    Model = Model.ClaudeOpus4_8,
    Messages = [new() { Role = Role.User, Content = "Hello" }],
});
long tokens = result.InputTokens;
```

`MessageCountTokensParams.Tools` 使用与 `MessageCreateParams.Tools`（`ToolUnion`）不同的联合类型（`MessageCountTokensTool`），如果你要传递工具，编译器会在需要时告诉你。

---

## PDF / 文档输入

`DocumentBlockParam` 接受 `DocumentBlockParamSource` 联合：`Base64PdfSource` / `UrlPdfSource` / `PlainTextSource` / `ContentBlockSource`。`Base64PdfSource` 自动设置 `MediaType = "application/pdf"` 和 `Type = "base64"`。

```csharp
new MessageParam {
    Role = Role.User,
    Content = new List<ContentBlockParam> {
        new DocumentBlockParam { Source = new Base64PdfSource { Data = base64String } },
        new TextBlockParam { Text = "Summarize this PDF" },
    },
}
```

---

## Fast Mode（Beta）

```csharp
var response = await client.Beta.Messages.Create(new MessageCreateParams {
    Model = "claude-opus-4-8", MaxTokens = 4096,
    Speed = Speed.Fast,
    Betas = ["fast-mode-2026-02-01"],
    Messages = [...],
});
```

## Models API

```csharp
var page = await client.Models.List();
foreach (var m in page.Items) { Console.WriteLine(m.ID); }
var one = await client.Models.Retrieve("claude-opus-4-8");
```

## 长输出（128k）+ 预填充

在 `client.Messages` 上设置 `MaxTokens = 128000` 并使用流式路径（参见 `streaming.md`）。在 Claude 4+ 模型上，128k 输出是原生的，不需要 `output-128k-*` beta header 或 beta 命名空间。

**预填充助手消息**（在输入中放置尾部的 `Role.Assistant` 消息）在 Claude Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6 上**不受支持**，请求会返回 400。请改用 `OutputConfig.Format`（结构化输出）。

## 停止详情

当 `StopReason` 为 `"refusal"` 时，响应包含结构化的 `StopDetails`：

```csharp
if (response.StopReason == "refusal" && response.StopDetails is { } details)
{
    Console.WriteLine($"Category: {details.Category}");
    Console.WriteLine($"Explanation: {details.Explanation}");
}
```

**拒绝回退（Claude Fable 5），默认选择加入。** 回退是可选的：不加入则被拒绝的请求直接停止。在 `claude-fable-5` 代码中，默认包含 `Fallbacks = [new(Model.ClaudeOpus4_8)]` 加上 beta messages 调用上的 `AnthropicBeta.ServerSideFallback2026_06_01` beta，策略拒绝会在同一调用中由回退模型重新服务。在 Amazon Bedrock、Vertex AI 或 Microsoft Foundry 上不可用，请使用客户端处理程序：`new AnthropicClient { Handlers = [new BetaRefusalFallbackHandler { Fallbacks = [new(Model.ClaudeOpus4_8)] }] }`（命名空间 `Anthropic.Helpers`），通过 `BetaFallbackState.Create()` 管理会话级状态，使用 `using (fallbackState.Use()) { ... }` 限定作用域。完整语义（计费、粘性路由、流式）和可运行示例：`shared/model-migration.md` → Migrating to Claude Fable 5 → `refusal` stop reason，以及 C# SDK 仓库的 `examples/`（通过 `shared/live-sources.md` 使用 WebFetch）。

---

## Managed Agents（Beta）

C# SDK 通过 `client.Beta.Agents`、`client.Beta.Sessions`、`client.Beta.Environments` 及相关命名空间支持 Managed Agents。参见 `shared/managed-agents-overview.md` 了解架构，`curl/managed-agents.md` 了解传输层参考。
