# Managed Agents — Go

> **此处未展示的绑定：** 本 README 涵盖了 Go 最常见的 managed-agents 流程。如果需要此处未展示的类、方法、命名空间、字段或行为，请通过 WebFetch 获取 Go SDK 仓库**或 `shared/live-sources.md` 中相关文档页面**，而非猜测。不要从 cURL 形状或其他语言的 SDK 进行推断。

> **Agent 是持久的——创建一次，通过 ID 引用。** 存储 `agents.New` 返回的 agent ID，并将其传递给后续每个 `sessions.New`；不要在请求路径中调用 `agents.New`。**推荐做法：** 将 agent 和 environment 定义为版本控制的 YAML，通过 `ant` CLI 应用——参见 `shared/anthropic-cli.md`（其实时文档 URL 在 `shared/live-sources.md` 中）。CLI 负责控制面（创建/更新）；你的代码负责数据面（使用存储的 ID 进行会话）。以下示例展示了必须以编程方式配置时的代码内创建；在生产环境中，创建调用应放在初始化阶段，而非请求路径中。

## 安装

```bash
go get github.com/anthropics/anthropic-sdk-go
```

## 客户端初始化

```go
import (
    "context"

    "github.com/anthropics/anthropic-sdk-go"
    "github.com/anthropics/anthropic-sdk-go/option"
)

// 默认（使用 ANTHROPIC_API_KEY 环境变量）
client := anthropic.NewClient()

// 显式 API key
client := anthropic.NewClient(
    option.WithAPIKey("your-api-key"),
)

ctx := context.Background()
```

---

## 创建 Environment

```go
environment, err := client.Beta.Environments.New(ctx, anthropic.BetaEnvironmentNewParams{
    Name: "my-dev-env",
    Config: anthropic.BetaEnvironmentNewParamsConfigUnion{
        OfCloud: &anthropic.BetaCloudConfigParams{
            Networking: anthropic.BetaCloudConfigParamsNetworkingUnion{
                OfUnrestricted: &anthropic.BetaUnrestrictedNetworkParam{},
            },
        },
    },
})
if err != nil {
    panic(err)
}
fmt.Println(environment.ID) // env_...
```

---

## 创建 Agent（必需的第一步）

> ⚠️ **没有内联 agent 配置。** `Model`/`System`/`Tools` 位于 agent 对象上，而非 session 上。始终以 `Beta.Agents.New()` 开始——session 只接受 `Agent: anthropic.BetaSessionNewParamsAgentUnion{OfString: anthropic.String(agent.ID)}`（或当你需要特定版本时使用类型化的 `OfBetaManagedAgentsAgents` 变体）。

### 最小示例

```go
// 1. 创建 agent（可复用、有版本）
agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
    Name: "Coding Assistant",
    Model: anthropic.BetaManagedAgentsModelConfigParams{
        ID:   "claude-opus-4-8",
        Type: anthropic.BetaManagedAgentsModelConfigParamsTypeModelConfig,
    },
    System: anthropic.String("You are a helpful coding assistant."),
    Tools: []anthropic.BetaAgentNewParamsToolUnion{{
        OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
            Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
        },
    }},
})
if err != nil {
    panic(err)
}

// 2. 启动 session
session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
    Agent: anthropic.BetaSessionNewParamsAgentUnion{
        OfBetaManagedAgentsAgents: &anthropic.BetaManagedAgentsAgentParams{
            Type:    anthropic.BetaManagedAgentsAgentParamsTypeAgent,
            ID:      agent.ID,
            Version: anthropic.Int(agent.Version),
        },
    },
    EnvironmentID: environment.ID,
    Title:         anthropic.String("Quickstart session"),
})
if err != nil {
    panic(err)
}
fmt.Printf("Session ID: %s, status: %s\n", session.ID, session.Status)
fmt.Printf("Trace: https://platform.claude.com/workspaces/default/sessions/%s\n", session.ID) // 如果 API key 不在 Default workspace 中，将 'default' 替换为你的 workspace ID
```

### 更新 Agent

更新会创建新版本；agent 对象在每个版本上是不可变的。

```go
updatedAgent, err := client.Beta.Agents.Update(ctx, agent.ID, anthropic.BetaAgentUpdateParams{
    Version: agent.Version,
    System:  anthropic.String("You are a helpful coding agent. Always write tests."),
})
if err != nil {
    panic(err)
}
fmt.Printf("New version: %d\n", updatedAgent.Version)

// 列出所有版本
iter := client.Beta.Agents.Versions.ListAutoPaging(ctx, agent.ID, anthropic.BetaAgentVersionListParams{})
for iter.Next() {
    version := iter.Current()
    fmt.Printf("Version %d: %s\n", version.Version, version.UpdatedAt.Format(time.RFC3339))
}
if err := iter.Err(); err != nil {
    panic(err)
}

// 归档 agent
_, err = client.Beta.Agents.Archive(ctx, agent.ID, anthropic.BetaAgentArchiveParams{})
if err != nil {
    panic(err)
}
```

---

## 发送用户消息

```go
_, err = client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
    Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
        OfUserMessage: &anthropic.BetaManagedAgentsUserMessageEventParams{
            Type: anthropic.BetaManagedAgentsUserMessageEventParamsTypeUserMessage,
            Content: []anthropic.BetaManagedAgentsUserMessageEventParamsContentUnion{{
                OfText: &anthropic.BetaManagedAgentsTextBlockParam{
                    Type: anthropic.BetaManagedAgentsTextBlockTypeText,
                    Text: "Review the auth module",
                },
            }},
        },
    }},
})
if err != nil {
    panic(err)
}
```

> 💡 **流优先：** 在发送消息*之前*（或同时）打开流。流只传递在它打开之后发生的事件——先发送后开流意味着早期事件会作为缓冲批量到达。参见 [引导模式](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式事件（SSE）

```go
// 先打开流，再发送用户消息
stream := client.Beta.Sessions.Events.StreamEvents(ctx, session.ID, anthropic.BetaSessionEventStreamParams{})
defer stream.Close()

if _, err := client.Beta.Sessions.Events.Send(ctx, session.ID, anthropic.BetaSessionEventSendParams{
    Events: []anthropic.BetaManagedAgentsEventParamsUnion{{
        OfUserMessage: &anthropic.BetaManagedAgentsUserMessageEventParams{
            Type: anthropic.BetaManagedAgentsUserMessageEventParamsTypeUserMessage,
            Content: []anthropic.BetaManagedAgentsUserMessageEventParamsContentUnion{{
                OfText: &anthropic.BetaManagedAgentsTextBlockParam{
                    Type: anthropic.BetaManagedAgentsTextBlockTypeText,
                    Text: "Summarize the repo README",
                },
            }},
        },
    }},
}); err != nil {
    panic(err)
}

events:
for stream.Next() {
    switch event := stream.Current().AsAny().(type) {
    case anthropic.BetaManagedAgentsAgentMessageEvent:
        for _, block := range event.Content {
            fmt.Print(block.Text)
        }
    case anthropic.BetaManagedAgentsAgentToolUseEvent:
        fmt.Printf("\n[Using tool: %s]\n", event.Name)
    case anthropic.BetaManagedAgentsSessionStatusIdleEvent:
        break events
    case anthropic.BetaManagedAgentsSessionErrorEvent:
        fmt.Printf("\n[Error: %s]\n", event.Error.Message)
        break events
    }
}
if err := stream.Err(); err != nil {
    panic(err)
}
```

### 重连与尾随

在会话中途重连时，先列出过去的事件进行去重，然后尾随实时事件：

```go
stream := client.Beta.Sessions.Events.StreamEvents(ctx, session.ID, anthropic.BetaSessionEventStreamParams{})
defer stream.Close()

// 流已打开并正在缓冲。在尾随实时事件之前列出历史。
seenEventIDs := map[string]struct{}{}
history := client.Beta.Sessions.Events.ListAutoPaging(ctx, session.ID, anthropic.BetaSessionEventListParams{})
for history.Next() {
    seenEventIDs[history.Current().ID] = struct{}{}
}
if err := history.Err(); err != nil {
    panic(err)
}

// 尾随实时事件，跳过已见过的
tail:
for stream.Next() {
    event := stream.Current()
    if _, seen := seenEventIDs[event.ID]; seen {
        continue
    }
    seenEventIDs[event.ID] = struct{}{}
    switch event := event.AsAny().(type) {
    case anthropic.BetaManagedAgentsAgentMessageEvent:
        for _, block := range event.Content {
            fmt.Print(block.Text)
        }
    case anthropic.BetaManagedAgentsSessionStatusIdleEvent:
        break tail
    }
}
if err := stream.Err(); err != nil {
    panic(err)
}
```

---

## 提供自定义工具结果

> ℹ️ Go managed-agents 绑定中 `user.custom_tool_result` 的用法尚未在本 skill 或应用源码示例中记录。请参阅 `shared/managed-agents-events.md` 了解传输格式，并参阅 `github.com/anthropics/anthropic-sdk-go` 仓库了解对应的 Go params 类型。

---

## 轮询事件

```go
// 自动分页迭代器
iter := client.Beta.Sessions.Events.ListAutoPaging(ctx, session.ID, anthropic.BetaSessionEventListParams{})
for iter.Next() {
    event := iter.Current()
    fmt.Printf("%s: %s\n", event.Type, event.ID)
}
if err := iter.Err(); err != nil {
    panic(err)
}
```

---

## 上传文件

```go
csvFile, err := os.Open("data.csv")
if err != nil {
    panic(err)
}
defer csvFile.Close()

file, err := client.Beta.Files.Upload(ctx, anthropic.BetaFileUploadParams{
    File: csvFile,
})
if err != nil {
    panic(err)
}
fmt.Printf("File ID: %s\n", file.ID)

// 在 session 中挂载
session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
    Agent: anthropic.BetaSessionNewParamsAgentUnion{
        OfString: anthropic.String(agent.ID),
    },
    EnvironmentID: environment.ID,
    Resources: []anthropic.BetaSessionNewParamsResourceUnion{{
        OfFile: &anthropic.BetaManagedAgentsFileResourceParams{
            Type:      anthropic.BetaManagedAgentsFileResourceParamsTypeFile,
            FileID:    file.ID,
            MountPath: anthropic.String("/workspace/data.csv"),
        },
    }},
})
if err != nil {
    panic(err)
}
```

### 在现有 Session 上添加和管理资源

```go
// 向打开的 session 附加额外文件
resource, err := client.Beta.Sessions.Resources.Add(ctx, session.ID, anthropic.BetaSessionResourceAddParams{
    BetaManagedAgentsFileResourceParams: anthropic.BetaManagedAgentsFileResourceParams{
        Type:   anthropic.BetaManagedAgentsFileResourceParamsTypeFile,
        FileID: file.ID,
    },
})
if err != nil {
    panic(err)
}
fmt.Println(resource.ID) // "sesrsc_01ABC..."

// 列出 session 上的资源
listed, err := client.Beta.Sessions.Resources.List(ctx, session.ID, anthropic.BetaSessionResourceListParams{})
if err != nil {
    panic(err)
}
for _, entry := range listed.Data {
    fmt.Println(entry.ID, entry.Type)
}

// 分离资源
if _, err := client.Beta.Sessions.Resources.Delete(ctx, resource.ID, anthropic.BetaSessionResourceDeleteParams{
    SessionID: session.ID,
}); err != nil {
    panic(err)
}
```

---

## 列出和下载 Session 文件

> ℹ️ Go 在本 skill 或应用源码示例中尚未记录列出和下载 agent 在 session 期间写入的文件。请参阅 `shared/managed-agents-events.md` 和 `github.com/anthropics/anthropic-sdk-go` 仓库了解 `Beta.Files.List` 和 `Beta.Files.Download` 的 Go params 类型。

---

## Session 管理

```go
// 列出 environment
environments, err := client.Beta.Environments.List(ctx, anthropic.BetaEnvironmentListParams{})
if err != nil {
    panic(err)
}

// 检索特定 environment
env, err := client.Beta.Environments.Get(ctx, environment.ID, anthropic.BetaEnvironmentGetParams{})
if err != nil {
    panic(err)
}

// 归档 environment（只读，现有 session 继续）
_, err = client.Beta.Environments.Archive(ctx, environment.ID, anthropic.BetaEnvironmentArchiveParams{})
if err != nil {
    panic(err)
}

// 删除 environment（仅当没有 session 引用它时）
_, err = client.Beta.Environments.Delete(ctx, environment.ID, anthropic.BetaEnvironmentDeleteParams{})
if err != nil {
    panic(err)
}

// 删除 session
_, err = client.Beta.Sessions.Delete(ctx, session.ID, anthropic.BetaSessionDeleteParams{})
if err != nil {
    panic(err)
}
```

---

## MCP Server 集成

```go
// Agent 声明 MCP server（此处无认证——认证放在 vault 中）
agent, err := client.Beta.Agents.New(ctx, anthropic.BetaAgentNewParams{
    Name: "GitHub Assistant",
    Model: anthropic.BetaManagedAgentsModelConfigParams{
        ID:   "claude-opus-4-8",
        Type: anthropic.BetaManagedAgentsModelConfigParamsTypeModelConfig,
    },
    MCPServers: []anthropic.BetaManagedAgentsURLMCPServerParams{{
        Type: anthropic.BetaManagedAgentsURLMCPServerParamsTypeURL,
        Name: "github",
        URL:  "https://api.githubcopilot.com/mcp/",
    }},
    Tools: []anthropic.BetaAgentNewParamsToolUnion{
        {
            OfAgentToolset20260401: &anthropic.BetaManagedAgentsAgentToolset20260401Params{
                Type: anthropic.BetaManagedAgentsAgentToolset20260401ParamsTypeAgentToolset20260401,
            },
        },
        {
            OfMCPToolset: &anthropic.BetaManagedAgentsMCPToolsetParams{
                Type:          anthropic.BetaManagedAgentsMCPToolsetParamsTypeMCPToolset,
                MCPServerName: "github",
            },
        },
    },
})
if err != nil {
    panic(err)
}

// Session 附加包含这些 MCP server URL 凭证的 vault
session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
    Agent: anthropic.BetaSessionNewParamsAgentUnion{
        OfBetaManagedAgentsAgents: &anthropic.BetaManagedAgentsAgentParams{
            Type:    anthropic.BetaManagedAgentsAgentParamsTypeAgent,
            ID:      agent.ID,
            Version: anthropic.Int(agent.Version),
        },
    },
    EnvironmentID: environment.ID,
    VaultIDs:      []string{vault.ID},
})
if err != nil {
    panic(err)
}
```

参见 `shared/managed-agents-tools.md` §Vaults 了解创建 vault 和添加凭证的方法。

---

## Vault

```go
// 创建 vault
vault, err := client.Beta.Vaults.New(ctx, anthropic.BetaVaultNewParams{
    DisplayName: "Alice",
    Metadata:    map[string]string{"external_user_id": "usr_abc123"},
})
if err != nil {
    panic(err)
}

// 添加 OAuth 凭证
credential, err := client.Beta.Vaults.Credentials.New(ctx, vault.ID, anthropic.BetaVaultCredentialNewParams{
    DisplayName: anthropic.String("Alice's Slack"),
    Auth: anthropic.BetaVaultCredentialNewParamsAuthUnion{
        OfMCPOAuth: &anthropic.BetaManagedAgentsMCPOAuthCreateParams{
            Type:         anthropic.BetaManagedAgentsMCPOAuthCreateParamsTypeMCPOAuth,
            MCPServerURL: "https://mcp.slack.com/mcp",
            AccessToken:  "xoxp-...",
            ExpiresAt:    anthropic.Time(time.Date(2026, time.April, 15, 0, 0, 0, 0, time.UTC)),
            Refresh: anthropic.BetaManagedAgentsMCPOAuthRefreshParams{
                TokenEndpoint: "https://slack.com/api/oauth.v2.access",
                ClientID:      "1234567890.0987654321",
                Scope:         anthropic.String("channels:read chat:write"),
                RefreshToken:  "xoxe-1-...",
                TokenEndpointAuth: anthropic.BetaManagedAgentsMCPOAuthRefreshParamsTokenEndpointAuthUnion{
                    OfClientSecretPost: &anthropic.BetaManagedAgentsTokenEndpointAuthPostParam{
                        Type:         anthropic.BetaManagedAgentsTokenEndpointAuthPostParamTypeClientSecretPost,
                        ClientSecret: "abc123...",
                    },
                },
            },
        },
    },
})
if err != nil {
    panic(err)
}

// 轮换凭证（例如 token 刷新后）
_, err = client.Beta.Vaults.Credentials.Update(ctx, credential.ID, anthropic.BetaVaultCredentialUpdateParams{
    VaultID: vault.ID,
    Auth: anthropic.BetaVaultCredentialUpdateParamsAuthUnion{
        OfMCPOAuth: &anthropic.BetaManagedAgentsMCPOAuthUpdateParams{
            Type:        anthropic.BetaManagedAgentsMCPOAuthUpdateParamsTypeMCPOAuth,
            AccessToken: anthropic.String("xoxp-new-..."),
            ExpiresAt:   anthropic.Time(time.Date(2026, time.May, 15, 0, 0, 0, 0, time.UTC)),
            Refresh: anthropic.BetaManagedAgentsMCPOAuthRefreshUpdateParams{
                RefreshToken: anthropic.String("xoxe-1-new-..."),
            },
        },
    },
})
if err != nil {
    panic(err)
}

// 归档 vault
_, err = client.Beta.Vaults.Archive(ctx, vault.ID, anthropic.BetaVaultArchiveParams{})
if err != nil {
    panic(err)
}
```

---

## GitHub 仓库集成

将 GitHub 仓库挂载为 session 资源（vault 持有 GitHub MCP 凭证）：

```go
session, err := client.Beta.Sessions.New(ctx, anthropic.BetaSessionNewParams{
    Agent:         anthropic.BetaSessionNewParamsAgentUnion{OfString: anthropic.String(agent.ID)},
    EnvironmentID: environment.ID,
    VaultIDs:      []string{vault.ID},
    Resources: []anthropic.BetaSessionNewParamsResourceUnion{
        {
            OfGitHubRepository: &anthropic.BetaManagedAgentsGitHubRepositoryResourceParams{
                Type:               anthropic.BetaManagedAgentsGitHubRepositoryResourceParamsTypeGitHubRepository,
                URL:                "https://github.com/org/repo",
                MountPath:          anthropic.String("/workspace/repo"),
                AuthorizationToken: "ghp_your_github_token",
            },
        },
    },
})
if err != nil {
    panic(err)
}
```

同一 session 上的多个仓库：

```go
resources := []anthropic.BetaSessionNewParamsResourceUnion{
    {
        OfGitHubRepository: &anthropic.BetaManagedAgentsGitHubRepositoryResourceParams{
            Type:               anthropic.BetaManagedAgentsGitHubRepositoryResourceParamsTypeGitHubRepository,
            URL:                "https://github.com/org/frontend",
            MountPath:          anthropic.String("/workspace/frontend"),
            AuthorizationToken: "ghp_your_github_token",
        },
    },
    {
        OfGitHubRepository: &anthropic.BetaManagedAgentsGitHubRepositoryResourceParams{
            Type:               anthropic.BetaManagedAgentsGitHubRepositoryResourceParamsTypeGitHubRepository,
            URL:                "https://github.com/org/backend",
            MountPath:          anthropic.String("/workspace/backend"),
            AuthorizationToken: "ghp_your_github_token",
        },
    },
}
```

轮换仓库的授权 token：

```go
listed, err := client.Beta.Sessions.Resources.List(ctx, session.ID, anthropic.BetaSessionResourceListParams{})
if err != nil {
    panic(err)
}
repoResourceID := listed.Data[0].ID

_, err = client.Beta.Sessions.Resources.Update(ctx, repoResourceID, anthropic.BetaSessionResourceUpdateParams{
    SessionID:          session.ID,
    AuthorizationToken: "ghp_your_new_github_token",
})
if err != nil {
    panic(err)
}
```
