# Managed Agents — PHP

> **未展示的绑定：** 本 README 涵盖了 PHP 最常见的 managed-agents 流程。如果你需要的类、方法、命名空间、字段或行为未在此展示，请通过 WebFetch 获取 `shared/live-sources.md` 中的 PHP SDK 仓库**或相关文档页面**，而不是猜测。不要从 cURL 形状或其他语言的 SDK 中推断。

> **代理是持久的——创建一次，通过 ID 引用。** 存储 `$client->beta->agents->create` 返回的代理 ID，并将其传递给每个后续的 `->sessions->create`；不要在请求路径中调用 `agents->create`。**推荐：** 将代理和环境定义为版本控制的 YAML，使用 `ant` CLI 应用——参见 `shared/anthropic-cli.md`（其实时文档 URL 在 `shared/live-sources.md` 中）。CLI 拥有控制面（创建/更新）；你的代码拥有数据面（使用存储的 ID 创建会话）。以下示例展示了必须在代码中创建的情况；在生产环境中，create 调用应属于设置阶段，而非请求路径。

## 安装

```bash
composer require "anthropic-ai/sdk" "guzzlehttp/guzzle:^7"
```

## 客户端初始化

```php
use Anthropic\Client;

// 默认（使用 ANTHROPIC_API_KEY 环境变量）
$client = new Client();

// 显式 API 密钥
$client = new Client(apiKey: 'your-api-key');
```

---

## 创建环境

```php
$environment = $client->beta->environments->create(
    name: 'my-dev-env',
    config: ['type' => 'cloud', 'networking' => ['type' => 'unrestricted']],
);
echo "Environment ID: {$environment->id}\n"; // env_...
```

---

## 创建代理（必需的第一步）

> ⚠️ **没有内联代理配置。** `model`/`system`/`tools` 位于代理对象上，而非会话上。始终以 `$client->beta->agents->create()` 开始——会话接受 `agent: $agent->id` 或类型化的 `BetaManagedAgentsAgentParams::with(type: 'agent', id: $agent->id, version: $agent->version)`。

### 最小化示例

```php
use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;

// 1. 创建代理（可复用，版本化）
$agent = $client->beta->agents->create(
    name: 'Coding Assistant',
    model: 'claude-opus-4-8',
    system: 'You are a helpful coding assistant.',
    tools: [
        BetaManagedAgentsAgentToolset20260401Params::with(
            type: 'agent_toolset_20260401',
        ),
    ],
);

// 2. 启动会话
$session = $client->beta->sessions->create(
    agent: ['type' => 'agent', 'id' => $agent->id, 'version' => $agent->version],
    environmentID: $environment->id,
    title: 'Quickstart session',
);
echo "Session ID: {$session->id}\n";
echo "Trace: https://platform.claude.com/workspaces/default/sessions/{$session->id}\n"; // 如果 API 密钥不在 Default 工作区中，将 'default' 替换为你的工作区 ID
```

### 更新代理

更新会创建新版本；代理对象按版本不可变。

```php
$updatedAgent = $client->beta->agents->update(
    $agent->id,
    version: $agent->version,
    system: 'You are a helpful coding agent. Always write tests.',
);
echo "New version: {$updatedAgent->version}\n";

// 列出所有版本
foreach ($client->beta->agents->versions->list($agent->id)->pagingEachItem() as $version) {
    echo "Version {$version->version}: {$version->updatedAt->format(DateTimeInterface::ATOM)}\n";
}

// 归档代理
$archived = $client->beta->agents->archive($agent->id);
echo "Archived at: {$archived->archivedAt->format(DateTimeInterface::ATOM)}\n";
```

---

## 发送用户消息

```php
$client->beta->sessions->events->send(
    $session->id,
    events: [
        [
            'type' => 'user.message',
            'content' => [['type' => 'text', 'text' => 'Review the auth module']],
        ],
    ],
);
```

> 💡 **流优先：** 在发送消息*之前*（或同时）打开流。流仅传递在其打开后发生的事件——先发送后开流意味着早期事件以缓冲批次到达。参见[引导模式](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式传输事件（SSE）

> ℹ️ **流式传输器：** PHP 默认的缓冲 PSR-18 客户端对于开放式会话事件流永远不会返回。为 `streamStream()` 调用使用流式 Guzzle 传输器——其他调用保持默认客户端。

```php
$streamingClient = new GuzzleHttp\Client(['stream' => true]);

// 先打开流，然后发送用户消息
$stream = $client->beta->sessions->events->streamStream(
    $session->id,
    requestOptions: ['transporter' => $streamingClient],
);
$client->beta->sessions->events->send(
    $session->id,
    events: [
        [
            'type' => 'user.message',
            'content' => [['type' => 'text', 'text' => 'Summarize the repo README']],
        ],
    ],
);

foreach ($stream as $event) {
    match ($event->type) {
        'agent.message' => array_walk(
            $event->content,
            static fn($block) => $block->type === 'text' ? print($block->text) : null,
        ),
        'agent.tool_use' => print("\n[Using tool: {$event->name}]\n"),
        'session.error' => printf("\n[Error: %s]", $event->error?->message ?? 'unknown'),
        default => null,
    };
    if ($event->type === 'session.status_idle' || $event->type === 'session.error') {
        break;
    }
}
$stream->close();
```

### 重连和跟踪

在会话中途重连时，先列出过去的事件进行去重，然后跟踪实时事件：

```php
$stream = $client->beta->sessions->events->streamStream(
    $session->id,
    requestOptions: ['transporter' => $streamingClient],
);

// 流已打开并正在缓冲。在跟踪实时事件之前列出历史。
$seenEventIds = [];
foreach ($client->beta->sessions->events->list($session->id)->pagingEachItem() as $event) {
    $seenEventIds[$event->id] = true;
}

// 跟踪实时事件，跳过已见过的
foreach ($stream as $event) {
    if (isset($seenEventIds[$event->id])) {
        continue;
    }
    $seenEventIds[$event->id] = true;
    match ($event->type) {
        'agent.message' => array_walk(
            $event->content,
            static fn($block) => $block->type === 'text' ? print($block->text) : null,
        ),
        default => null,
    };
    if ($event->type === 'session.status_idle') {
        break;
    }
}
$stream->close();
```

---

## 提供自定义工具结果

> ℹ️ PHP managed-agents 中 `user.custom_tool_result` 的绑定尚未在本技能或应用源示例中记录。请参阅 `shared/managed-agents-events.md` 了解传输格式，并参阅 `anthropic-ai/sdk` PHP 仓库获取对应的参数。

---

## 轮询事件

```php
foreach ($client->beta->sessions->events->list($session->id)->pagingEachItem() as $event) {
    echo "{$event->type}: {$event->id}\n";
}
```

---

## 上传文件

> ℹ️ **PHP 文件上传：** PHP SDK 的 beta managed-agents 文件上传绑定未在应用源示例中展示；规范的 PHP 示例使用原始 cURL 调用 `POST /v1/files`。如果你的代码库偏好使用 SDK，在编写代码之前先 WebFetch `anthropic-ai/sdk` PHP 仓库获取最新绑定。

```php
use Anthropic\Beta\Sessions\BetaManagedAgentsFileResourceParams;

// 原始 cURL 上传（来自应用源的规范示例）
$csvPath = 'data.csv';
$ch = curl_init('https://api.anthropic.com/v1/files');
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST => true,
    CURLOPT_HTTPHEADER => [
        'x-api-key: ' . getenv('ANTHROPIC_API_KEY'),
        'anthropic-version: 2023-06-01',
        'anthropic-beta: files-api-2025-04-14',
    ],
    CURLOPT_POSTFIELDS => ['file' => new CURLFile($csvPath, 'text/csv', 'data.csv')],
]);
$file = json_decode(curl_exec($ch));
echo "File ID: {$file->id}\n";

// 在会话中挂载
$session = $client->beta->sessions->create(
    agent: $agent->id,
    environmentID: $environment->id,
    resources: [
        BetaManagedAgentsFileResourceParams::with(
            type: 'file',
            fileID: $file->id,
            mountPath: '/workspace/data.csv',
        ),
    ],
);
```

### 在现有会话上添加和管理资源

```php
// 向打开的会话附加额外文件
$resource = $client->beta->sessions->resources->add(
    $session->id,
    type: 'file',
    fileID: $file->id,
);
echo "{$resource->id}\n"; // "sesrsc_01ABC..."

// 列出会话上的资源
$listed = $client->beta->sessions->resources->list($session->id);
foreach ($listed->data as $entry) {
    echo "{$entry->id} {$entry->type}\n";
}

// 分离资源
$client->beta->sessions->resources->delete($resource->id, sessionID: $session->id);
```

---

## 列出和下载会话文件

```php
$files = $client->beta->files->list(
    scopeID: 'sesn_abc123',
    betas: ['managed-agents-2026-04-01'],
);
$content = $client->beta->files->download($files->data[0]->id);
file_put_contents('output.txt', $content);
```

---

## 会话管理

```php
// 列出环境
$environments = $client->beta->environments->list();

// 检索特定环境
$env = $client->beta->environments->retrieve($environment->id);

// 归档环境（只读，现有会话继续运行）
$client->beta->environments->archive($environment->id);

// 删除环境（仅当没有会话引用它时）
$client->beta->environments->delete($environment->id);

// 删除会话
$client->beta->sessions->delete($session->id);
```

---

## MCP 服务器集成

```php
use Anthropic\Beta\Agents\BetaManagedAgentsAgentToolset20260401Params;
use Anthropic\Beta\Agents\BetaManagedAgentsMCPToolsetParams;
use Anthropic\Beta\Agents\BetaManagedAgentsURLMCPServerParams;
use Anthropic\Beta\Sessions\BetaManagedAgentsAgentParams;

// 代理声明 MCP 服务器（此处无认证——认证放在 vault 中）
$agent = $client->beta->agents->create(
    name: 'GitHub Assistant',
    model: 'claude-opus-4-8',
    mcpServers: [
        BetaManagedAgentsURLMCPServerParams::with(
            type: 'url',
            name: 'github',
            url: 'https://api.githubcopilot.com/mcp/',
        ),
    ],
    tools: [
        BetaManagedAgentsAgentToolset20260401Params::with(type: 'agent_toolset_20260401'),
        BetaManagedAgentsMCPToolsetParams::with(
            type: 'mcp_toolset',
            mcpServerName: 'github',
        ),
    ],
);

// 会话附加包含这些 MCP 服务器 URL 凭证的 vault
$session = $client->beta->sessions->create(
    agent: BetaManagedAgentsAgentParams::with(
        type: 'agent',
        id: $agent->id,
        version: $agent->version,
    ),
    environmentID: $environment->id,
    vaultIDs: [$vault->id],
);
```

参见 `shared/managed-agents-tools.md` §Vaults 了解创建 vault 和添加凭证的方法。

---

## Vaults

```php
// 创建 vault
$vault = $client->beta->vaults->create(
    displayName: 'Alice',
    metadata: ['external_user_id' => 'usr_abc123'],
);
echo $vault->id . "\n"; // "vlt_01ABC..."

// 添加 OAuth 凭证
$credential = $client->beta->vaults->credentials->create(
    vaultID: $vault->id,
    displayName: "Alice's Slack",
    auth: [
        'type' => 'mcp_oauth',
        'mcp_server_url' => 'https://mcp.slack.com/mcp',
        'access_token' => 'xoxp-...',
        'expires_at' => '2026-04-15T00:00:00Z',
        'refresh' => [
            'token_endpoint' => 'https://slack.com/api/oauth.v2.access',
            'client_id' => '1234567890.0987654321',
            'scope' => 'channels:read chat:write',
            'refresh_token' => 'xoxe-1-...',
            'token_endpoint_auth' => [
                'type' => 'client_secret_post',
                'client_secret' => 'abc123...',
            ],
        ],
    ],
);

// 轮换凭证（例如令牌刷新后）
$client->beta->vaults->credentials->update(
    $credential->id,
    vaultID: $vault->id,
    auth: [
        'type' => 'mcp_oauth',
        'access_token' => 'xoxp-new-...',
        'expires_at' => '2026-05-15T00:00:00Z',
        'refresh' => ['refresh_token' => 'xoxe-1-new-...'],
    ],
);

// 归档 vault
$client->beta->vaults->archive($vault->id);
```

---

## GitHub 仓库集成

将 GitHub 仓库挂载为会话资源（vault 持有 GitHub MCP 凭证）：

```php
$session = $client->beta->sessions->create(
    agent: $agent->id,
    environmentID: $environment->id,
    vaultIDs: [$vault->id],
    resources: [
        [
            'type' => 'github_repository',
            'url' => 'https://github.com/org/repo',
            'mount_path' => '/workspace/repo',
            'authorization_token' => 'ghp_your_github_token',
        ],
    ],
);
```

同一会话上的多个仓库：

```php
$resources = [
    [
        'type' => 'github_repository',
        'url' => 'https://github.com/org/frontend',
        'mount_path' => '/workspace/frontend',
        'authorization_token' => 'ghp_your_github_token',
    ],
    [
        'type' => 'github_repository',
        'url' => 'https://github.com/org/backend',
        'mount_path' => '/workspace/backend',
        'authorization_token' => 'ghp_your_github_token',
    ],
];
```

轮换仓库的授权令牌：

```php
$listed = $client->beta->sessions->resources->list($session->id);
$repoResourceId = $listed->data[0]->id;

$client->beta->sessions->resources->update(
    $repoResourceId,
    sessionID: $session->id,
    authorizationToken: 'ghp_your_new_github_token',
);
```
