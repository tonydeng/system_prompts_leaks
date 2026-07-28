# Claude API — PHP

> **注意：** PHP SDK 是 Anthropic 官方 PHP SDK。可通过 `$client->beta->messages->toolRunner()` 使用 beta 工具运行器。通过 `StructuredOutputModel` 类支持结构化输出辅助功能。Agent SDK 不可用。支持 Bedrock、Vertex AI 和 Foundry 客户端。

## 安装

```bash
composer require "anthropic-ai/sdk"
```

## 客户端初始化

```php
use Anthropic\Client;

// 使用环境变量中的 API Key
$client = new Client(apiKey: getenv("ANTHROPIC_API_KEY"));
```

### Amazon Bedrock

```php
use Anthropic\Bedrock\MantleClient;

// Messages-API Bedrock 端点。从环境变量读取 AWS 凭证。
$client = new MantleClient(awsRegion: 'us-east-1');
```

Bedrock 上的模型 ID 使用 `anthropic.` 前缀，例如 `model: 'anthropic.claude-opus-4-8'`。

### Google Vertex AI

```php
use Anthropic\Vertex;

// 构造函数为 private。参数是 `location`，不是 `region`。
$client = Vertex\client::fromEnvironment(
    location: 'us-east5',
    projectId: 'my-project-id',
);
```

### Anthropic Foundry

```php
use Anthropic\Foundry;

// 构造函数为 private。baseUrl 或 resource 为必填。
$client = Foundry\Client::withCredentials(
    apiKey: getenv('ANTHROPIC_FOUNDRY_API_KEY'),
    baseUrl: 'https://<resource>.services.ai.azure.com/anthropic/v1',
);
```

---

## 基本消息请求

```php
$message = $client->messages->create(
    model: 'claude-opus-4-8',
    maxTokens: 16000,
    messages: [
        ['role' => 'user', 'content' => 'What is the capital of France?'],
    ],
);

// content 是多态块数组（TextBlock、ToolUseBlock、ThinkingBlock）。
// 不检查块类型直接访问 content[0] 的 ->text，当第一个块不是
// TextBlock 时会抛出异常（例如启用扩展思考后 ThinkingBlock 排在前面）。
// 务必进行类型守卫：
foreach ($message->content as $block) {
    if ($block->type === 'text') {
        echo $block->text;
    }
}
```

如果只需要第一个文本块：

```php
foreach ($message->content as $block) {
    if ($block->type === 'text') {
        echo $block->text;
        break;
    }
}
```

---

## 扩展思考

**自适应思考是 Claude 4.6+ 模型的推荐模式。** Claude 动态决定何时思考以及思考多少。

```php
use Anthropic\Messages\ThinkingBlock;

$message = $client->messages->create(
    model: 'claude-opus-4-8',
    maxTokens: 16000,
    thinking: ['type' => 'adaptive', 'display' => 'summarized'], // display 需手动开启：Fable 5 / Mythos 5 / Opus 4.8 / 4.7 上默认省略（思考文本为空）
    messages: [
        ['role' => 'user', 'content' => 'Solve: 27 * 453'],
    ],
);

// ThinkingBlock(s) 在 content 中排在 TextBlock 之前
foreach ($message->content as $block) {
    if ($block instanceof ThinkingBlock) {
        echo "Thinking:\n{$block->thinking}\n\n";
        // $block->signature 是不透明字符串，在多轮对话中传回思考块时
        // 需原样保留
    } elseif ($block->type === 'text') {
        echo "Answer: {$block->text}\n";
    }
}
```

> **Fable 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考（见上文）。`['type' => 'enabled', 'budgetTokens' => N]` 在 Fable 5、Opus 4.8 和 4.7 上已移除（发送会返回 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。
> **旧模型：** 使用 `thinking: ['type' => 'enabled', 'budgetTokens' => N]`（budget 必须小于 `maxTokens`，最小 1024）。

`$block->type === 'thinking'` 也可用于检查；`instanceof` 可为 PHPStan 提供类型收窄。

---

## 提示词缓存

`system:` 接收文本块数组；在最后一个块上设置 `cacheControl`。数组语法（驼峰键名）是惯用写法。关于放置模式和静默失效因子审计清单，参见 `shared/prompt-caching.md`。

```php
$message = $client->messages->create(
    model: 'claude-opus-4-8',
    maxTokens: 16000,
    system: [
        ['type' => 'text', 'text' => $longSystemPrompt, 'cacheControl' => ['type' => 'ephemeral']],
    ],
    messages: [['role' => 'user', 'content' => 'Summarize the key points']],
);
```

1 小时 TTL：`'cacheControl' => ['type' => 'ephemeral', 'ttl' => '1h']`。`messages->create(...)` 上还有一个顶层 `cacheControl:` 参数，会自动放置在最后一个可缓存块上。

通过 `$message->usage->cacheCreationInputTokens` / `$message->usage->cacheReadInputTokens` 验证命中。

---

## 停止详情

当 `stopReason` 为 `'refusal'` 时，响应包含结构化 `stopDetails`：

```php
if ($message->stopReason === 'refusal' && $message->stopDetails !== null) {
    echo "Category: " . $message->stopDetails->category . "\n";     // 例如 "cyber"、"bio"、"reasoning_extraction"、"frontier_llm" 或 null — 完整列表见文档
    echo "Explanation: " . $message->stopDetails->explanation . "\n";
}
```

**拒绝回退（Claude Fable 5）— 默认开启。** 回退是可选的：不启用时被拒绝的请求直接停止。新的 `claude-fable-5` 代码应默认包含服务端 `fallbacks` 参数（beta header `server-side-fallback-2026-06-01`，回退模型 `claude-opus-4-8`，在 beta messages 调用上）。确切的 PHP 绑定（以及不支持服务端回退的提供商的客户端中间件）不在本文档中，通过 `shared/live-sources.md` WebFetch PHP SDK 仓库的 `examples/`；完整语义见 `shared/model-migration.md` → 迁移到 Claude Fable 5 → `refusal` 停止原因。

---

## 错误类型

`APIStatusException` 暴露 `->type` 属性用于程序化错误分类：

```php
try {
    $client->messages->create(...);
} catch (\Anthropic\Core\Exceptions\APIStatusException $e) {
    echo $e->type?->value;  // "rate_limit_error"、"overloaded_error" 等
}
```
