> **说明**：本文件为英文原文（`tool-use.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 工具使用 — Go

概念概述（工具定义、工具选择、技巧）请参见 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用

### 工具运行器（Beta — 推荐）

**Beta：** Go SDK 通过 `toolrunner` 包提供 `BetaToolRunner`，用于自动工具使用循环。

```go
import (
    "context"
    "fmt"
    "log"

    "github.com/anthropics/anthropic-sdk-go"
    "github.com/anthropics/anthropic-sdk-go/toolrunner"
)

// Define tool input with jsonschema tags for automatic schema generation
type GetWeatherInput struct {
    City string `json:"city" jsonschema:"required,description=The city name"`
}

// Create a tool with automatic schema generation from struct tags
weatherTool, err := toolrunner.NewBetaToolFromJSONSchema(
    "get_weather",
    "Get current weather for a city",
    func(ctx context.Context, input GetWeatherInput) (anthropic.BetaToolResultBlockParamContentUnion, error) {
        return anthropic.BetaToolResultBlockParamContentUnion{
            OfText: &anthropic.BetaTextBlockParam{
                Text: fmt.Sprintf("The weather in %s is sunny, 72°F", input.City),
            },
        }, nil
    },
)
if err != nil {
    log.Fatal(err)
}

// Create a tool runner that handles the conversation loop automatically
runner := client.Beta.Messages.NewToolRunner(
    []anthropic.BetaTool{weatherTool},
    anthropic.BetaToolRunnerParams{
        BetaMessageNewParams: anthropic.BetaMessageNewParams{
            Model:     anthropic.ModelClaudeOpus4_8,
            MaxTokens: 16000,
            Messages: []anthropic.BetaMessageParam{
                anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("What's the weather in Paris?")),
            },
        },
        MaxIterations: 5,
    },
)

// Run until Claude produces a final response
message, err := runner.RunToCompletion(context.Background())
if err != nil {
    log.Fatal(err)
}

// RunToCompletion returns *BetaMessage; content is []BetaContentBlockUnion.
// Narrow via AsAny() switch — note the Beta-namespace types (BetaTextBlock,
// not TextBlock):
for _, block := range message.Content {
    switch block := block.AsAny().(type) {
    case anthropic.BetaTextBlock:
        fmt.Println(block.Text)
    }
}
```

**Go 工具运行器的关键特性：**

- 通过 `jsonschema` 标签从 Go struct 自动生成 schema
- `RunToCompletion()` 用于简单的一次性使用
- `All()` 迭代器用于处理对话中的每条消息
- `NextMessage()` 用于逐步迭代
- 通过 `NewToolRunnerStreaming()` 和 `AllStreaming()` 提供流式变体

### 手动循环

优先使用上面的工具运行器。对于拦截、验证、日志记录或人工审批，在工具的 run 函数内门控，或用 `NextMessage()`/`All()` 逐步推进运行器并检查每条消息（运行器的公共 `Params` 字段让你调整下一次请求），手动循环不是必需的。仅当你需要运行器未暴露的控制时才降级为手动循环：用 `ToolParam` 定义工具、检查 `StopReason`、自行执行工具并回传 `tool_result` 块。

源自 `anthropic-sdk-go/examples/tools/main.go`。

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    "github.com/anthropics/anthropic-sdk-go"
)

func main() {
    client := anthropic.NewClient()

    // 1. Define tools. ToolParam.InputSchema uses a map, no struct tags needed.
    addTool := anthropic.ToolParam{
        Name:        "add",
        Description: anthropic.String("Add two integers"),
        InputSchema: anthropic.ToolInputSchemaParam{
            Properties: map[string]any{
                "a": map[string]any{"type": "integer"},
                "b": map[string]any{"type": "integer"},
            },
        },
    }
    // ToolParam must be wrapped in ToolUnionParam for the Tools slice
    tools := []anthropic.ToolUnionParam{{OfTool: &addTool}}

    messages := []anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("What is 2 + 3?")),
    }

    for {
        resp, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
            Model:     anthropic.ModelClaudeSonnet4_6,
            MaxTokens: 16000,
            Messages:  messages,
            Tools:     tools,
        })
        if err != nil {
            log.Fatal(err)
        }

        // 2. Append the assistant response to history BEFORE processing tool calls.
        //    resp.ToParam() converts Message → MessageParam in one call.
        messages = append(messages, resp.ToParam())

        // 3. Walk content blocks. ContentBlockUnion is a flattened struct;
        //    use block.AsAny().(type) to switch on the actual variant.
        toolResults := []anthropic.ContentBlockParamUnion{}
        for _, block := range resp.Content {
            switch variant := block.AsAny().(type) {
            case anthropic.TextBlock:
                fmt.Println(variant.Text)
            case anthropic.ToolUseBlock:
                // 4. Parse the tool input. Use variant.JSON.Input.Raw() to get the
                //    raw JSON — block.Input is json.RawMessage, not the parsed value.
                var in struct {
                    A int `json:"a"`
                    B int `json:"b"`
                }
                if err := json.Unmarshal([]byte(variant.JSON.Input.Raw()), &in); err != nil {
                    log.Fatal(err)
                }
                result := fmt.Sprintf("%d", in.A+in.B)
                // 5. NewToolResultBlock(toolUseID, content, isError) builds the
                //    ContentBlockParamUnion for you. block.ID is the tool_use_id.
                toolResults = append(toolResults,
                    anthropic.NewToolResultBlock(block.ID, result, false))
            }
        }

        // 6. Exit when Claude stops asking for tools
        if resp.StopReason != anthropic.StopReasonToolUse {
            break
        }

        // 7. Tool results go in a user message (variadic: all results in one turn)
        messages = append(messages, anthropic.NewUserMessage(toolResults...))
    }
}
```

**关键 API 界面：**

| 符号 | 用途 |
|---|---|
| `resp.ToParam()` | 将 `Message` 响应转换为 `MessageParam` 用于历史记录 |
| `block.AsAny().(type)` | 对 `ContentBlockUnion` 变体进行类型切换 |
| `variant.JSON.Input.Raw()` | 工具输入的原始 JSON 字符串（用于 `json.Unmarshal`） |
| `anthropic.NewToolResultBlock(id, content, isError)` | 构建 `tool_result` 块 |
| `anthropic.NewUserMessage(blocks...)` | 将工具结果包装为用户轮次 |
| `anthropic.StopReasonToolUse` | 用于检查循环终止的 `StopReason` 常量 |
| `anthropic.ToolUnionParam{OfTool: &t}` | 将 `ToolParam` 包装在联合中用于 `Tools:` |

---

## Anthropic 定义的工具

带版本后缀的 struct 名称加上 `Param` 后缀。`Name`/`Type` 是 `constant.*` 类型，零值正确序列化，因此 `{}` 即可。用匹配的 `Of*` 字段包装在 `ToolUnionParam` 中。Web 搜索和代码执行为服务端执行；bash 和文本编辑器为客户端执行（你在本地处理 `tool_use`，参见 `shared/tool-use-concepts.md`）。

```go
Tools: []anthropic.ToolUnionParam{
    {OfWebSearchTool20260209: &anthropic.WebSearchTool20260209Param{}},
    {OfBashTool20250124: &anthropic.ToolBash20250124Param{}},
    {OfTextEditor20250728: &anthropic.ToolTextEditor20250728Param{}},
    {OfCodeExecutionTool20260120: &anthropic.CodeExecutionTool20260120Param{}},
},
```

另有：`WebFetchTool20260209Param`、`ToolSearchToolBm25_20251119Param`、`ToolSearchToolRegex20251119Param`。对于 advisor 和记忆工具，在 `client.Beta.Messages.New` 的 beta 命名空间中使用 `BetaAdvisorTool20260301Param` / `BetaMemoryTool20250818Param`。

### Advisor 工具（beta）

服务端，无需 tool_result 往返。advisor 模型必须 ≥ 执行器（顶层）模型；无效配对返回 400。

```go
response, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
    Model:     anthropic.ModelClaudeSonnet4_6,
    MaxTokens: 4096,
    Tools: []anthropic.BetaToolUnionParam{
        {OfAdvisorTool20260301: &anthropic.BetaAdvisorTool20260301Param{
            Model: anthropic.ModelClaudeOpus4_8,
        }},
    },
    Messages: []anthropic.BetaMessageParam{ /* ... */ },
    Betas:    []anthropic.AnthropicBeta{anthropic.AnthropicBetaAdvisorTool2026_03_01},
})
```

---
