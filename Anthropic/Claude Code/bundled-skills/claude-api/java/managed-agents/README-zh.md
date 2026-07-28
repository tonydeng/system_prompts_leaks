# Managed Agents — Java

> **此处未展示的绑定：** 本 README 涵盖 Java 最常见的 managed-agents 流程。如果你需要一个未展示的类、方法、命名空间、字段或行为，请通过 WebFetch 访问 `shared/live-sources.md` 中的 Java SDK 仓库**或相关文档页面**，而非猜测。不要从 cURL 形状或其他语言的 SDK 进行推断。

> **智能体是持久的——创建一次，通过 ID 引用。** 存储 `client.beta().agents().create` 返回的 agent ID，并将其传递给每个后续的 `client.beta().sessions().create`；不要在请求路径中调用 `agents().create`。**推荐做法：** 将智能体和环境定义为版本控制的 YAML，使用 `ant` CLI 应用——参见 `shared/anthropic-cli.md`（其实时文档 URL 在 `shared/live-sources.md` 中）。CLI 拥有控制面（创建/更新）；你的代码拥有数据面（使用存储的 ID 进行会话）。下面的示例展示了需要编程方式配置时的代码内创建；在生产环境中，create 调用应属于初始化设置，而非请求路径。

## 安装

```xml
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-java</artifactId>
</dependency>
```

## 客户端初始化

```java
import com.anthropic.client.okhttp.AnthropicOkHttpClient;

// 默认（使用 ANTHROPIC_API_KEY 环境变量）
var client = AnthropicOkHttpClient.fromEnv();
```

---

## 创建环境

```java
import com.anthropic.models.beta.environments.BetaCloudConfigParams;
import com.anthropic.models.beta.environments.BetaUnrestrictedNetwork;
import com.anthropic.models.beta.environments.EnvironmentCreateParams;

var environment = client.beta().environments().create(EnvironmentCreateParams.builder()
    .name("my-dev-env")
    .config(BetaCloudConfigParams.builder()
        .networking(BetaUnrestrictedNetwork.builder().build())
        .build())
    .build());
System.out.println("Environment ID: " + environment.id()); // env_...
```

---

## 创建智能体（必需的第一步）

> ⚠️ **没有内联智能体配置。** 模型、系统和工具存在于智能体对象上，而非会话上。始终以 `client.beta().agents().create()` 开始——会话接受 `.agent(agent.id())` 或类型化的 `BetaManagedAgentsAgentParams.builder()...build()`。

### 最小示例

```java
import com.anthropic.models.beta.agents.AgentCreateParams;
import com.anthropic.models.beta.agents.BetaManagedAgentsAgentToolset20260401Params;
import com.anthropic.models.beta.sessions.BetaManagedAgentsAgentParams;
import com.anthropic.models.beta.sessions.SessionCreateParams;

// 1. 创建智能体（可复用、有版本）
var agent = client.beta().agents().create(AgentCreateParams.builder()
    .name("Coding Assistant")
    .model("claude-opus-4-8")
    .system("You are a helpful coding assistant.")
    .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
        .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
        .build())
    .build());

// 2. 启动会话
var session = client.beta().sessions().create(SessionCreateParams.builder()
    .agent(BetaManagedAgentsAgentParams.builder()
        .type(BetaManagedAgentsAgentParams.Type.AGENT)
        .id(agent.id())
        .version(agent.version())
        .build())
    .environmentId(environment.id())
    .title("Quickstart session")
    .build());
System.out.println("Session ID: " + session.id());
System.out.println("Trace: https://platform.claude.com/workspaces/default/sessions/" + session.id()); // 如果 API key 不在 Default workspace 中，将 'default' 替换为你的 workspace ID
```

### 更新智能体

更新会创建新版本；智能体对象在每个版本上是不可变的。

```java
import com.anthropic.models.beta.agents.AgentUpdateParams;

var updatedAgent = client.beta().agents().update(agent.id(), AgentUpdateParams.builder()
    .version(agent.version())
    .system("You are a helpful coding agent. Always write tests.")
    .build());
System.out.println("New version: " + updatedAgent.version());

// 列出所有版本
for (var version : client.beta().agents().versions().list(agent.id()).autoPager()) {
    System.out.println("Version " + version.version() + ": " + version.updatedAt());
}

// 归档智能体
var archived = client.beta().agents().archive(agent.id());
System.out.println("Archived at: " + archived.archivedAt().orElseThrow());
```

---

## 发送用户消息

```java
import com.anthropic.models.beta.sessions.events.BetaManagedAgentsUserMessageEventParams;
import com.anthropic.models.beta.sessions.events.EventSendParams;

client.beta().sessions().events().send(session.id(), EventSendParams.builder()
    .addEvent(BetaManagedAgentsUserMessageEventParams.builder()
        .type(BetaManagedAgentsUserMessageEventParams.Type.USER_MESSAGE)
        .addTextContent("Review the auth module")
        .build())
    .build());
```

> 💡 **流优先：** 在发送消息*之前*（或同时）打开流。流仅传递在它打开之后发生的事件——先发送后打开流意味着早期事件会以缓冲批次到达。参见 [Steering Patterns](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式事件（SSE）

```java
import com.anthropic.models.beta.sessions.events.StreamEvents;

// 先打开流，再发送用户消息
try (var stream = client.beta().sessions().events().streamStreaming(session.id())) {
    client.beta().sessions().events().send(session.id(), EventSendParams.builder()
        .addEvent(BetaManagedAgentsUserMessageEventParams.builder()
            .type(BetaManagedAgentsUserMessageEventParams.Type.USER_MESSAGE)
            .addTextContent("Summarize the repo README")
            .build())
        .build());

    for (var event : (Iterable<StreamEvents>) stream.stream()::iterator) {
        if (event.isAgentMessage()) {
            event.asAgentMessage().content().forEach(block -> System.out.print(block.text()));
        } else if (event.isAgentToolUse()) {
            System.out.println("\n[Using tool: " + event.asAgentToolUse().name() + "]");
        } else if (event.isSessionStatusIdle()) {
            break;
        } else if (event.isSessionError()) {
            System.out.println("\n[Error]");
            break;
        }
    }
}
```

### 重连和追踪

在会话中途重连时，先列出历史事件进行去重，然后追踪实时事件。跨变体的 `id` 字段从原始 `_json()` 值中读取：

```java
import com.anthropic.core.JsonValue;
import java.util.HashSet;
import java.util.Map;
import java.util.Optional;

try (var stream = client.beta().sessions().events().streamStreaming(session.id())) {
    // 流已打开并正在缓冲。在追踪实时事件前列出历史。
    var seenEventIds = new HashSet<String>();
    for (var past : client.beta().sessions().events().list(session.id()).autoPager()) {
        Optional<Map<String, JsonValue>> obj = past._json().orElseThrow().asObject();
        seenEventIds.add(obj.orElseThrow().get("id").asStringOrThrow());
    }

    // 追踪实时事件，跳过已见过的事件
    for (var event : (Iterable<StreamEvents>) stream.stream()::iterator) {
        Optional<Map<String, JsonValue>> obj = event._json().orElseThrow().asObject();
        if (!seenEventIds.add(obj.orElseThrow().get("id").asStringOrThrow())) continue;
        if (event.isAgentMessage()) {
            event.asAgentMessage().content().forEach(block -> System.out.print(block.text()));
        } else if (event.isSessionStatusIdle()) {
            break;
        }
    }
}
```

---

## 提供自定义工具结果

> ℹ️ Java managed-agents 绑定中 `user.custom_tool_result` 的用法尚未在本 skill 或 apps 源码示例中记录。请参阅 `shared/managed-agents-events.md` 了解线格式，参阅 `anthropic-java` 仓库了解对应的参数类型。

---

## 轮询事件

```java
for (var event : client.beta().sessions().events().list(session.id()).autoPager()) {
    System.out.println(event.type() + ": " + event);
}
```

---

## 上传文件

```java
import com.anthropic.models.beta.files.FileUploadParams;
import com.anthropic.models.beta.sessions.BetaManagedAgentsFileResourceParams;
import java.nio.file.Path;

var dataCsv = Path.of("data.csv");

var file = client.beta().files().upload(FileUploadParams.builder()
    .file(dataCsv)
    .build());
System.out.println("File ID: " + file.id());

// 在会话中挂载
var session = client.beta().sessions().create(SessionCreateParams.builder()
    .agent(agent.id())
    .environmentId(environment.id())
    .addResource(BetaManagedAgentsFileResourceParams.builder()
        .type(BetaManagedAgentsFileResourceParams.Type.FILE)
        .fileId(file.id())
        .mountPath("/workspace/data.csv")
        .build())
    .build());
```

### 在现有会话上添加和管理资源

```java
import com.anthropic.models.beta.sessions.resources.ResourceAddParams;
import com.anthropic.models.beta.sessions.resources.ResourceDeleteParams;

// 向打开的会话附加额外文件
var resource = client.beta().sessions().resources().add(session.id(), ResourceAddParams.builder()
    .betaManagedAgentsFileResourceParams(BetaManagedAgentsFileResourceParams.builder()
        .type(BetaManagedAgentsFileResourceParams.Type.FILE)
        .fileId(file.id())
        .build())
    .build());
System.out.println(resource.id()); // "sesrsc_01ABC..."

// 列出会话上的资源——条目是判别联合类型
var listed = client.beta().sessions().resources().list(session.id());
for (var entry : listed.data()) {
    if (entry.isFile()) {
        var fileResource = entry.asFile();
        System.out.println(fileResource.id() + " " + fileResource.type());
    } else if (entry.isGitHubRepository()) {
        var repoResource = entry.asGitHubRepository();
        System.out.println(repoResource.id() + " " + repoResource.type());
    }
}

// 分离资源
client.beta().sessions().resources().delete(resource.id(), ResourceDeleteParams.builder()
    .sessionId(session.id())
    .build());
```

---

## 列出和下载会话文件

> ℹ️ 列出和下载智能体在会话期间写入的文件尚未在本 skill 或 apps 源码示例中为 Java 记录。参见 `shared/managed-agents-events.md` 和 `anthropic-java` 仓库了解文件列表/下载绑定。

---

## 会话管理

```java
// 列出环境
var environments = client.beta().environments().list();

// 检索特定环境
var env = client.beta().environments().retrieve(environment.id());

// 归档环境（只读，现有会话继续运行）
client.beta().environments().archive(environment.id());

// 删除环境（仅当没有会话引用它时）
client.beta().environments().delete(environment.id());

// 删除会话
client.beta().sessions().delete(session.id());
```

---

## MCP 服务器集成

```java
import com.anthropic.models.beta.agents.BetaManagedAgentsMcpToolsetParams;
import com.anthropic.models.beta.agents.BetaManagedAgentsUrlMcpServerParams;

// 智能体声明 MCP 服务器（此处无认证——认证放在 vault 中）
var agent = client.beta().agents().create(AgentCreateParams.builder()
    .name("GitHub Assistant")
    .model("claude-opus-4-8")
    .addMcpServer(BetaManagedAgentsUrlMcpServerParams.builder()
        .type(BetaManagedAgentsUrlMcpServerParams.Type.URL)
        .name("github")
        .url("https://api.githubcopilot.com/mcp/")
        .build())
    .addTool(BetaManagedAgentsAgentToolset20260401Params.builder()
        .type(BetaManagedAgentsAgentToolset20260401Params.Type.AGENT_TOOLSET_20260401)
        .build())
    .addTool(BetaManagedAgentsMcpToolsetParams.builder()
        .type(BetaManagedAgentsMcpToolsetParams.Type.MCP_TOOLSET)
        .mcpServerName("github")
        .build())
    .build());

// 会话附加包含这些 MCP 服务器 URL 凭据的 vault
var session = client.beta().sessions().create(SessionCreateParams.builder()
    .agent(BetaManagedAgentsAgentParams.builder()
        .type(BetaManagedAgentsAgentParams.Type.AGENT)
        .id(agent.id())
        .version(agent.version())
        .build())
    .environmentId(environment.id())
    .addVaultId(vault.id())
    .build());
```

参见 `shared/managed-agents-tools.md` §Vaults 了解创建 vault 和添加凭据的方法。

---

## Vaults

```java
import com.anthropic.core.JsonValue;
import com.anthropic.models.beta.vaults.VaultCreateParams;
import com.anthropic.models.beta.vaults.credentials.BetaManagedAgentsMcpOAuthCreateParams;
import com.anthropic.models.beta.vaults.credentials.BetaManagedAgentsMcpOAuthRefreshParams;
import com.anthropic.models.beta.vaults.credentials.BetaManagedAgentsMcpOAuthRefreshUpdateParams;
import com.anthropic.models.beta.vaults.credentials.BetaManagedAgentsMcpOAuthUpdateParams;
import com.anthropic.models.beta.vaults.credentials.CredentialCreateParams;
import com.anthropic.models.beta.vaults.credentials.CredentialUpdateParams;
import java.time.OffsetDateTime;

// 创建 vault
var vault = client.beta().vaults().create(VaultCreateParams.builder()
    .displayName("Alice")
    .metadata(VaultCreateParams.Metadata.builder()
        .putAdditionalProperty("external_user_id", JsonValue.from("usr_abc123"))
        .build())
    .build());
System.out.println(vault.id()); // "vlt_01ABC..."

// 添加 OAuth 凭据
var credential = client.beta().vaults().credentials().create(vault.id(),
    CredentialCreateParams.builder()
        .displayName("Alice's Slack")
        .auth(BetaManagedAgentsMcpOAuthCreateParams.builder()
            .type(BetaManagedAgentsMcpOAuthCreateParams.Type.MCP_OAUTH)
            .mcpServerUrl("https://mcp.slack.com/mcp")
            .accessToken("xoxp-...")
            .expiresAt(OffsetDateTime.parse("2026-04-15T00:00:00Z"))
            .refresh(BetaManagedAgentsMcpOAuthRefreshParams.builder()
                .tokenEndpoint("https://slack.com/api/oauth.v2.access")
                .clientId("1234567890.0987654321")
                .scope("channels:read chat:write")
                .refreshToken("xoxe-1-...")
                .clientSecretPostTokenEndpointAuth("abc123...")
                .build())
            .build())
        .build());

// 轮换凭据（例如在 token 刷新后）
client.beta().vaults().credentials().update(credential.id(),
    CredentialUpdateParams.builder()
        .vaultId(vault.id())
        .auth(BetaManagedAgentsMcpOAuthUpdateParams.builder()
            .type(BetaManagedAgentsMcpOAuthUpdateParams.Type.MCP_OAUTH)
            .accessToken("xoxp-new-...")
            .expiresAt(OffsetDateTime.parse("2026-05-15T00:00:00Z"))
            .refresh(BetaManagedAgentsMcpOAuthRefreshUpdateParams.builder()
                .refreshToken("xoxe-1-new-...")
                .build())
            .build())
        .build());

// 归档 vault
client.beta().vaults().archive(vault.id());
```

---

## GitHub 仓库集成

将 GitHub 仓库挂载为会话资源（vault 持有 GitHub MCP 凭据）：

```java
import com.anthropic.models.beta.sessions.BetaManagedAgentsGitHubRepositoryResourceParams;

var session = client.beta().sessions().create(SessionCreateParams.builder()
    .agent(agent.id())
    .environmentId(environment.id())
    .addVaultId(vault.id())
    .addResource(BetaManagedAgentsGitHubRepositoryResourceParams.builder()
        .type(BetaManagedAgentsGitHubRepositoryResourceParams.Type.GITHUB_REPOSITORY)
        .url("https://github.com/org/repo")
        .mountPath("/workspace/repo")
        .authorizationToken("ghp_your_github_token")
        .build())
    .build());
```

同一会话上的多个仓库：

```java
import java.util.List;

var resources = List.of(
    BetaManagedAgentsGitHubRepositoryResourceParams.builder()
        .type(BetaManagedAgentsGitHubRepositoryResourceParams.Type.GITHUB_REPOSITORY)
        .url("https://github.com/org/frontend")
        .mountPath("/workspace/frontend")
        .authorizationToken("ghp_your_github_token")
        .build(),
    BetaManagedAgentsGitHubRepositoryResourceParams.builder()
        .type(BetaManagedAgentsGitHubRepositoryResourceParams.Type.GITHUB_REPOSITORY)
        .url("https://github.com/org/backend")
        .mountPath("/workspace/backend")
        .authorizationToken("ghp_your_github_token")
        .build());
```

轮换仓库的授权 token：

```java
import com.anthropic.models.beta.sessions.resources.ResourceUpdateParams;

var listed = client.beta().sessions().resources().list(session.id());
var repoResourceId = listed.data().get(0).asGitHubRepository().id();

client.beta().sessions().resources().update(repoResourceId, ResourceUpdateParams.builder()
    .sessionId(session.id())
    .authorizationToken("ghp_your_new_github_token")
    .build());
```
