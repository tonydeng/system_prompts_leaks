# 托管代理 — Ruby

> **此处未展示的绑定：** 本 README 涵盖了 Ruby 最常见的托管代理流程。如果你需要的类、方法、命名空间、字段或行为未在此展示，请通过 WebFetch 获取 Ruby SDK 仓库**或相关文档页面**（来自 `shared/live-sources.md`），而不是猜测。不要从 cURL 形状或其他语言的 SDK 进行推断。

> **代理是持久的——创建一次，通过 ID 引用。** 存储 `client.beta.agents.create` 返回的代理 ID，并将其传递给后续每次 `client.beta.sessions.create`；不要在请求路径中调用 `agents.create`。**推荐做法：** 将代理和环境定义为受版本控制的 YAML，使用 `ant` CLI 应用——参见 `shared/anthropic-cli.md`（其实时文档 URL 在 `shared/live-sources.md` 中）。CLI 拥有控制面（创建/更新），你的代码拥有数据面（使用存储的 ID 进行会话）。下面的示例展示了必须以编程方式配置时的代码内创建；在生产环境中，创建调用应属于设置阶段，而非请求路径。

## 安装

```bash
gem install anthropic
```

## 客户端初始化

```ruby
require "anthropic"

# 默认（使用 ANTHROPIC_API_KEY 环境变量）
client = Anthropic::Client.new

# 显式 API 密钥
client = Anthropic::Client.new(api_key: "your-api-key")
```

> ⚠️ **尾部下划线：** Ruby SDK 使用 `system_:` 和 `send_(`（尾部下划线）来避免遮蔽 `Kernel#system` 和 `Kernel#send`。在托管代理代码中始终使用这些形式。

---

## 创建环境

```ruby
environment = client.beta.environments.create(
  name: "my-dev-env",
  config: {
    type: "cloud",
    networking: {type: "unrestricted"}
  }
)
puts "Environment ID: #{environment.id}" # env_...
```

---

## 创建代理（必需的第一步）

> ⚠️ **没有内联代理配置。** `model`/`system_`/`tools` 位于代理对象上，而非会话上。始终以 `client.beta.agents.create()` 开始——会话接受 `agent: agent.id` 或类型化哈希形式 `agent: {type: "agent", id: agent.id, version: agent.version}`。

### 最小示例

```ruby
# 1. 创建代理（可复用、有版本）
agent = client.beta.agents.create(
  name: "Coding Assistant",
  model: :"claude-opus-4-8",
  system_: "You are a helpful coding assistant.",
  tools: [{type: "agent_toolset_20260401"}]
)

# 2. 启动会话
session = client.beta.sessions.create(
  agent: {type: "agent", id: agent.id, version: agent.version},
  environment_id: environment.id,
  title: "Quickstart session"
)
puts "Session ID: #{session.id}"
puts "Trace: https://platform.claude.com/workspaces/default/sessions/#{session.id}"  # 如果 API 密钥不在 Default 工作区，将 'default' 替换为你的工作区 ID
```

### 更新代理

更新会创建新版本；代理对象在每个版本上是不可变的。

```ruby
updated_agent = client.beta.agents.update(
  agent.id,
  version: agent.version,
  system_: "You are a helpful coding agent. Always write tests."
)
puts "New version: #{updated_agent.version}"

# 列出所有版本
client.beta.agents.versions.list(agent.id).auto_paging_each do |version|
  puts "Version #{version.version}: #{version.updated_at.iso8601}"
end

# 归档代理
archived = client.beta.agents.archive(agent.id)
puts "Archived at: #{archived.archived_at.iso8601}"
```

---

## 发送用户消息

```ruby
client.beta.sessions.events.send_(
  session.id,
  events: [{
    type: "user.message",
    content: [{type: "text", text: "Review the auth module"}]
  }]
)
```

> 💡 **流优先：** 在发送消息*之前*（或同时）打开流。流仅投递在它打开之后发生的事件——先发送后开流意味着早期事件会以缓冲批次到达。参见[引导模式](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式事件（SSE）

```ruby
# 先打开流，再发送用户消息
stream = client.beta.sessions.events.stream_events(session.id)

client.beta.sessions.events.send_(
  session.id,
  events: [{
    type: "user.message",
    content: [{type: "text", text: "Summarize the repo README"}]
  }]
)

stream.each do |event|
  case event.type
  in :"agent.message"
    event.content.each { |block| print block.text }
  in :"agent.tool_use"
    puts "\n[Using tool: #{event.name}]"
  in :"session.status_idle"
    break
  in :"session.error"
    puts "\n[Error: #{event.error&.message || "unknown"}]"
    break
  else
    # 忽略其他事件类型
  end
end
```

> ℹ️ 事件 `.type` 是 Symbol（用 `:"agent.message"` 比较，而非 `"agent.message"`）。

### 重连和追踪

会话中途重连时，先列出历史事件进行去重，再追踪实时事件：

```ruby
require "set"

stream = client.beta.sessions.events.stream_events(session.id)

# 流已打开并正在缓冲。在追踪实时事件前列出历史。
seen_event_ids = Set.new
client.beta.sessions.events.list(session.id).auto_paging_each { |past| seen_event_ids << past.id }

# 追踪实时事件，跳过已见过的
stream.each do |event|
  next if seen_event_ids.include?(event.id)
  seen_event_ids << event.id
  case event.type
  in :"agent.message"
    event.content.each { |block| print block.text }
  in :"session.status_idle"
    break
  else
    # 忽略其他事件类型
  end
end
```

---

## 提供自定义工具结果

> ℹ️ `user.custom_tool_result` 的 Ruby 托管代理绑定尚未在本技能或应用源示例中文档化。关于传输格式请参阅 `shared/managed-agents-events.md`，关于对应参数请参阅 `anthropic` Ruby gem 仓库。

---

## 轮询事件

```ruby
client.beta.sessions.events.list(session.id).auto_paging_each do |event|
  puts "#{event.type}: #{event.id}"
end
```

---

## 上传文件

```ruby
require "pathname"

file = client.beta.files.upload(file: Pathname("data.csv"))
puts "File ID: #{file.id}"

# 在会话中挂载
session = client.beta.sessions.create(
  agent: agent.id,
  environment_id: environment.id,
  resources: [
    {
      type: "file",
      file_id: file.id,
      mount_path: "/workspace/data.csv"
    }
  ]
)
```

### 在现有会话上添加和管理资源

```ruby
# 向打开的会话附加额外文件
resource = client.beta.sessions.resources.add(
  session.id,
  type: "file",
  file_id: file.id
)
puts resource.id # "sesrsc_01ABC..."

# 列出会话上的资源
listed = client.beta.sessions.resources.list(session.id)
listed.data.each { |entry| puts "#{entry.id} #{entry.type}" }

# 分离资源
client.beta.sessions.resources.delete(resource.id, session_id: session.id)
```

---

## 列出和下载会话文件

```ruby
files = client.beta.files.list(scope_id: "sesn_abc123", betas: ["managed-agents-2026-04-01"])
content = client.beta.files.download(files.data[0].id)
File.binwrite("output.txt", content.read)
```

---

## 会话管理

```ruby
# 列出环境
environments = client.beta.environments.list

# 检索特定环境
env = client.beta.environments.retrieve(environment.id)

# 归档环境（只读，现有会话继续运行）
client.beta.environments.archive(environment.id)

# 删除环境（仅当没有会话引用它时）
client.beta.environments.delete(environment.id)

# 删除会话
client.beta.sessions.delete(session.id)
```

---

## MCP 服务器集成

```ruby
# 代理声明 MCP 服务器（此处无认证——认证放在凭据库中）
agent = client.beta.agents.create(
  name: "GitHub Assistant",
  model: :"claude-opus-4-8",
  mcp_servers: [
    {
      type: "url",
      name: "github",
      url: "https://api.githubcopilot.com/mcp/"
    }
  ],
  tools: [
    {type: "agent_toolset_20260401"},
    {type: "mcp_toolset", mcp_server_name: "github"}
  ]
)

# 会话附加包含这些 MCP 服务器 URL 凭据的凭据库
session = client.beta.sessions.create(
  agent: {type: "agent", id: agent.id, version: agent.version},
  environment_id: environment.id,
  vault_ids: [vault.id]
)
```

创建凭据库和添加凭据请参见 `shared/managed-agents-tools.md` §Vaults。

---

## 凭据库（Vaults）

```ruby
# 创建凭据库
vault = client.beta.vaults.create(
  display_name: "Alice",
  metadata: {external_user_id: "usr_abc123"}
)
puts vault.id # "vlt_01ABC..."

# 添加 OAuth 凭据
credential = client.beta.vaults.credentials.create(
  vault.id,
  display_name: "Alice's Slack",
  auth: {
    type: "mcp_oauth",
    mcp_server_url: "https://mcp.slack.com/mcp",
    access_token: "xoxp-...",
    expires_at: "2026-04-15T00:00:00Z",
    refresh: {
      token_endpoint: "https://slack.com/api/oauth.v2.access",
      client_id: "1234567890.0987654321",
      scope: "channels:read chat:write",
      refresh_token: "xoxe-1-...",
      token_endpoint_auth: {
        type: "client_secret_post",
        client_secret: "abc123..."
      }
    }
  }
)

# 轮换凭据（如令牌刷新后）
client.beta.vaults.credentials.update(
  credential.id,
  vault_id: vault.id,
  auth: {
    type: "mcp_oauth",
    access_token: "xoxp-new-...",
    expires_at: "2026-05-15T00:00:00Z",
    refresh: {refresh_token: "xoxe-1-new-..."}
  }
)

# 归档凭据库
client.beta.vaults.archive(vault.id)
```

---

## GitHub 仓库集成

将 GitHub 仓库挂载为会话资源（凭据库持有 GitHub MCP 凭据）：

```ruby
session = client.beta.sessions.create(
  agent: agent.id,
  environment_id: environment.id,
  vault_ids: [vault.id],
  resources: [
    {
      type: "github_repository",
      url: "https://github.com/org/repo",
      mount_path: "/workspace/repo",
      authorization_token: "ghp_your_github_token"
    }
  ]
)
```

同一会话上的多个仓库：

```ruby
resources = [
  {
    type: "github_repository",
    url: "https://github.com/org/frontend",
    mount_path: "/workspace/frontend",
    authorization_token: "ghp_your_github_token"
  },
  {
    type: "github_repository",
    url: "https://github.com/org/backend",
    mount_path: "/workspace/backend",
    authorization_token: "ghp_your_github_token"
  }
]
```

轮换仓库的授权令牌：

```ruby
listed = client.beta.sessions.resources.list(session.id)
repo_resource_id = listed.data.first.id

client.beta.sessions.resources.update(
  repo_resource_id,
  session_id: session.id,
  authorization_token: "ghp_your_new_github_token"
)
```
