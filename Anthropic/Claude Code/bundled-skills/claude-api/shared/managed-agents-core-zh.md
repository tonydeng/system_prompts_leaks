> **说明**：本文件为英文原文（`managed-agents-core.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Managed Agents — 核心概念

## 架构

Managed Agents 围绕四个核心概念构建：

| 概念 | 端点 | 是什么 |
|---|---|---|
| **Agent** | `/v1/agents` | 一个持久化的、版本化的对象，定义智能体的能力和人设：模型、系统提示、工具、MCP 服务器、技能。**必须在启动会话之前创建。** 参见下文 Agents 部分。 |
| **Session** | `/v1/sessions` | 与智能体的有状态交互。通过 ID + 环境 + 初始指令引用预创建的智能体。产生事件流。 |
| **Environment** | `/v1/environments` | 定义容器配置的模板。 |
| **Container** | N/A | 一个隔离的计算实例，智能体的**工具**在其中执行（bash、文件操作、代码）。智能体循环不在此运行，它在 Anthropic 的编排层上运行，并通过工具调用操作容器。 |

```
                        ┌─────────────────────────────────────┐
                        │  Anthropic orchestration layer      │
Agent (config) ───────▶│  (agent loop: Claude + tool calls)  │
                        └──────────────┬──────────────────────┘
                                       │ tool calls
                                       ▼
Environment (template) ──▶ Container (tool execution workspace)
                                  │
                          Session ─┤
                                  ├── Resources (files, repos, memory stores — attached at startup)
                                  ├── Vault IDs (MCP credential references)
                                  └── Conversation (event stream in/out)
```

> **Agent 创建是先决条件。** 会话通过 ID 引用预创建的智能体 — `model`/`system`/`tools` 位于智能体对象上，从不在会话上。每个流程从 `POST /v1/agents` 开始。

---

## 会话生命周期

```
rescheduling → running ↔ idle → terminated
```

| 状态         | 描述                                                        |
| -------------- | ------------------------------------------------------------------ |
| `idle` | 智能体已完成当前任务，正在等待输入。它要么等待输入以通过 `user.message` 继续工作，要么被阻塞等待 `user.custom_tool_result` 或 `user.tool_confirmation`。附加的 `stop_reason` 包含关于智能体为何停止工作的更多信息。 |
| `running` | 会话已开始运行，智能体正在积极工作。 |
| `rescheduling` | 会话在可重试错误发生后正在（重新）调度，准备被编排系统拾取。 |
| `terminated` | 会话已终止，进入不可逆且不可用的状态。  |

- 事件可以在会话 `running` 或 `idle` 时发送。消息按顺序排队和处理。
- 智能体在收到新事件时从 `idle → running` 转换，完成后回到 `idle`。
- 错误以 `session.error` 事件出现在流中，而非作为状态值。

每个会话在 Anthropic Console 中有一个实时追踪视图，地址为 `https://platform.claude.com/workspaces/{workspace}/sessions/{session_id}`。创建会话后立即打印此 URL，以便用户可以实时观看工具调用和消息流。**`{workspace}` 是 API key 所属的工作区**，仅当那是组织的 Default 工作区时才使用 `default`。会话响应**不**包含 workspace 字段，Console 没有工作区无关的会话路由，因此对于非默认工作区，替换为工作区的 ID（在 Console URL 栏中可见，或将其作为配置值与 API key 一起暴露）。指向另一个工作区中会话的 `default` 链接会落在**"Session not found"**页面上，那里的**搜索工作区**按钮会找到它，但不是自动重定向。

### 内置会话功能

- **上下文压缩** — 如果你接近最大上下文，API 自动浓缩会话历史以保持交互继续
- **Prompt caching** — 历史重复的 token 被缓存，减少处理时间和成本
- **扩展思考** — 默认开启，以 `agent.thinking` 事件返回

### 会话操作

| 操作 | 备注 |
|---|---|
| 列表 / 获取 | 分页列表或按 ID 获取单个资源 |
| 更新 | 仅 `title` 可更新 |
| 归档 | 会话变为**只读**。不可逆。 |
| 删除 | 永久删除会话、事件历史、容器和检查点。 |

这些是运维/检查调用，通常从终端执行，而非应用代码。从 shell（参见 `shared/anthropic-cli.md`）：

```sh
ant beta:sessions list --transform '{id,title,status,created_at}' --format jsonl
ant beta:sessions retrieve --session-id "$SID"
ant beta:sessions:events stream --session-id "$SID"   # watch events live
ant beta:sessions archive  --session-id "$SID"
ant beta:sessions delete   --session-id "$SID"
```

---

## 会话

会话是环境内运行的智能体实例。

### 会话对象

API 返回的关键字段：

| 字段           | 类型     | 描述                                         |
| --------------- | -------- | --------------------------------------------------- |
| `type` | string | 始终为 `"session"` |
| `id` | string | 唯一会话 ID |
| `title` | string | 人类可读的标题 |
| `status` | string | `idle`、`running`、`rescheduling`、`terminated` |
| `created_at` | string | ISO 8601 时间戳 |
| `updated_at` | string | ISO 8601 时间戳 |
| `archived_at` | string | ISO 8601 时间戳（可空） |
| `environment_id` | string | 环境 ID |
| `agent` | object | 智能体配置 |
| `resources` | array | 附加的文件、仓库和 memory store |
| `metadata` | object | 用户提供的键值对（最多 8 个键） |
| `usage` | object | Token 使用统计 |

### 创建会话

**没有智能体的会话是毫无意义的。** 会话通过 ID 引用预创建的智能体。首先通过 `agents.create()` 创建智能体，然后引用它：

```ts
// 1. Create the agent (reusable, versioned)
const agent = await client.beta.agents.create(
  {
    name: "Coding Assistant",
    model: "claude-opus-4-8",
    system: "You are a helpful coding agent.",
    tools: [{ type: "agent_toolset_20260401"}],
  },
);

// 2. Start a session that references it
const session = await client.beta.sessions.create(
  {
    agent: agent.id,  // string shorthand → latest version. Or: { type: "agent", id: agent.id, version: agent.version }
    environment_id: environmentId,
    title: "Hello World Session",
  },
);
```

> 💡 **在 Console 中实时观看。** 开发时打印一个链接，这样你可以点击进入会话 UI，而非解析事件流：`https://platform.claude.com/workspaces/{workspace}/sessions/{session.id}`。仅当 API key 属于组织的 Default 工作区时才使用 `default` 作为 `{workspace}`，否则替换为工作区的 ID（会话响应不携带它 — 从 Console URL 栏读取或将其作为配置值）。在为本地迭代的用户生成的代码中包含此 `print`/`console.log`。

**会话创建参数：**

| 字段           | 类型     | 必需 | 描述                                    |
| --------------- | -------- | -------- | ---------------------------------------------- |
| `agent`         | string or object | **是** | 三种形式：字符串简写 `"agent_abc123"`（最新版本）；固定 `{type: "agent", id, version}`；或 `{type: "agent_with_overrides", id, version?, ...}` 仅为此会话覆盖 `model`/`system`/`tools`/`mcp_servers`/`skills` — 参见 § 为会话覆盖智能体配置 |
| `environment_id`| string   | **是**  | 环境 ID                                 |
| `title`         | string   | 否       | 人类可读的名称（出现在日志/仪表板中） |
| `resources`     | array    | 否       | 文件、GitHub 仓库或 memory store，在启动时附加到容器。Memory store 仅在创建会话时可附加（不能通过 `resources.add()` 添加）。 |
| `vault_ids`     | array    | 否       | Vault ID（`vlt_*`）— MCP 凭据带自动刷新 + `environment_variable` 密钥在出口处替换。参见 `shared/managed-agents-tools.md` → Vaults。 |
| `metadata`      | object   | 否       | 用户提供的键值对                  |

**智能体配置字段**（传给 `agents.create()`，而非 `sessions.create()`）：

| 字段         | 类型     | 必需 | 描述                                    |
| ------------- | -------- | -------- | ---------------------------------------------- |
| `name`        | string   | **是**  | 人类可读的名称（1-256 字符）              |
| `model`       | string or object | **是** | Claude 模型 ID（裸字符串，或 `{id, speed}` 对象）。支持所有 Claude 4.5+ 模型。 |
| `system`      | string   | 否       | 系统提示 — 定义智能体的行为（最多 100K 字符） |
| `tools`       | array    | 否       | 包含三种：(1) 预构建的 Claude Agent 工具（`agent_toolset_20260401`），(2) MCP 工具（`mcp_toolset`），和 (3) 自定义客户端工具。最多 128。 |
| `mcp_servers` | array    | 否       | MCP 服务器连接 — 标准化的第三方能力（例如 GitHub、Asana）。最多 20，名称唯一。参见 `shared/managed-agents-tools.md` → MCP 服务器。 |
| `skills`      | array    | 否       | 自定义的"最佳实践"上下文，带渐进式披露。最多 20。参见 `shared/managed-agents-tools.md` → 技能。 |
| `description` | string   | 否       | 智能体的描述（最多 2048 字符）    |
| `multiagent`  | object   | 否       | `{type: "coordinator", agents: [...]}` — 此智能体可委派的花名册。参见 `shared/managed-agents-multiagent.md`。 |
| `metadata`    | object   | 否       | 任意键值对（最多 16，键 ≤64 字符，值 ≤512 字符） |

---

## 智能体

**这是每个 Managed Agents 流程的开始。** 智能体对象是一个持久化的、版本化的配置 — 你创建它一次，然后每次启动会话时通过 ID 引用它。没有智能体 → 没有会话。

### 智能体对象

API 是**扁平的** — `model`、`system`、`tools` 等是顶层字段，不包装在 `agent:{}` 子对象中。

| 字段              | 类型     | 必需 | 描述                                        |
| ------------------ | -------- | -------- | -------------------------------------------------- |
| `name`             | string   | 是      | 人类可读的名称                                |
| `model`            | string   | 是      | Claude 模型 ID                                    |
| `system`           | string   | 否       | 系统提示                                      |
| `tools`            | array    | 否       | Agent 工具集 / MCP 工具集 / 自定义工具         |
| `mcp_servers`      | array    | 否       | MCP 服务器连接                             |
| `skills`           | array    | 否       | 技能引用（最多 20）                          |
| `description`      | string   | 否       | 智能体的描述                           |
| `multiagent`       | object   | 否       | 协调者花名册 — 参见 `shared/managed-agents-multiagent.md` |
| `metadata`         | object   | 否       | 任意键值对                          |

### 生命周期：创建一次，运行多次，就地更新

智能体是**持久资源**，而非每次运行的参数。预期的模式：

```
┌─ setup (once) ─────────┐     ┌─ runtime (every invocation) ─┐
│ agents.create()        │     │ sessions.create(             │
│   → store agent_id     │ ──→ │   agent={type:..., id: ID}   │
│     in config/env/db   │     │ )                            │
└────────────────────────┘     └──────────────────────────────┘
```

**反模式：** 在每次脚本运行的顶部调用 `agents.create()`。这会积累孤立的智能体对象，每次调用都支付创建延迟，并破坏版本模型。如果你看到 `agents.create()` 在一个按请求或按 cron tick 调用的函数中，那是错误的 — 将它提升到一次性设置并持久化 ID。

> **推荐 — 将 agents 和 environments 定义为 YAML + 通过 `ant` CLI 应用。** 分工是 **CLI 管控制面，SDK 管数据面**：agents 和 environments 是相对静态的资源，你用 `ant` 管理（版本控制的 YAML，从 CI 应用）；sessions 是动态的，由你的应用通过 SDK 驱动。参见 `shared/anthropic-cli.md` → *版本控制的 Managed Agents 资源* 了解 `ant beta:agents create < agent.yaml` / `update --version N` 流程。本文档其他地方展示的 SDK `agents.create()` 调用是代码中的等价物 — 在你需要以编程方式配置时使用它，但对于人类维护的内容，优先使用 YAML 流程。

### 版本控制

每次 `POST /v1/agents/{id}`（更新）创建一个新的不可变版本（数字时间戳，例如 `1772585501101368014`）。智能体的历史是只追加的，你不能编辑过去的版本。

**为何版本控制：**
- **可重现性** — 将会话固定到已知良好的配置：`{type: "agent", id, version: 3}`
- **安全迭代** — 更新智能体而不破坏已在旧版本上运行的会话
- **回滚** — 如果新系统提示退化，在调试时将新会话固定回先前版本

**`version` 是可选的。** 省略它（或使用字符串简写 `agent="agent_abc123"`）在会话创建时获取最新版本。显式传递（`{type: "agent", id, version: N}`）以固定用于可重现性。

**获取要固定的版本：** `agents.create()` 和 `agents.update()` 都在响应中返回 `version`。将其与 `agent_id` 一起存储。要获取现有智能体的当前最新版本：`GET /v1/agents/{id}` → `.version`。

**何时更新 vs 创建新的：** 当它在概念上是同一个智能体但行为有微调（更好的提示、额外工具）时更新（`POST /v1/agents/{id}`）。当它是不同的人设/目的时创建新智能体。经验法则：如果你会给它相同的 `name`，就更新。

### 智能体端点

| 操作        | 方法   | 路径                                  |
| ---------------- | -------- | ------------------------------------- |
| 创建           | `POST`   | `/v1/agents`                          |
| 列表             | `GET`    | `/v1/agents`                          |
| 获取              | `GET`    | `/v1/agents/{id}`                     |
| 更新           | `POST`   | `/v1/agents/{id}`                     |
| 归档          | `POST`   | `/v1/agents/{id}/archive`             |

> ⚠️ **归档是永久的。** 归档使智能体变为只读：现有会话继续运行，但**新会话不能引用它**，且没有取消归档。由于智能体没有 `delete`，这是终止生命周期状态。绝不要将归档生产智能体作为例行清理，先与用户确认。

### 在会话中使用智能体

通过字符串 ID（最新版本）或带显式版本的对象引用智能体：

```python
# String shorthand — uses the agent's latest version
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment_id,
)

# Or pin to a specific version (int)
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment_id,
)
```

### 为会话覆盖智能体配置

第三种 `agent` 形式 `agent_with_overrides`，替换**单个会话**的部分智能体配置 — 尝试不同的模型或授予额外工具而无需版本化智能体。传入 `id`（以及可选的 `version`；省略 = 最新，与其他两种形式相同的默认值）加上 `model`、`system`、`tools`、`mcp_servers`、`skills` 中的任意项：

```python
session = client.beta.sessions.create(
    agent={
        "type": "agent_with_overrides",
        "id": agent.id,
        "model": "claude-opus-4-8",   # replace the agent's model for this session
        "system": None,           # clear the system prompt for this session
    },
    environment_id=environment_id,
)
```

每个可覆盖字段遵循三态规则：
- **省略** → 会话从引用的智能体版本继承值。
- **`null`（或列表字段的 `[]`）** → 会话以该字段清空运行。完全适用于 `system`、`mcp_servers`、`skills`。两个例外：`model` 不可清空（`model: null` → 400 `agent_model_required`）；当会话的有效 `skills` 非空时清空 `tools` 返回 400（技能需要 `read` 工具），否则 `tools: null` / `tools: []` 清空。
- **一个值** → **完全**替换智能体的值。覆盖从不合并 — `tools` 覆盖必须列出会话应拥有的每个工具。

覆盖是会话局部的：它们**不**修改智能体资源或创建新的智能体版本。响应的 `agent` 对象反映覆盖后的配置，而其 `id` 和 `version` 仍标识基础智能体，因此你可以将会话追溯到其基础。在多智能体会话中，覆盖应用于协调者及其 `{type: "self"}` 副本；通过 ID 引用的花名册智能体始终使用其自身的原始配置（参见 `shared/managed-agents-multiagent.md`）。

### 在会话中途更新智能体配置

`sessions.update()` 可以在**现有**会话上更改 `agent.tools`、`agent.mcp_servers`（包括权限策略）和 `vault_ids`。这是**会话局部覆盖**，不创建新的智能体版本，也不传播回智能体对象。提供的数组是**完全替换**；要追加一个工具，`GET` 会话，修改，然后 `POST` 回去。会话必须处于 `idle` 状态 — 如果正在运行则先中断。

创建会话后只有 `tools` 和 `mcp_servers` 可以更改 — 要使用不同于智能体值的 `model`、`system` 或 `skills` 运行，在创建时使用 `agent_with_overrides`（上文）。智能体配置的 `system` 字段在会话生命周期内固定；你仍可以在**回合之间替换有效系统提示**，方法是发送 `system.message` 事件（参见 `shared/managed-agents-events.md` § 在会话中途更新系统提示）。

```python
client.beta.sessions.update(
    session.id,
    agent={
        "tools": [
            {"type": "agent_toolset_20260401"},
            {"type": "mcp_toolset", "mcp_server_name": "linear"},
        ],
        "mcp_servers": [{"type": "url", "name": "linear", "url": "https://mcp.linear.app/sse"}],
    },
    vault_ids=["vlt_..."],
)
```
