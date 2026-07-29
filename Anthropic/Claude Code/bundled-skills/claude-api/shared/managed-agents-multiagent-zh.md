# 托管智能体 — 多智能体会话

协调器智能体可以在一个会话内委派给其他智能体。所有智能体**共享容器和文件系统**；每个智能体运行在自己的**线程**中——一个上下文隔离的事件流，拥有自己的对话历史、模型、系统提示词、工具、MCP 服务器和技能（来自该智能体自身的配置）。线程是持久的：协调器可以向它之前调用的子智能体发送后续消息，该子智能体会保留之前的对话轮次。

SDK 在所有 `client.beta.{agents,sessions}.*` 调用上自动设置 `managed-agents-2026-04-01` beta 头；多智能体无需额外头部。

---

## 在协调器上声明名册

`multiagent` 是 `agents.create()` / `agents.update()` 上的一个**顶层字段**——**不是** `tools[]` 条目。`agents` 列出 1-20 个名册条目。`sessions.create()` 上无需任何更改——名册从协调器的配置中解析。

```python
orchestrator = client.beta.agents.create(
    name="Engineering Lead",
    model="claude-opus-4-8",
    system="You coordinate engineering work. Delegate code review to the reviewer and test writing to the test agent.",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={
        "type": "coordinator",
        "agents": [
            reviewer.id,                                            # 纯字符串 — 最新版本
            {"type": "agent", "id": test_writer.id, "version": 4},  # 固定版本
            {"type": "self"},                                       # 协调器自身
        ],
    },
)

session = client.beta.sessions.create(agent=orchestrator.id, environment_id=env.id)
```

| 名册条目 | 形式 | 说明 |
|---|---|---|
| 字符串简写 | `"agent_abc123"` | 引用已存储智能体的最新版本。 |
| 智能体引用 | `{type: "agent", id, version?}` | 省略 `version` 则在协调器保存时固定为最新版本。 |
| 自身 | `{type: "self"}` | 协调器可以生成自身的副本。 |

如果会话是通过 `agent_with_overrides` 创建的（参见 `shared/managed-agents-core.md` → 为会话覆盖智能体配置），这些覆盖会应用到**协调器及其 `self` 副本**。通过 ID 引用的名册智能体始终使用其创建时的配置——覆盖不会传播到它们。

名册中最多 **20 个唯一智能体**；协调器可以生成每个智能体的**多个副本**。**仅支持一级委派**——深度大于 1 的会被忽略。

---

## 线程

会话级事件流是**主线程**——它显示协调器的追踪以及子智能体活动的精简视图（线程状态转换和跨线程消息，而非每个子智能体的工具调用）。通过每线程端点深入查看特定子智能体：

| 操作 | HTTP | SDK (`client.beta.sessions.threads.*`) |
|---|---|---|
| 列出线程 | `GET /v1/sessions/{sid}/threads` | `.list(session_id)` |
| 检索单个 | `GET /v1/sessions/{sid}/threads/{tid}` | `.retrieve(thread_id, session_id=...)` |
| 归档 | `POST /v1/sessions/{sid}/threads/{tid}/archive` | `.archive(thread_id, session_id=...)` |
| 列出线程事件 | `GET /v1/sessions/{sid}/threads/{tid}/events` | `.events.list(thread_id, session_id=...)` |
| 流式传输线程事件 | `GET /v1/sessions/{sid}/threads/{tid}/stream` | `.events.stream(thread_id, session_id=...)` |

每个 `SessionThread` 包含 `id`、`status`（`running` | `idle` | `rescheduling` | `terminated`）、`agent`（智能体配置的已解析快照——`id`、`name`、`model`、`system`、`tools`、`skills`、`mcp_servers`、`version`）、`parent_thread_id`（主线程为 null，主线程也包含在列表中）、`archived_at` 以及可选的 `stats`/`usage`。**会话状态聚合线程状态**——如果有任何线程处于 `running` 状态，`session.status` 就是 `running`。最多 **25 个并发线程**。在排空每线程流时，遇到 `session.thread_status_idle` 时中断（并像检查会话级 idle 那样检查其 `stop_reason`）。

---

## 多智能体事件（在会话流上）

| 事件 | 载荷要点 | 含义 |
|---|---|---|
| `session.thread_created` | `session_thread_id`, `agent_name` | 创建了新线程。 |
| `session.thread_status_running` | `session_thread_id`, `agent_name` | 线程开始活动。 |
| `session.thread_status_idle` | `session_thread_id`, `agent_name`, **`stop_reason`** | 线程等待输入。检查 `stop_reason`（与 `session.status_idle.stop_reason` 形式相同）。 |
| `session.thread_status_rescheduled` | `session_thread_id`, `agent_name` | 线程在可重试错误后重新调度。 |
| `session.thread_status_terminated` | `session_thread_id`, `agent_name` | 线程被归档或遇到终止错误。 |
| `agent.thread_message_sent` | `to_session_thread_id`, `to_agent_name`, `content` | 协调器向另一个线程发送了后续消息。 |
| `agent.thread_message_received` | `from_session_thread_id`, `from_agent_name`, `content` | 智能体将其结果传递给了协调器。 |

---

## 工具权限和来自子智能体线程的自定义工具

当子智能体需要你的客户端（一个 `always_ask` 确认，或一个自定义工具结果）时，请求会**跨投递到主线程**，其中 `session_thread_id` 标识发起线程——因此你只需监视会话流。用 `user.tool_confirmation`（携带 `tool_use_id`）或 `user.custom_tool_result`（携带 `custom_tool_use_id`）回复，并**回显发起事件中的 `session_thread_id`**（SDK 参数类型和文档字符串期望这样做）。服务器也会通过工具使用 ID 路由，所以回显是双保险而非关键——但请包含它。

```python
for event_id in stop.event_ids:
    pending = events_by_id[event_id]
    confirmation = {
        "type": "user.tool_confirmation",
        "tool_use_id": event_id,
        "result": "allow",
    }
    if pending.session_thread_id is not None:
        confirmation["session_thread_id"] = pending.session_thread_id
    client.beta.sessions.events.send(session.id, events=[confirmation])
```

同样的模式适用于 `user.custom_tool_result`。

---

## 陷阱

- **不要将名册放在 `sessions.create()` 或 `tools[]` 上。** `multiagent` 是一个顶层智能体字段；更新协调器，然后启动引用它的会话。
- **不要假设共享上下文。** 线程共享文件系统但不共享对话历史或工具。如果协调器需要子智能体处理某些内容，它必须在委派消息中说明（或写入磁盘）。
- **深度大于 1 的会被忽略。** 子智能体自身的 `multiagent` 名册（如果有）不会级联——只有会话的协调器进行委派。

如需 Python 以外的语言绑定，WebFetch `https://platform.claude.com/docs/en/managed-agents/multi-agent.md`（参见 `shared/live-sources.md`）。
