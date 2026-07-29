# Claude API — Java

> **注意：** Java SDK 支持 Claude API 和带有注解类的 beta 工具使用。Agent SDK 尚未提供 Java 版本。

## 包参考

类型按包组织。如果你需要的类没有在下面的示例中出现，先通过此表定位——不要因为获取 SDK 源码而阻塞在网络请求上。

| `import` 前缀 | 包含内容 |
|---|---|
| `com.anthropic.client` / `com.anthropic.client.okhttp` | `AnthropicClient`、`AnthropicOkHttpClient` |
| `com.anthropic.models.messages` | 非 beta 请求/响应类型——`MessageCreateParams`、`Model`、`Message`、`TextBlockParam`、`ContentBlockParam`、`ToolUseBlockParam`、`ToolResultBlockParam`、`CacheControlEphemeral`、`Tool*`（如 `ToolBash20250124`、`ToolTextEditor20250728`）、`StopReason`、`StructuredMessage*` |
| `com.anthropic.models.messages.batches` | Batch API——`BatchResultsParams`、`MessageBatchIndividualResponse` |
| `com.anthropic.models.beta` | `AnthropicBeta`（beta 标志常量） |
| `com.anthropic.models.beta.messages` | beta 端点类型——`MessageCreateParams`、`BetaMessage`、`BetaStopReason`、`BetaContextManagementConfig`、`BetaMcpToolset`、`BetaRequestMcpServerUrlDefinition`、`BetaTool*` |
| `com.anthropic.core` | `JsonValue`、`JsonField`、`JsonSchemaLocalValidation`、`com.anthropic.core.http.StreamResponse` |
| `com.anthropic.errors` | 类型化异常——`AnthropicServiceException`、`RateLimitException`、`NotFoundException` 等（见 `shared/error-codes.md`） |

`client.messages()` 使用 `com.anthropic.models.messages.*`；`client.beta().messages()` 使用 `com.anthropic.models.beta.messages.*`。两个包都定义了 `MessageCreateParams`——导入与你调用的客户端路径匹配的那个。

### 各功能的关键类型

从此表编写代码，而非通过 `javap`/jar 检查。Endpoint 列告诉你使用 `client.messages()` 还是 `client.beta().messages()`。

| 功能 | 端点 | 关键 Java 类型 / builder 调用 |
|---|---|---|
| 用户配置 | beta | `client.beta().userProfiles().create(...)` / `.retrieve(id)` / `.list()`。将返回的 profile id 传递给 beta `MessageCreateParams`。需要 beta header——查看 SDK 的 beta-headers 参考获取当前标志。 |
| Agent Skills | beta | `BetaContainerParams`、`BetaSkillParams`、`BetaCodeExecutionTool20250825`。`.addBeta("code-execution-2025-08-25").addBeta("skills-2025-10-02")`。通过 `client.beta().files().download(fileId)` 下载输出。 |
| 缓存诊断 | beta | `BetaDiagnosticsParam`、`BetaCacheControlEphemeral` |
| 上下文编辑 | beta | `.contextManagement(BetaContextManagementConfig.builder()…)`。编辑策略是 `BetaClearToolUses20250919Edit`（或 `BetaClearThinking20251015Edit`）；其触发器是单独构建的 `BetaInputTokensTrigger`，传递给 edit 的 builder——edit builder 上没有直接的 `.inputTokensTrigger(N)` 快捷方式。用 `javap` 查看 edit 和 trigger 类的确切 setter 名称。 |
| 内存工具 | non-beta | `.addTool(MemoryTool20250818.builder().build())`，来自 `com.anthropic.models.messages` |
| 程序化工具调用 | non-beta | `CodeExecutionTool20260120`、`Tool`、`ContentBlockParam` |
| 严格工具使用 | non-beta | `Tool`、`Tool.InputSchema` |
| 任务预算 | beta | `.outputConfig(BetaOutputConfig.builder().taskBudget(BetaTokenTaskBudget.builder()...))` |
| 工具搜索 | non-beta | `.addTool(ToolSearchToolRegex20251119.builder()...)`，来自 `com.anthropic.models.messages` |
| 网络搜索 | non-beta | `WebSearchTool20260209`，来自 `com.anthropic.models.messages`——最新变体，支持动态过滤（Opus 4.8/4.7/4.6 + Sonnet 4.6）。对于较旧模型或 Vertex，使用 `WebSearchTool20250305` |

### 发现类型和成员名称

如果你需要的类或 builder 方法不在上面的表中，`jar tf <anthropic-java-core jar> | grep -i <term>` 或 `javap -classpath <jar> com.anthropic.models.…` 足够快来定位名称。**不要编译并运行单独的反射程序**来枚举成员——第一次构建足够慢，在许多环境中会被放至后台，使你陷入轮询循环。用你找到的名称编写脚本，让编译器错误（`cannot find symbol`）指向任何错误的成员。

## 安装

Maven：

```xml
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-java</artifactId>
    <version>2.34.0</version>
</dependency>
```

Gradle：

```groovy
implementation("com.anthropic:anthropic-java:2.34.0")
```

## 客户端初始化

```java
import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;

// 默认（从环境变量读取 ANTHROPIC_API_KEY）
AnthropicClient client = AnthropicOkHttpClient.fromEnv();

// 显式 API key
AnthropicClient client = AnthropicOkHttpClient.builder()
    .apiKey("your-api-key")
    .build();
```

---

## 基本消息请求

```java
import com.anthropic.models.messages.MessageCreateParams;
import com.anthropic.models.messages.Message;
import com.anthropic.models.messages.Model;

MessageCreateParams params = MessageCreateParams.builder()
    .model(Model.CLAUDE_OPUS_4_8)
    .maxTokens(16000L)
    .addUserMessage("What is the capital of France?")
    .build();

Message response = client.messages().create(params);
response.content().stream()
    .flatMap(block -> block.text().stream())
    .forEach(textBlock -> System.out.println(textBlock.text()));
```

---

## Thinking

**自适应思考是 Claude 4.6+ 模型的推荐模式。** Claude 动态决定何时思考以及思考多少。builder 有直接的 `.thinking(ThinkingConfigAdaptive)` 重载——无需手动 union 包装。

> **Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考（见下文）。`ThinkingConfigEnabled.builder().budgetTokens(N)` 在 Fable 5、Opus 4.8 和 4.7 上已移除（发送则返回 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。
> **较旧模型：** 使用 `.thinking(ThinkingConfigEnabled.builder().budgetTokens(N).build())`（budget 必须 < `maxTokens`，最小 1024）。

```java
import com.anthropic.models.messages.ContentBlock;
import com.anthropic.models.messages.MessageCreateParams;
import com.anthropic.models.messages.Model;
import com.anthropic.models.messages.ThinkingConfigAdaptive;

MessageCreateParams params = MessageCreateParams.builder()
    .model(Model.CLAUDE_SONNET_4_6)
    .maxTokens(16000L)
    .thinking(ThinkingConfigAdaptive.builder().build())
    .addUserMessage("Solve this step by step: 27 * 453")
    .build();

for (ContentBlock block : client.messages().create(params).content()) {
    block.thinking().ifPresent(t -> System.out.println("[thinking] " + t.thinking()));
    block.text().ifPresent(t -> System.out.println(t.text()));
}
```

`ContentBlock` 窄化：`.thinking()` / `.text()` 返回 `Optional<T>`——使用 `.ifPresent(...)` 或 `.stream().flatMap(...)`。替代方式：`isThinking()` / `asThinking()` 布尔+解包对（在错误变体上抛出异常）。

---

## Effort 参数

Effort 嵌套在 `OutputConfig` 内部——`MessageCreateParams.Builder` 上没有直接的 `.effort()`。

```java
import com.anthropic.models.messages.OutputConfig;

.outputConfig(OutputConfig.builder()
    .effort(OutputConfig.Effort.HIGH)  // 或 LOW, MEDIUM, MAX
    .build())
```

与 `Thinking = ThinkingConfigAdaptive` 结合用于成本-质量控制。

---

## 提示缓存

系统消息作为带有 `CacheControlEphemeral` 的 `TextBlockParam` 列表。使用 `.systemOfTextBlockParams(...)`——普通的 `.system(String)` 重载无法携带缓存控制。关于放置模式和静默失效审计清单，见 `shared/prompt-caching.md`。

```java
import com.anthropic.models.messages.TextBlockParam;
import com.anthropic.models.messages.CacheControlEphemeral;

.systemOfTextBlockParams(List.of(
    TextBlockParam.builder()
        .text(longSystemPrompt)
        .cacheControl(CacheControlEphemeral.builder()
            .ttl(CacheControlEphemeral.Ttl.TTL_1H)  // 可选；也有 TTL_5M
            .build())
        .build()))
```

`MessageCreateParams.Builder` 和 `Tool.builder()` 上还有顶层的 `.cacheControl(CacheControlEphemeral)`。

通过 `response.usage().cacheCreationInputTokens()` / `response.usage().cacheReadInputTokens()` 验证命中。

---

## Token 计数

```java
import com.anthropic.models.messages.MessageCountTokensParams;

long tokens = client.messages().countTokens(
    MessageCountTokensParams.builder()
        .model(Model.CLAUDE_SONNET_4_6)
        .addUserMessage("Hello")
        .build()
).inputTokens();
```

---

## PDF / 文档输入

`DocumentBlockParam` builder 有 source 快捷方式。用 `ContentBlockParam.ofDocument()` 包装并通过 `.addUserMessageOfBlockParams()` 传递。

```java
import com.anthropic.models.messages.DocumentBlockParam;
import com.anthropic.models.messages.ContentBlockParam;
import com.anthropic.models.messages.TextBlockParam;

DocumentBlockParam doc = DocumentBlockParam.builder()
    .source(Base64PdfSource.builder().data(base64String).build())
    // 或 .source(UrlPdfSource.builder().url("https://...").build())
    .title("My Document")        // 可选
    .build();
```

对于 **Files API** 文档引用，使用 beta 路径和 beta 类型——见 `files-api.md`：`BetaRequestDocumentBlock.builder().source(BetaFileDocumentSource.builder().fileId(id).build())`。

```java
.addUserMessageOfBlockParams(List.of(
    ContentBlockParam.ofDocument(doc),
    ContentBlockParam.ofText(TextBlockParam.builder().text("Summarize this").build())))
```

---

## 停止详情

当 `stopReason()` 为 `"refusal"` 时，响应包含结构化的 `stopDetails()`：

```java
response.stopDetails().ifPresent(details -> {
    System.out.println("Category: " + details.category());
    System.out.println("Explanation: " + details.explanation());
});
```

**拒绝回退（Claude Fable 5）——默认选择启用。** 回退是可选的：没有它们，被拒绝的请求直接停止。新的 `claude-fable-5` 代码默认应包含服务端 `fallbacks` 参数（beta header `server-side-fallback-2026-06-01`，回退模型 `claude-opus-4-8`，在 beta messages 调用上）。确切的 Java builder 方法（以及针对无服务端支持提供商的客户端中间件）此处未记录——从 `shared/live-sources.md` WebFetch Java SDK 仓库的 `examples/`；完整语义见 `shared/model-migration.md` → 迁移到 Claude Fable 5 → `refusal` 停止原因。

---

## 错误类型

`AnthropicServiceException` 暴露 `.errorType()` 返回 `Optional<ErrorType>`，用于程序化错误分类：

```java
try {
    client.messages().create(params);
} catch (AnthropicServiceException e) {
    e.errorType().ifPresent(type ->
        System.out.println("Error type: " + type)  // RATE_LIMIT_ERROR, OVERLOADED_ERROR 等
    );
}
```

---
