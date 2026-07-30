> **说明**：本文件为英文原文（`tool-use.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 工具使用 — PHP

概念概述（工具定义、工具选择、技巧）请参见 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用

### 工具运行器（Beta）

**Beta：** PHP SDK 通过 `$client->beta->messages->toolRunner()` 提供工具运行器。使用 `BetaRunnableTool` 定义工具，它是一个定义数组加上一个 `run` 闭包：

```php
use Anthropic\Lib\Tools\BetaRunnableTool;

$weatherTool = new BetaRunnableTool(
    definition: [
        'name' => 'get_weather',
        'description' => 'Get the current weather for a location.',
        'inputSchema' => [
            'type' => 'object',
            'properties' => [
                'location' => ['type' => 'string', 'description' => 'City and state'],
            ],
            'required' => ['location'],
        ],
    ],
    run: function (array $input): string {
        return "The weather in {$input['location']} is sunny and 72°F.";
    },
);

$runner = $client->beta->messages->toolRunner(
    maxTokens: 16000,
    messages: [['role' => 'user', 'content' => 'What is the weather in Paris?']],
    model: 'claude-opus-4-8',
    tools: [$weatherTool],
);

foreach ($runner as $message) {
    foreach ($message->content as $block) {
        if ($block->type === 'text') {
            echo $block->text;
        }
    }
}
```

### 手动循环

工具以数组形式传入。**SDK 使用 camelCase 键**（`inputSchema`、`toolUseID`、`stopReason`），并自动映射为 API 传输时的 snake_case，自 v0.5.0 起。循环模式请参见 [共享工具使用概念](../../shared/tool-use-concepts.md)。

```php
use Anthropic\Messages\ToolUseBlock;

$tools = [
    [
        'name' => 'get_weather',
        'description' => 'Get the current weather in a given location',
        'inputSchema' => [  // camelCase, not input_schema
            'type' => 'object',
            'properties' => [
                'location' => ['type' => 'string', 'description' => 'City and state'],
            ],
            'required' => ['location'],
        ],
    ],
];

$messages = [['role' => 'user', 'content' => 'What is the weather in SF?']];

$response = $client->messages->create(
    model: 'claude-opus-4-8',
    maxTokens: 16000,
    tools: $tools,
    messages: $messages,
);

while ($response->stopReason === 'tool_use') {  // camelCase property
    $toolResults = [];
    foreach ($response->content as $block) {
        if ($block instanceof ToolUseBlock) {
            // $block->name  : string               — tool name to dispatch on
            // $block->input : array<string,mixed>  — parsed JSON input
            // $block->id    : string               — pass back as toolUseID
            $result = executeYourTool($block->name, $block->input);
            $toolResults[] = [
                'type' => 'tool_result',
                'toolUseID' => $block->id,  // camelCase, not tool_use_id
                'content' => $result,
            ];
        }
    }

    // Append assistant turn + user turn with tool results
    $messages[] = ['role' => 'assistant', 'content' => $response->content];
    $messages[] = ['role' => 'user', 'content' => $toolResults];

    $response = $client->messages->create(
        model: 'claude-opus-4-8',
        maxTokens: 16000,
        tools: $tools,
        messages: $messages,
    );
}

// Final text response
foreach ($response->content as $block) {
    if ($block->type === 'text') {
        echo $block->text;
    }
}
```

`$block->type === 'tool_use'` 也可以使用；`instanceof ToolUseBlock` 可为 PHPStan 提供类型收窄。


---

## 结构化输出

### 使用 StructuredOutputModel（推荐）

定义一个实现 `StructuredOutputModel` 的 PHP 类，将其作为 `outputConfig` 传入：

```php
use Anthropic\Lib\Contracts\StructuredOutputModel;
use Anthropic\Lib\Concerns\StructuredOutputModelTrait;
use Anthropic\Lib\Attributes\Constrained;

class Person implements StructuredOutputModel
{
    use StructuredOutputModelTrait;

    #[Constrained(description: 'Full name')]
    public string $name;

    public int $age;

    public ?string $email = null;  // nullable = optional field
}

$message = $client->messages->create(
    model: 'claude-opus-4-8',
    maxTokens: 16000,
    messages: [['role' => 'user', 'content' => 'Generate a profile for Alice, age 30']],
    outputConfig: ['format' => Person::class],
);

$person = $message->parsedOutput();  // Person instance
echo $person->name;
```

类型从 PHP 类型提示推断。使用 `#[Constrained(description: '...')]` 添加描述。可空属性（`?string`）会成为可选字段。

### 原始 Schema

```php
$message = $client->messages->create(
    model: 'claude-opus-4-8',
    maxTokens: 16000,
    messages: [['role' => 'user', 'content' => 'Extract: John (john@co.com), Enterprise plan']],
    outputConfig: [
        'format' => [
            'type' => 'json_schema',
            'schema' => [
                'type' => 'object',
                'properties' => [
                    'name' => ['type' => 'string'],
                    'email' => ['type' => 'string'],
                    'plan' => ['type' => 'string'],
                ],
                'required' => ['name', 'email', 'plan'],
                'additionalProperties' => false,
            ],
        ],
    ],
);

// First text block contains valid JSON
foreach ($message->content as $block) {
    if ($block->type === 'text') {
        $data = json_decode($block->text, true);
        break;
    }
}
```

---

## Beta 功能与 Anthropic 定义的工具

**`betas:` 不是 `$client->messages->create()` 的参数**，它只存在于 beta 命名空间中。用于需要显式 opt-in 请求头的功能：

```php
use Anthropic\Beta\Messages\BetaRequestMCPServerURLDefinition;

$response = $client->beta->messages->create(
    model: 'claude-opus-4-8',
    maxTokens: 16000,
    mcpServers: [
        BetaRequestMCPServerURLDefinition::with(
            name: 'my-server',
            url: 'https://example.com/mcp',
        ),
    ],
    betas: ['mcp-client-2025-11-20'],  // only valid on ->beta->messages
    messages: [['role' => 'user', 'content' => 'Use the MCP tools']],
);
```

### 任务预算

```php
$response = $client->beta->messages->create(
    model: 'claude-opus-4-8',
    maxTokens: 16000,
    outputConfig: ['taskBudget' => ['type' => 'tokens', 'total' => 64000]],
    tools: [...],
    messages: [...],
    betas: ['task-budgets-2026-03-13'],
);
```

### 缓存诊断

在下一次请求时传入前一个响应的 `id`；打印响应中的 `diagnostics` 对象：

```php
$r2 = $client->beta->messages->create(
    model: 'claude-opus-4-8', maxTokens: 1024,
    diagnostics: ['previousMessageId' => $r1->id],
    betas: ['cache-diagnosis-2026-04-07'],
    messages: [...],
);
```

**Anthropic 定义的工具**（bash、web_search、text_editor、code_execution）已 GA，在两条路径上均可使用。其中 web_search 和 code_execution 为服务端执行；bash 和 text_editor 为客户端执行（你在本地处理 `tool_use`）。非 beta 版本对应 `Anthropic\Messages\ToolBash20250124` / `WebSearchTool20260209` / `ToolTextEditor20250728` / `CodeExecutionTool20260120`，beta 版本对应 `Anthropic\Beta\Messages\BetaToolBash20250124` / `BetaWebSearchTool20260209` / `BetaToolTextEditor20250728` / `BetaCodeExecutionTool20260120`。这些工具不需要 `betas:` 请求头。

### 工具搜索（非 beta，服务端）

```php
tools: [
    ['type' => 'tool_search_tool_regex_20251119', 'name' => 'tool_search_tool_regex'],
    ['name' => 'get_weather', 'description' => '...', 'inputSchema' => [...], 'deferLoading' => true],
    // ... other user tools with 'deferLoading' => true
],
```

### 记忆工具（非 beta，客户端执行）

声明 `['type' => 'memory_20250818', 'name' => 'memory']`。通过在固定的 `/memories` 目录下读写文件来处理 `tool_use`。**验证模型提供的每个路径**：解析为其规范形式并确认它仍在记忆目录内；拒绝路径遍历（`..`、符号链接），参见 `shared/tool-use-concepts.md` § 客户端工具。

---
