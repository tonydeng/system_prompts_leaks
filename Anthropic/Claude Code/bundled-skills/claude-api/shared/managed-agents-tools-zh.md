> **说明**：本文件为英文原文（`managed-agents-tools.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Managed Agents — 工具与 Skills

## 工具

### 服务器工具 vs 客户端工具

| 类型 | 谁运行 | 工作方式 |
|---|---|---|
| **预构建 Claude Agent 工具**（`agent_toolset_20260401`） | Anthropic，在 session 的容器上运行（对于 `cloud` 环境；对于 `self_hosted`，**你的** worker 提供并运行它们——参见 `shared/managed-agents-self-hosted-sandboxes.md`） | 文件操作、bash、web 搜索等。一次性全部启用，或通过 `enabled: true/false` 逐个配置。 |
| **MCP 工具**（`mcp_toolset`） | Anthropic 的编排层 | 已连接 MCP 服务器暴露的能力。通过 toolset 按服务器授权访问。 |
| **自定义工具** | **你**——你的应用处理调用并返回结果 | Agent 发出 `agent.custom_tool_use` 事件，session 进入 `idle`，你发回 `user.custom_tool_result` 事件。 |

**建议：** 通过 `agent_toolset_20260401` 启用所有预构建工具，然后按需逐个禁用。

**版本管理：** toolset 是版本化的静态资源。当底层工具发生变化时，会创建新的 toolset 版本（因此是 `_20260401`），让你始终确切知道你会得到什么。

### Agent 工具集

`agent_toolset_20260401` 提供以下内置工具：

| 工具                   | 描述                              |
| ---------------------- | ---------------------------------------- |
| `bash` | 在 shell session 中执行 bash 命令 |
| `read` | 从本地文件系统读取文件，包括文本、图片、PDF 和 Jupyter notebook |
| `write` | 向本地文件系统写入文件 |
| `edit` | 在文件中执行字符串替换 |
| `glob` | 使用 glob 模式进行快速文件模式匹配 |
| `grep` | 使用 regex 模式进行文本搜索 |
| `web_fetch` | 从 URL 获取内容 |
| `web_search` | 搜索网络获取信息 |

启用完整工具集：

```json
{
  "tools": [
    { "type": "agent_toolset_20260401" }
  ]
}
```

### 逐工具配置

覆盖单个工具的默认值。此示例启用除 bash 外的所有工具：

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": true },
      "configs": [
        { "name": "bash", "enabled": false }
      ]
    }
  ]
}
```

| 字段 | 必需 | 描述 |
|---|---|---|
| `type` | ✅ | `"agent_toolset_20260401"` |
| `default_config` | ❌ | 应用于所有工具。`{ "enabled": bool, "permission_policy": {...} }` |
| `configs` | ❌ | 逐工具覆盖：`[{ "name": "...", "enabled": bool, "permission_policy": {...} }]` |

### 权限策略

控制服务器执行的工具（agent 工具集 + MCP）何时自动运行 vs 等待审批。不适用于自定义工具。

| 策略 | 行为 |
|---|---|
| `always_allow` | 工具自动执行（默认） |
| `always_ask` | Session 发出 `session.status_idle` 并暂停，直到你发送 `tool_confirmation` 事件 |

```json
{
  "type": "agent_toolset_20260401",
  "default_config": {
    "enabled": true,
    "permission_policy": { "type": "always_allow" }
  },
  "configs": [
    { "name": "bash", "permission_policy": { "type": "always_ask" } }
  ]
}
```

**响应 `always_ask`：** 发送 `user.tool_confirmation` 事件，其中 `tool_use_id` 来自触发的 `agent_tool_use`/`mcp_tool_use` 事件：

```json
{ "type": "tool_confirmation", "tool_use_id": "sevt_abc123", "result": "allow" }
{ "type": "tool_confirmation", "tool_use_id": "sevt_def456", "result": "deny", "message": "Read .env.example instead" }
```

deny 上的可选 `message` 会传递给 agent，使其可以调整方法。

要仅启用特定工具，将默认值关闭并逐工具开启：

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": false },
      "configs": [
        { "name": "bash", "enabled": true },
        { "name": "read", "enabled": true }
      ]
    }
  ]
}
```

### 自定义工具（客户端）

自定义工具由**你的应用**执行，而非 Anthropic。流程如下：

1. Agent 决定使用工具 → session 发出带有输入的 `agent.custom_tool_use` 事件
2. Session 进入 `idle` 等待你
3. 你的应用执行工具
4. 你发回带有输出的 `user.custom_tool_result` 事件
5. Session 恢复为 `running`

无需权限策略——执行方是你。

```json
{
  "tools": [
    {
      "type": "custom",
      "name": "get_weather",
      "description": "Fetch current weather for a city.",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": { "type": "string", "description": "City name" }
        },
        "required": ["city"]
      }
    }
  ]
}
```

### MCP 服务器

MCP（Model Context Protocol）服务器暴露标准化的第三方能力（例如 Asana、GitHub、Linear）。**配置分布在 agent 和 vault 上：**

1. **Agent 创建**声明要连接哪些服务器（`type`、`name`、`url`——无认证）。Agent 的 `mcp_servers` 数组没有认证字段。
2. **Vault** 存储 OAuth 凭证。通过 session 创建时的 `vault_ids` 附加。

这样可以将密钥排除在可复用的 agent 定义之外。每个 vault 凭证绑定到一个 MCP 服务器 URL；Anthropic 按 URL 将凭证匹配到服务器。

**Agent 侧——声明服务器（无认证）：**

| 字段 | 必需 | 描述 |
|---|---|---|
| `type` | ✅ | `"url"` |
| `name` | ✅ | 唯一名称——被 `mcp_toolset.mcp_server_name` 引用 |
| `url` | ✅ | MCP 服务器的端点 URL（Streamable HTTP 传输） |

```json
{
  "mcp_servers": [
    { "type": "url", "name": "linear", "url": "https://mcp.linear.app/mcp" }
  ],
  "tools": [
    { "type": "mcp_toolset", "mcp_server_name": "linear" }
  ]
}
```

**Session 侧——附加 vault：**

```json
{
  "agent": "agent_abc123",
  "environment_id": "env_abc123",
  "vault_ids": ["vlt_abc123"]
}
```

> 💡 **逐工具启用（经验观察）：** 已观察到 `mcp_toolset` 接受 `default_config: {enabled: false}` + `configs: [{name, enabled: true}]` 来实现白名单模式。API 参考仅展示最小形式 `{type, mcp_server_name}`。

> 💡 **在运行中的 session 上更改工具/MCP 服务器：** `sessions.update()` 可以在 session 处于 `idle` 状态时替换 `agent.tools`、`agent.mcp_servers` 和 `vault_ids`——这是一个 session 级别的覆盖，不会触及 agent 对象。参见 `shared/managed-agents-core.md` → 在 session 中途更新 agent 配置。

**大型 MCP 工具输出。** 如果 MCP 工具返回超过 **100K token**，输出会自动卸载到沙箱中的文件——agent 收到截断的预览加上文件路径，可以通过 `read` 读取完整内容。无需配置。

**无效的 vault 凭证不会阻止 session 创建。** 如果 vault 凭证对声明的 MCP 服务器无效，session 仍会成功创建；`session.error` 事件描述 MCP 认证失败，认证在下次 `session.status_idle` → `session.status_running` 转换时重试。

> ⚠️ **MCP 认证 token ≠ REST API token。** 托管 MCP 服务器（`mcp.notion.com`、`mcp.linear.app` 等）通常需要 **OAuth bearer token**，而非服务原生的 API key。Notion `ntn_` 集成 token 对 Notion 的 REST API 进行认证，但**不能**用作 Notion MCP 服务器的 vault 凭证。这是不同的认证系统。

### Vault — 凭证存储

**Vault** 存储 Anthropic 代你管理的凭证。两类凭证：

- **MCP 凭证**（`mcp_oauth`、`static_bearer`）——以 `mcp_server_url` 为键。当 agent 连接到该 URL 的服务器时，token 自动注入。`mcp_oauth` token 通过标准 OAuth 2.0 `refresh_token` 授权自动刷新。这是认证 MCP 服务器的唯一方式。
- **环境变量**（`environment_variable`）——以 `secret_name`（环境变量名）为键。沙箱只看到**不透明占位符**；真正的密钥在**出口处**被替换到出站请求中。用于任何通过环境变量认证的服务：CLI（`aws`、`gcloud`、`stripe`）、SDK，或通过 `bash` 工具直接 `curl` 调用。

你提供的密钥字段（`token`、`access_token`、`refresh_token`、`client_secret`、`secret_value`）是只写的——永远不会在 API 响应中返回。

#### 凭证与沙箱

Vault 存储凭证；这些凭证**永远不会进入沙箱**。这是一个刻意的安全边界——在沙箱中运行的代码（包括 agent 写入的任何内容）无法读取或泄露 vault 中的凭证，即使在提示注入下也不行。相反，凭证由 Anthropic 侧的代理在请求离开沙箱**之后**注入：

- **MCP 工具调用**通过 Anthropic 侧的代理路由，该代理从 vault 获取凭证并添加到出站请求中。
- **对附加 GitHub 仓库的 Git 操作**（`git pull`、`git push`、GitHub REST 调用）通过 git 代理路由，以相同方式注入 `github_repository` 资源的 `authorization_token`。
- **环境变量凭证**在沙箱中显示为不透明占位符；真实值在出口处替换占位符，仅在请求发往凭证允许的主机时进行。替换仅覆盖请求的**header 和 body**——嵌入在 **URL 路径**中的密钥永远不会被替换，因此路径密钥端点（例如 Slack incoming-webhook URL）无法被 vault 化；改用基于 header 的认证（对于 Slack：通过 `chat.postMessage` 在 `Authorization` 中使用 bot token）。

**当 vault 凭证不适用时**（例如自托管沙箱——`environment_variable` 尚不支持），**注册自定义工具：** agent 发出 `agent.custom_tool_use`，你的编排器（已持有凭证）执行调用并通过同一认证事件流返回 `user.custom_tool_result`。不暴露公共端点；沙箱永远看不到密钥。参见 `shared/managed-agents-client-patterns.md` → Pattern 9。

**不要将 API key 放在系统提示或用户消息中作为变通方法**——它们会持久存在于 session 的事件历史中。

> 以前在内部称为 TATs（Tool/Tenant Access Tokens）。

**流程：**

1. 创建 vault（`client.beta.vaults.create(...)`）——每个租户/用户一个，或共享一个，取决于你的模型
2. 向其添加凭证（`client.beta.vaults.credentials.create(...)`）——MCP 凭证以 MCP 服务器 URL 为键；环境变量凭证以 `secret_name` 为键
3. 在 session 创建时通过 `vault_ids: ["vlt_..."]` 引用 vault
4. Anthropic 在 OAuth token 过期前自动刷新并在运行时替换密钥

**MCP OAuth 凭证结构**：

```json
{
  "display_name": "Notion (workspace-foo)",
  "auth": {
    "type": "mcp_oauth",
    "mcp_server_url": "https://mcp.notion.com/mcp",
    "access_token": "<current access token>",
    "expires_at": "2026-04-02T14:00:00Z",
    "refresh": {
      "refresh_token": "<refresh token>",
      "client_id": "<your OAuth client_id>",
      "token_endpoint": "https://api.notion.com/v1/oauth/token",
      "token_endpoint_auth": { "type": "none" }
    }
  }
}
```

`refresh` 块是启用自动刷新的关键——`token_endpoint` 是 Anthropic 发送 `refresh_token` 授权的地方。`token_endpoint_auth` 是一个判别联合：

| `type` | 结构 | 适用场景 |
|---|---|---|
| `"none"` | `{type: "none"}` | 公共 OAuth 客户端（无 secret） |
| `"client_secret_basic"` | `{type: "client_secret_basic", client_secret: "..."}` | 机密客户端，通过 HTTP Basic auth 传递 secret |
| `"client_secret_post"` | `{type: "client_secret_post", client_secret: "..."}` | 机密客户端，secret 在请求体中传递 |

如果你只有 access token 而没有刷新能力，则完全省略 `refresh`——它在过期前可用，之后 agent 失去访问权限。

> 💡 **获取 OAuth token。** 如何获取初始 access 和 refresh token 取决于 MCP 服务器——查阅其文档。一旦获得，使用上述结构将其存储在 vault 凭证中；Anthropic 从 `refresh.token_endpoint` 自动刷新。

**环境变量凭证结构**：

```json
{
  "display_name": "Twilio API key for sandbox",
  "auth": {
    "type": "environment_variable",
    "secret_name": "TWILIO_API_KEY",
    "secret_value": "sk-your-secret-here",
    "networking": {
      "type": "limited",
      "allowed_hosts": ["api.twilio.com", "*.twilio.com"]
    }
  }
}
```

`networking.allowed_hosts` 控制密钥可以被替换到发往哪些出站主机的请求中——`{"type": "limited", "allowed_hosts": [...]}` 或 `{"type": "unrestricted"}`（如果无法提前枚举域名）。强烈建议限制：它防止密钥被发送到未授权的主机。

**`injection_location`**（可选，`networking` 的同级字段）控制密钥在出站请求中**哪里**被替换——`{header: bool, body: bool}`。两者独立：`allowed_hosts` 限定替换请求可以发往*哪些主机*；`injection_location` 限定在所有这些主机中密钥被替换到请求的*哪些部分*。大多数服务从请求 header 中读取 API key，因此 `{"header": true}` 是更窄的配置——请求体通常由 agent 正在处理的内容组装而成，使 body 成为更广泛的暴露面。在禁用位置中的占位符**既不会被替换也不会被剥离**——字面的不透明占位符字符串会按原样发送给第三方。

| 操作 | `injection_location` 语义 |
|---|---|
| 创建凭证 | 完全省略该字段 → 两个位置都启用。提供对象 → 省略的任何字段默认为 `false`（`{"header": true}` 创建仅 header 的凭证）。 |
| 更新凭证 | 字段**逐个合并**——`{"body": false}` 禁用 body 替换并保持 `header` 不变。对于运行中的 session，更新在 session 的下次操作时生效。 |

凭证必须至少启用一个位置；创建或更新如果会禁用两者则返回 400，对对象或任一字段显式指定 `null` 也会返回 400（请改为省略）。响应始终返回两个字段及其解析后的值。

> ⚠️ **两层网络配置，都必需。** 凭证上的 `networking.allowed_hosts` 控制哪些请求*使用密钥*，而非哪些请求*被允许*。Agent 还必须能够在**环境级别**访问该域名（`unrestricted`，或该域名列在环境的 `allowed_hosts` 中——参见 `shared/managed-agents-environments.md`）。任一层缺少该域名意味着密钥替换请求失败。

> ⚠️ **客户端验证注意事项。** 替换发生在出口处，而非沙箱内——在进行网络请求之前在本地验证凭证*格式*的客户端（例如检查 key 是否以 `sk-` 开头的 CLI）会看到不透明占位符并可能在启动时失败。如果客户端在任何网络调用之前拒绝凭证，原因就在这里。

> 💡 **最小化密钥权限范围。** Agent 可以做密钥允许的任何事；权限范围超出任务需求的密钥在 agent 行为异常时会增加影响范围。

**不支持自托管沙箱**——`environment_variable` 凭证需要 Anthropic 管理的出口。参见 `shared/managed-agents-self-hosted-sandboxes.md`。

**约束（所有凭证类型）：**

- **每个 vault 中键唯一。** `mcp_server_url`（MCP 凭证）和 `secret_name`（环境变量凭证）在 vault 的活跃凭证中必须唯一；重复返回 409。
- **键不可变。** 密钥值、`display_name` 和（环境变量凭证上的）`injection_location` 可以更新；要更改 `mcp_server_url`、`secret_name`、`token_endpoint` 或 `client_id`，需归档凭证并创建新的。归档会清除密钥并释放键供替换使用。
- **每个 vault 最多 20 个凭证。**
- 凭证按提供的方式存储，**在 session 运行时之前不进行验证**——无效凭证在 session 期间表现为认证或下游错误，会被发出但不阻止 session 继续。

**作用域：** Vault 是 workspace 级别的。API workspace 中任何具有 developer+ 角色的人都可以创建、读取（仅元数据——密钥是只写的）和附加 vault。`vault_ids` 可以在 session**创建**时设置，但不能通过 session 更新设置（SDK 文档字符串说"尚不支持；设置此字段的请求会被拒绝"）。

---

## Skills

Skills 是可复用的、基于文件系统的资源，为你的 agent 提供特定领域专业知识：工作流、上下文和最佳实践，将通用 agent 转变为专家。与提示（用于一次性任务的对话级指令）不同，skills 按需加载，无需在多个对话中重复提供相同的指导。

两种类型——工作方式相同；agent 在与当前任务相关时自动使用它们：

| 类型 | 是什么 |
|---|---|
| **预构建 Anthropic skills** | 常见文档任务（PowerPoint、Excel、Word、PDF）。按名称引用（例如 `xlsx`）。 |
| **自定义 skills** | 你通过 Skills API 在组织中创建的 skills。通过 `skill_id` + 可选 `version` 引用。 |

**每个 agent 最多 20 个 skills。** Agent 创建使用 `managed-agents-2026-04-01`；单独的 Skills API（用于管理自定义 skill 定义）使用 `skills-2025-10-02`。

### 在 session 上启用 skills

Skills 通过 `agents.create()` 附加到 **agent** 定义上：

```ts
const agent = await client.beta.agents.create(
  {
    name: "Financial Agent",
    model: "claude-opus-4-8",
    system: "You are a financial analysis agent.",
    skills: [
      { type: "anthropic", skill_id: "xlsx" },
      { type: "custom", skill_id: "skill_abc123", version: "latest" },
    ],
  }
);
```

Python：

```python
agent = client.beta.agents.create(
    name="Financial Agent",
    model="claude-opus-4-8",
    system="You are a financial analysis agent.",
    skills=[
        {"type": "anthropic", "skill_id": "xlsx"},
        {"type": "custom", "skill_id": "skill_abc123", "version": "latest"},
    ]
)
```

**Skill 引用字段：**

| 字段 | Anthropic skill | 自定义 skill |
|---|---|---|
| `type` | `"anthropic"` | `"custom"` |
| `skill_id` | Skill 名称（例如 `"xlsx"`、`"docx"`、`"pptx"`、`"pdf"`） | 来自 Skills API 的 Skill ID（例如 `"skill_abc123"`） |
| `version` | — | `"latest"` 或特定版本号 |

### Skills API

| 操作             | 方法   | 路径                                            |
| --------------------- | -------- | ----------------------------------------------- |
| 创建 Skill          | `POST`   | `/v1/skills`                                    |
| 列出 Skills           | `GET`    | `/v1/skills`                                    |
| 获取 Skill             | `GET`    | `/v1/skills/{id}`                               |
| 删除 Skill          | `DELETE` | `/v1/skills/{id}`                               |
| 创建版本        | `POST`   | `/v1/skills/{id}/versions`                      |
| 列出版本         | `GET`    | `/v1/skills/{id}/versions`                      |
| 获取版本           | `GET`    | `/v1/skills/{id}/versions/{version}`            |
| 删除版本        | `DELETE` | `/v1/skills/{id}/versions/{version}`            |
