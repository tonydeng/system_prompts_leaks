# Claude API — Go

> **注意：** Go SDK 支持 Claude API 和通过 `BetaToolRunner` 实现的 beta 工具使用。Agent SDK 尚未提供 Go 版本。

## 安装

```bash
go get github.com/anthropics/anthropic-sdk-go
```

## 客户端初始化

```go
import (
    "github.com/anthropics/anthropic-sdk-go"
    "github.com/anthropics/anthropic-sdk-go/option"
)

// 默认（使用 ANTHROPIC_API_KEY 环境变量）
client := anthropic.NewClient()

// 显式 API 密钥
client := anthropic.NewClient(
    option.WithAPIKey("your-api-key"),
)
```

---

## 模型常量

Go SDK 提供类型化的模型常量：`anthropic.ModelClaudeFable5`、`anthropic.ModelClaudeOpus4_8`、`anthropic.ModelClaudeOpus4_7`、`anthropic.ModelClaudeSonnet4_6`、`anthropic.ModelClaudeHaiku4_5_20251001`。除非用户另有指定，请使用 `ModelClaudeOpus4_8`；如果用户要求使用 Fable 或最强大的模型，请使用 `anthropic.ModelClaudeFable5`（完整解析表参见 `shared/models.md`）。

---

## 基本消息请求

```go
response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
    Model:     anthropic.ModelClaudeOpus4_8,
    MaxTokens: 16000,
    Messages: []anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("What is the capital of France?")),
    },
})
if err != nil {
    log.Fatal(err)
}
for _, block := range response.Content {
    switch variant := block.AsAny().(type) {
    case anthropic.TextBlock:
        fmt.Println(variant.Text)
    }
}
```

---

## 思考（Thinking）

通过在 `MessageNewParams` 中设置 `Thinking` 来启用 Claude 的内部推理。响应将在最终 `TextBlock` 之前包含 `ThinkingBlock` 内容。

**自适应思考是 Claude 4.6+ 模型的推荐模式。** Claude 会动态决定何时思考以及思考多少。结合 `effort` 参数进行成本-质量控制。

源自 `anthropic-sdk-go/message.go`（`ThinkingConfigParamUnion`、`ThinkingConfigAdaptiveParam`）。

```go
// 没有 ThinkingConfigParamOfAdaptive 辅助函数——直接构造联合体
// 字面量结构体并取变体的地址。
adaptive := anthropic.ThinkingConfigAdaptiveParam{}
params := anthropic.MessageNewParams{
    Model:     anthropic.ModelClaudeSonnet4_6,
    MaxTokens: 16000,
    Thinking:  anthropic.ThinkingConfigParamUnion{OfAdaptive: &adaptive},
    Messages: []anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("How many r's in strawberry?")),
    },
}

resp, err := client.Messages.New(context.Background(), params)
if err != nil {
    log.Fatal(err)
}

// ThinkingBlock(s) 在 content 中位于 TextBlock 之前
for _, block := range resp.Content {
    switch b := block.AsAny().(type) {
    case anthropic.ThinkingBlock:
        fmt.Println("[thinking]", b.Thinking)
    case anthropic.TextBlock:
        fmt.Println(b.Text)
    }
}
```

> **Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考（如上）。`ThinkingConfigParamOfEnabled(budgetTokens)` 在 Fable 5、Opus 4.8 和 4.7 上已移除（发送会返回 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。
> **旧模型：** 使用 `anthropic.ThinkingConfigParamOfEnabled(N)`（预算必须 < `MaxTokens`，最小 1024）。

禁用方式：`anthropic.ThinkingConfigParamUnion{OfDisabled: &anthropic.ThinkingConfigDisabledParam{}}`。

---

## 提示词缓存

`System` 是 `[]TextBlockParam`；在最后一个块上设置 `CacheControl` 以将工具和系统提示一起缓存。关于放置模式和静默失效审计清单，参见 `shared/prompt-caching.md`。

```go
System: []anthropic.TextBlockParam{{
    Text:         longSystemPrompt,
    CacheControl: anthropic.NewCacheControlEphemeralParam(), // 默认 5 分钟 TTL
}},
```

1 小时 TTL：`anthropic.CacheControlEphemeralParam{TTL: anthropic.CacheControlEphemeralTTLTTL1h}`。`MessageNewParams` 上还有一个顶层 `CacheControl`，会自动放置在最后一个可缓存的块上。

通过 `resp.Usage.CacheCreationInputTokens` / `resp.Usage.CacheReadInputTokens` 验证命中。

---

## 停止详情

当 `StopReason` 为 `anthropic.StopReasonRefusal` 时，响应包含结构化的 `StopDetails`：

```go
if resp.StopReason == anthropic.StopReasonRefusal {
    fmt.Println("Category:", resp.StopDetails.Category)     // 如 "cyber"、"bio"、"reasoning_extraction"、"frontier_llm" 或 "" — 完整集合见文档
    fmt.Println("Explanation:", resp.StopDetails.Explanation)
}
```

**拒绝回退（Claude Fable 5）——默认选择启用。** 回退是可选的：不启用时，被拒绝的请求直接停止。在 `claude-fable-5` 代码中，默认包含 `Fallbacks: []anthropic.BetaFallbackParam{{Model: "claude-opus-4-8"}}` 以及 `client.Beta.Messages.New` 上的 `anthropic.AnthropicBetaServerSideFallback2026_06_01` beta——策略拒绝会在同一次调用中由回退模型重新服务。在 Amazon Bedrock、Vertex AI 或 Microsoft Foundry 上不可用——在这些平台上注册客户端中间件：`option.WithMiddleware(betafallback.BetaRefusalFallbackMiddleware(...))`（来自 `lib/betafallback`），通过 `betafallback.WithBetaFallbackState(&betafallback.BetaFallbackState{})` 维护每会话状态。完整语义（计费、粘性路由、流式传输）和可运行示例：`shared/model-migration.md` → 迁移到 Claude Fable 5 → `refusal` 停止原因，以及 Go SDK 仓库的 `examples/`（通过 `shared/live-sources.md` 中的 WebFetch 获取）。

---

## PDF / 文档输入

`NewDocumentBlock` 泛型辅助函数接受任何源类型。`MediaType`/`Type` 会自动设置。

```go
b64 := base64.StdEncoding.EncodeToString(pdfBytes)

msg := anthropic.NewUserMessage(
    anthropic.NewDocumentBlock(anthropic.Base64PDFSourceParam{Data: b64}),
    anthropic.NewTextBlock("Summarize this document"),
)
```

其他源：`URLPDFSourceParam{URL: "https://..."}`、`PlainTextSourceParam{Data: "..."}`。

---

## 上下文编辑 / 压缩（Beta）

使用 `Beta.Messages.New` 并在 `BetaMessageNewParams` 上设置 `ContextManagement`。没有 `NewBetaAssistantMessage`——使用 `.ToParam()` 进行往返。

```go
params := anthropic.BetaMessageNewParams{
    Model:     anthropic.ModelClaudeOpus4_8,  // 也支持：ModelClaudeSonnet4_6
    MaxTokens: 16000,
    Betas:     []anthropic.AnthropicBeta{"compact-2026-01-12"},
    ContextManagement: anthropic.BetaContextManagementConfigParam{
        Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
            {OfCompact20260112: &anthropic.BetaCompact20260112EditParam{}},
        },
    },
    Messages: []anthropic.BetaMessageParam{ /* ... */ },
}

resp, err := client.Beta.Messages.New(ctx, params)
if err != nil {
    log.Fatal(err)
}

// 往返：通过 .ToParam() 将响应追加到历史
params.Messages = append(params.Messages, resp.ToParam())

// 从响应中读取压缩块
for _, block := range resp.Content {
    if c, ok := block.AsAny().(anthropic.BetaCompactionBlock); ok {
        fmt.Println("compaction summary:", c.Content)
    }
}
```

其他编辑类型：`BetaClearToolUses20250919EditParam`、`BetaClearThinking20251015EditParam`——这些需要 `Betas: []anthropic.AnthropicBeta{"context-management-2025-06-27"}`，而非 `compact-2026-01-12`。
