> **说明**：本文件为英文原文（`tool-use.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 工具使用 — C#

概念概述（工具定义、工具选择、技巧）参见 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用

### 定义工具

使用 `Tool`（不是 `ToolParam`）加 `InputSchema` 记录。`InputSchema.Type` 由构造函数自动设置为 `"object"`，不要手动设置。`ToolUnion` 有从 `Tool` 的隐式转换，通过集合表达式 `[...]` 触发。

```csharp
using System.Text.Json;
using Anthropic.Models.Messages;

var parameters = new MessageCreateParams
{
    Model = Model.ClaudeSonnet4_6,
    MaxTokens = 16000,
    Tools = [
        new Tool {
            Name = "get_weather",
            Description = "Get the current weather in a given location",
            InputSchema = new() {
                Properties = new Dictionary<string, JsonElement> {
                    ["location"] = JsonSerializer.SerializeToElement(
                        new { type = "string", description = "City name" }),
                },
                Required = ["location"],
            },
        },
    ],
    Messages = [new() { Role = Role.User, Content = "Weather in Paris?" }],
};
```

源自 `anthropic-sdk-csharp/src/Anthropic/Models/Messages/Tool.cs` 和 `ToolUnion.cs:799`（隐式转换）。

循环模式参见 [共享工具使用概念](../../shared/tool-use-concepts.md)。
### 将响应内容转换为后续 assistant 消息

在 assistant 回合中回传 Claude 的响应时，**没有 `.ToParam()` 辅助方法**，需手动将每个 `ContentBlock` 变体重构为其 `*Param` 对应类型。不要使用 `new ContentBlockParam(block.Json)`：它能编译和序列化，但 `.Value` 保持 `null`，导致 `TryPick*`/`Validate()` 失败（退化为 JSON 透传，而非类型化路径）。

```csharp
using Anthropic.Models.Messages;

Message response = await client.Messages.Create(parameters);

// 没有 .ToParam() — 按变体重构。每个 *Param 类型到 ContentBlockParam 的
// 隐式转换意味着无需显式包装。
List<ContentBlockParam> assistantContent = [];
List<ContentBlockParam> toolResults = [];
foreach (ContentBlock block in response.Content)
{
    if (block.TryPickText(out TextBlock? text))
    {
        assistantContent.Add(new TextBlockParam { Text = text.Text });
    }
    else if (block.TryPickThinking(out ThinkingBlock? thinking))
    {
        // Signature 必须保留 — API 会拒绝篡改
        assistantContent.Add(new ThinkingBlockParam
        {
            Thinking = thinking.Thinking,
            Signature = thinking.Signature,
        });
    }
    else if (block.TryPickRedactedThinking(out RedactedThinkingBlock? redacted))
    {
        assistantContent.Add(new RedactedThinkingBlockParam { Data = redacted.Data });
    }
    else if (block.TryPickToolUse(out ToolUseBlock? toolUse))
    {
        // ToolUseBlock 有必需的 Caller；ToolUseBlockParam.Caller 是可选的 — 不要复制
        assistantContent.Add(new ToolUseBlockParam
        {
            ID = toolUse.ID,
            Name = toolUse.Name,
            Input = toolUse.Input,
        });
        // 执行工具；每个 tool_use 块收集一个结果 — API
        // 在任何 tool_use ID 缺少匹配 tool_result 时拒绝后续请求。
        string result = ExecuteYourTool(toolUse.Name, toolUse.Input);
        toolResults.Add(new ToolResultBlockParam
        {
            ToolUseID = toolUse.ID,
            Content = result,
        });
    }
}

// 后续请求：之前的消息 + assistant 回显 + user tool_result(s)
List<MessageParam> followUpMessages =
[
    .. parameters.Messages,
    new() { Role = Role.Assistant, Content = assistantContent },
    new() { Role = Role.User, Content = toolResults },
];
```

`ToolResultBlockParam` 没有元组构造函数，使用对象初始化器。`Content` 是 string 或 list 联合类型；纯 `string` 会隐式转换。

---

## 结构化输出

```csharp
OutputConfig = new OutputConfig {
    Format = new JsonOutputFormat {
        Schema = new Dictionary<string, JsonElement> {
            ["type"] = JsonSerializer.SerializeToElement("object"),
            ["properties"] = JsonSerializer.SerializeToElement(
                new { name = new { type = "string" } }),
            ["required"] = JsonSerializer.SerializeToElement(new[] { "name" }),
        },
    },
},
```

`JsonOutputFormat.Type` 由构造函数自动设置为 `"json_schema"`。`Schema` 为必填。

---

## Anthropic 定义的工具

Web 搜索、bash、文本编辑器和代码执行是 Anthropic 定义的工具，具有内置 schema。Web 搜索和代码执行由服务端执行；bash 和文本编辑器由客户端执行（你在本地处理 `tool_use`，参见 `shared/tool-use-concepts.md`）。类型名带有版本后缀；构造函数自动设置 `name`/`type`。**用 `new ToolUnion(...)` 显式包装每一个。**

```csharp
Tools = [
    new ToolUnion(new WebSearchTool20260209()),
    new ToolUnion(new ToolBash20250124()),
    new ToolUnion(new ToolTextEditor20250728()),
    new ToolUnion(new CodeExecutionTool20260120()),
],
```

还可使用：`new ToolUnion(new WebFetchTool20260209())`、`new ToolUnion(new MemoryTool20250818())`。`WebSearchTool20260209` 可选参数：`AllowedDomains`、`BlockedDomains`、`MaxUses`、`UserLocation`。

---

## 工具运行器（Beta）

C# SDK 提供 `BetaToolRunner` 用于自动工具执行循环。用原始 JSON schema 定义工具，运行器处理 API 调用、工具执行、结果反馈循环。

```csharp
using Anthropic.Models.Beta.Messages;

// 按上方工具使用部分所示定义工具并创建参数，
// 但使用 beta 命名空间类型（BetaToolUnion 等）
var runner = client.Beta.Messages.ToolRunner(betaParams);

await foreach (BetaMessage message in runner)
{
    foreach (var block in message.Content)
    {
        if (block.TryPickText(out var text))
        {
            Console.WriteLine(text.Text);
        }
    }
}
```

---
