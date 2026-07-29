# Managed Agents — 概览

Managed Agents 为每个会话提供一个容器作为智能体的工作区。智能体循环运行在 Anthropic 的编排层上；容器是智能体*工具*执行的地方——bash 命令、文件操作、代码。你创建一个持久化的 **Agent** 配置（模型、系统提示词、工具、MCP 服务器、技能），然后启动引用它的 **Session**。会话向你流式返回事件；你发送用户消息和工具结果。

## ⚠️ 强制流程：Agent（一次）→ Session（每次运行）

**为什么 agent 是独立对象：版本管理。** Agent 是一个持久化的、有版本的配置——每次更新创建一个新的不可变版本，会话在创建时锁定到一个版本。这让你可以迭代 agent（调整提示词、添加工具）而不破坏已在运行的会话，在更改导致回归时回滚，以及并行 A/B 测试版本。如果你每次运行都 `agents.create()` 新建，这些都不可能。

每个会话引用一个预先创建的 `/v1/agents` 对象。创建 agent 一次，存储 ID，并在多次运行中复用它。

| 步骤 | 调用 | 频率 |
|---|---|---|
| 1 | `POST /v1/agents` — `model`、`system`、`tools`、`mcp_servers`、`skills` 在这里 | **一次。** 存储 `agent.id` **和** `agent.version`。 |
| 2 | `POST /v1/sessions` — `agent: "agent_abc123"` 或 `{type: "agent", id, version}` | **每次运行。** 字符串简写使用最新版本。 |

如果你即将在 `sessions.create()` 的 session body 上写 `model`、`system` 或 `tools`——**停下来**。这些字段在 `agents.create()` 上。Session 只接受一个*指针*。

**生成代码时，将设置与运行时分离。** `agents.create()` 属于设置脚本（或有守卫的 `if agent_id is None:` 块），不在热路径的顶部。如果用户的代码在每次调用时都执行 `agents.create()`，它们在积累孤立的 agent 并白白支付创建延迟。正确的形状是：将 agent 定义为版本控制的 YAML 清单，用 `ant beta:agents create < agent.yaml` 一次性应用（或有守卫的设置脚本——见 `shared/anthropic-cli.md`），持久化返回的 ID（配置文件、环境变量、密钥管理器），每次运行加载 ID 并调用 `sessions.create()`。

**要更改 agent 的行为，使用 `POST /v1/agents/{id}`——不要创建新的。** 每次更新提升版本；运行中的会话保持其锁定版本，新会话获取最新版本（或通过 `{type: "agent", id, version}` 显式锁定）。见 `shared/managed-agents-core.md` → Agents → 版本管理。要在**一个运行中的会话**上更改 `tools`/`mcp_servers`/`vault_ids` 而不触碰 agent 对象，使用 `sessions.update()`——见 `shared/managed-agents-core.md` → 会话中途更新 agent 配置。

## Beta Headers

Managed Agents 处于 beta 阶段。SDK 自动设置所需的 beta headers：

| Beta Header                    | 启用的功能                                      |
| ------------------------------ | ---------------------------------------------------- |
| `managed-agents-2026-04-01`    | Agents、Environments、Sessions、Events、Session Resources、Session Threads、Outcomes、Multiagent、Vaults、Credentials、Memory Stores、Deployments |
| `skills-2025-10-02`            | Skills API（用于管理自定义 skill 定义）   |
| `files-api-2025-04-14`         | Files API，用于文件上传                           |

**哪个 beta header 用在哪里：** SDK 在 `client.beta.{agents,environments,sessions,vaults,memory_stores,deployments,deployment_runs}.*` 调用上自动设置 `managed-agents-2026-04-01`，在 `client.beta.files.*` / `client.beta.skills.*` 调用上自动设置 `files-api-2025-04-14` / `skills-2025-10-02`。你不需要在调用 Managed Agents 端点时添加 Skills 或 Files beta header。**例外——会话级文件列表：** `client.beta.files.list({scope_id: session.id})` 是一个 Files 端点，接受 Managed Agents 参数，因此需要**两个** header。在该调用上显式传递 `betas: ["managed-agents-2026-04-01"]`（SDK 添加 Files header；你添加 Managed Agents 的）。见 `shared/managed-agents-environments.md` → Session 输出。


## 阅读指南

| 用户想要...                       | 阅读这些文件                                        |
| -------------------------------------- | ------------------------------------------------------- |
| **从零开始 / "帮我设置一个 agent"** | `shared/managed-agents-onboarding.md` — 引导式访谈（WHERE→WHO→WHAT→WATCH），然后生成代码 |
| 了解 API 如何工作           | `shared/managed-agents-core.md`                         |
| 查看完整端点参考        | `shared/managed-agents-api-reference.md`                |
| **创建 agent**（必需的第一步） | `shared/managed-agents-core.md`（Agents 章节）+ 语言文件 |
| 更新/版本化 agent                | `shared/managed-agents-core.md`（Agents → 版本管理）——更新，不要重新创建 |
| 创建会话                       | `shared/managed-agents-core.md` + `{lang}/managed-agents/README.md`（cURL/C#: `curl/managed-agents.md`） |
| 配置工具和权限        | `shared/managed-agents-tools.md`                        |
| 设置 MCP 服务器                     | `shared/managed-agents-tools.md`（MCP Servers 章节）  |
| 流式事件 / 处理 tool_use        | `shared/managed-agents-events.md` + 语言文件       |
| 通过 webhook 获取会话状态变更通知（无需轮询） | `shared/managed-agents-webhooks.md` — Console 注册端点，HMAC 验证，精简负载 + 获取 |
| 定义 outcome / 基于评分标准的迭代循环 | `shared/managed-agents-outcomes.md` — `user.define_outcome` 事件，grader，`span.outcome_evaluation_*` 事件 |
| 协调多个 agent / 子 agent / 线程 | `shared/managed-agents-multiagent.md` — agent 上的 `multiagent: {type: "coordinator", agents: [...]}`，会话线程，交叉发布的工具确认 |
| 设置环境                    | `shared/managed-agents-environments.md` + 语言文件 |
| 在自己的基础设施 / VPC 中运行工具执行（自托管沙箱） | `shared/managed-agents-self-hosted-sandboxes.md` — `config:{type:"self_hosted"}`，`ANTHROPIC_ENVIRONMENT_KEY`，`EnvironmentWorker.run()` / `ant beta:worker poll` |
| 上传文件 / 附加仓库            | `shared/managed-agents-environments.md`（Resources）     |
| 给 agent 跨会话的持久记忆 | `shared/managed-agents-memory.md` — memory stores，`memory_store` 会话资源，前置条件，版本/编辑 |
| 将 agent/环境定义为版本控制的 YAML；从 shell 驱动 API | `shared/anthropic-cli.md` — `ant beta:agents create < agent.yaml`，`--transform`，`@file` 内联 |
| 存储凭据（MCP 认证、CLI/SDK 的 API 密钥） | `shared/managed-agents-tools.md`（Vaults 章节）— `mcp_oauth` / `static_bearer` / `environment_variable` |
| 调用需要密钥的非 MCP API / CLI | `shared/managed-agents-tools.md`（Vaults 章节）— `environment_variable` 凭据，在出口处替换。如果不适合（例如自托管沙箱），`shared/managed-agents-client-patterns.md` Pattern 9 通过自定义工具将密钥保留在主机端 |
| 按定期 cron 计划运行 agent | `shared/managed-agents-scheduled-deployments.md` — deployments，deployment runs，暂停/自动暂停 |

## 常见陷阱

- **先 Agent，后会话——无例外** — 会话的 `agent` 字段**只**接受字符串 ID 或 `{type: "agent", id, version}`。`model`、`system`、`tools`、`mcp_servers`、`skills` 是 `POST /v1/agents` 上的**顶层字段**，绝不在 `sessions.create()` 上。如果用户还没创建 agent，那是每个示例的第零步。
- **Agent 一次创建，不是每次运行** — `agents.create()` 是设置步骤。存储返回的 `agent_id` 并复用它；不要在热路径顶部调用 `agents.create()`。如果 agent 的配置需要更改，`POST /v1/agents/{id}`——每次更新创建一个新版本，会话可以锁定到特定版本以实现可复现性。
- **MCP 认证通过 vaults** — agent 的 `mcp_servers` 数组只声明 `{type, name, url}`（无认证）。凭据存在于 vaults（`client.beta.vaults.credentials.create`）中，通过 `vault_ids` 附加到会话。Anthropic 使用存储的 refresh token 自动刷新 OAuth 令牌。Vaults 还持有非 MCP 服务的 `environment_variable` 凭据（CLI、SDK、直接 API 调用）——在出口处替换，在沙箱中永不可见。
- **首次运行前核对资源** — 一个有明确请求但缺少工具、凭据、数据挂载或上下文的会话会在运行中途发现缺口，然后挣扎并放弃。创建会话之前，检查任务中的每个操作是否映射到已配置的工具/MCP 服务器，每个 MCP 服务器是否有 vault 凭据，每个引用的文件/主机是否已挂载/可达。帮助用户设置时，运行 `shared/managed-agents-onboarding.md` → §3 预检可行性检查中的核对。
- **流式获取事件** — `GET /v1/sessions/{id}/events/stream` 是实时接收 agent 输出的主要方式。
- **SSE 流没有重放——重连时合并** — 如果流在 `agent.tool_use`、`agent.mcp_tool_use` 或 `agent.custom_tool_use` 等待解析（前两者的 `user.tool_confirmation`，最后一个的 `user.custom_tool_result`）时断开，会话会死锁（客户端断开 → 会话空闲 → 重连发生 → 无客户端解析发生）。每次（重新）连接时：用 `GET /v1/sessions/{id}/events/stream` 打开流，获取 `GET /v1/sessions/{id}/events`，按事件 ID 去重，然后继续。见 `shared/managed-agents-events.md` → 断流后重连。
- **不要信任 HTTP 库的超时作为墙钟上限** — `requests` 的 `timeout=(c, r)` 和 `httpx.Timeout(n)` 是*按块*的读取超时；它们在每个字节后重置，所以涓流连接可以无限阻塞。对于原始 HTTP 轮询的硬截止时间，在循环层面跟踪 `time.monotonic()` 并显式退出。优先使用 SDK 的 `sessions.events.stream()` / `session.events.list()` 而非手写的 HTTP。见 `shared/managed-agents-events.md` → 接收事件。
- **消息排队** — 你可以在会话 `running` 或 `idle` 时发送事件；它们按顺序处理。无需等待响应再发送下一条消息。
- **环境 `config.type` 是 `"cloud"` 或 `"self_hosted"`** — `cloud` 在 Anthropic 的基础设施上运行容器；`self_hosted` 将工具执行移至你自己的（见 `shared/managed-agents-self-hosted-sandboxes.md`）。
- **归档对每个资源是永久的** — 归档 agent、环境、会话、vault、凭据或 memory store 使其变为只读，不可取消归档。对于 agent、环境和 memory store，归档的资源不能被新会话引用（现有会话继续运行）。不要将 `.archive()` 作为清理操作用于生产 agent、环境或 memory store——**在归档之前始终与用户确认**。
