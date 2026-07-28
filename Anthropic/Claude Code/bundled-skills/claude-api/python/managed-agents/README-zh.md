# Managed Agents — Python

> **未展示的绑定：** 本 README 涵盖 Python 最常见的 managed-agents 流程。如果需要未展示的类、方法、命名空间、字段或行为，请从 `shared/live-sources.md` 中 WebFetch Python SDK 仓库**或相关文档页面**，而不是猜测。不要从 cURL 形状或其他语言的 SDK 推断。

> **Agent 是持久的，创建一次，按 ID 引用。** 存储 `agents.create` 返回的 agent ID，传给后续每次 `sessions.create`；不要在请求路径中调用 `agents.create`。**推荐做法：** 将 agent 和 environment 定义为版本控制的 YAML，通过 `ant` CLI 应用，参见 `shared/anthropic-cli.md`（其在线文档 URL 在 `shared/live-sources.md` 中）。CLI 拥有控制面（创建/更新），你的代码拥有数据面（用存储的 ID 创建会话）。以下示例展示的是必须编程式配置时的代码内创建；在生产环境中，创建调用应放在设置阶段，不在请求路径中。

## 安装

```bash
pip install anthropic
```

## 客户端初始化

```python
import anthropic

# 默认方式 — 从环境变量解析凭证：
# ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN 或 `ant auth login` 配置。
# 本地开发优先使用此方式；不要硬编码 key。
client = anthropic.Anthropic()

# 显式 API key（仅在需要注入特定 key 时使用）
client = anthropic.Anthropic(api_key="your-api-key")
```

---

## 创建环境

```python
environment = client.beta.environments.create(
    name="my-dev-env",
    config={
        "type": "cloud",
        "networking": {"type": "unrestricted"},
    },
)
print(environment.id)  # env_...
```

---

## 创建 Agent（必需的第一步）

> ⚠️ **没有内联 agent 配置。** `model`/`system`/`tools` 位于 agent 对象上，不在 session 上。始终从 `agents.create()` 开始，session 只接受 `agent={"type": "agent", "id": agent.id}`。

### 最小配置

```python
# 1. 创建 agent（可复用、有版本）
agent = client.beta.agents.create(
    name="Coding Assistant",
    model="claude-opus-4-8",
    tools=[{"type": "agent_toolset_20260401", "default_config": {"enabled": True}}],
)

# 2. 启动会话
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment.id,
)
print(session.id, session.status)
print(f"Trace: https://platform.claude.com/workspaces/default/sessions/{session.id}")  # 如果 API key 不在 Default workspace 中，将 'default' 替换为你的 workspace ID
```

### 带系统提示和自定义工具

```python
import os

agent = client.beta.agents.create(
    name="Code Reviewer",
    model="claude-opus-4-8",
    system="You are a senior code reviewer.",
    tools=[
        {"type": "agent_toolset_20260401"},
        {
            "type": "custom",
            "name": "run_tests",
            "description": "Run the test suite",
            "input_schema": {
                "type": "object",
                "properties": {
                    "test_path": {"type": "string", "description": "Path to test file"}
                },
                "required": ["test_path"],
            },
        },
    ],
)

session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment.id,
    title="Code review session",
    resources=[
        {
            "type": "github_repository",
            "url": "https://github.com/owner/repo",
            "mount_path": "/workspace/repo",
            "authorization_token": os.environ["GITHUB_TOKEN"],
            "branch": "main",
        }
    ],
)
```

---

## 发送用户消息

```python
client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.message",
            "content": [{"type": "text", "text": "Review the auth module"}],
        }
    ],
)
```

> 💡 **先开流：** 在发送消息*之前*（或同时）打开流。流只传递打开后发生的事件，先发后开会导致早期事件以一批缓冲数据到达。参见 [Steering Patterns](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式事件（SSE）

```python
import json

# 先开流：打开流，然后在流活跃时发送
with client.beta.sessions.events.stream(
    session_id=session.id,
) as stream:
    client.beta.sessions.events.send(
        session_id=session.id,
        events=[{"type": "user.message", "content": [{"type": "text", "text": "..."}]}],
    )
    for event in stream:
        ...  # 处理事件

# 独立流迭代：
with client.beta.sessions.events.stream(
    session_id=session.id,
) as stream:
    for event in stream:
        if event.type == "agent.message":
            for block in event.content:
                if block.type == "text":
                    print(block.text, end="", flush=True)
        elif event.type == "agent.custom_tool_use":
            # 自定义工具调用 — 会话现在空闲
            print(f"\nCustom tool call: {event.name}")
            print(f"Input: {json.dumps(event.input)}")
            # 发送结果回去（见下方）
        elif event.type == "session.status_idle":
            print("\n--- Agent idle ---")
        elif event.type == "session.status_terminated":
            print("\n--- Session terminated ---")
            break
```

---

## 提供自定义工具结果

```python
client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.custom_tool_result",
            "custom_tool_use_id": "sevt_abc123",
            "content": [{"type": "text", "text": "All 42 tests passed."}],
        }
    ],
)
```

---

## 轮询事件

```python
events = client.beta.sessions.events.list(
    session_id=session.id,
)
for event in events.data:
    print(f"{event.type}: {event.id}")
```

> ⚠️ **优先使用 SDK 而非原始 `requests`/`httpx`。** 如果你手写轮询循环，不要假设 `timeout=(5, 60)` 或 `httpx.Timeout(120)` 限制了总调用时长，两者都是**按块**读取超时（每收到一个字节就重置），因此缓慢滴流的响应可以永远阻塞。如需硬性墙钟截止时间，在循环级别跟踪 `time.monotonic()` 并显式退出，或用 `asyncio.wait_for()` 包装。参见 [Receiving Events](../../shared/managed-agents-events.md#receiving-events)。

---

## 带自定义工具的完整流式循环

```python
import json


def run_custom_tool(tool_name: str, tool_input: dict) -> str:
    """执行自定义工具并返回结果。"""
    if tool_name == "run_tests":
        # 你的工具实现
        return "All tests passed."
    return f"Unknown tool: {tool_name}"


def run_session(client, session_id: str):
    """流式处理事件并处理自定义工具调用。"""
    while True:
        with client.beta.sessions.events.stream(
            session_id=session_id,
        ) as stream:
            tool_calls = []
            for event in stream:
                if event.type == "agent.message":
                    for block in event.content:
                        if block.type == "text":
                            print(block.text, end="", flush=True)
                elif event.type == "agent.custom_tool_use":
                    tool_calls.append(event)
                elif event.type == "session.status_idle":
                    break
                elif event.type == "session.status_terminated":
                    return

        if not tool_calls:
            break

        # 处理自定义工具调用
        results = []
        for call in tool_calls:
            result = run_custom_tool(call.name, call.input)
            results.append({
                "type": "user.custom_tool_result",
                "custom_tool_use_id": call.id,
                "content": [{"type": "text", "text": result}],
            })

        client.beta.sessions.events.send(
            session_id=session_id,
            events=results,
        )
```

---

## 上传文件

```python
with open("data.csv", "rb") as f:
    file = client.beta.files.upload(
        file=f,
    )

# 在会话中使用
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment.id,
    resources=[{"type": "file", "file_id": file.id, "mount_path": "/workspace/data.csv"}],
)
```

---

## 列出和下载会话文件

列出 agent 在会话期间写入 `/mnt/session/outputs/` 的文件，然后下载。

```python
# 列出与会话关联的文件
files = client.beta.files.list(
    scope_id=session.id,
    betas=["managed-agents-2026-04-01"],
)
for f in files.data:
    print(f.filename, f.size_bytes)
    # 下载每个文件并保存到磁盘
    file_content = client.beta.files.download(f.id)
    file_content.write_to_file(f.filename)
```

> 💡 在 `session.status_idle` 和输出文件出现在 `files.list` 之间有短暂的索引延迟（约 1-3 秒）。如果列表为空，重试一两次。

---

## 会话管理

```python
# 获取会话详情
session = client.beta.sessions.retrieve(session_id="sesn_011CZxAbc123Def456")
print(session.status, session.usage)

# 列出会话
sessions = client.beta.sessions.list()

# 删除会话
client.beta.sessions.delete(session_id="sesn_011CZxAbc123Def456")

# 归档会话
client.beta.sessions.archive(session_id="sesn_011CZxAbc123Def456")
```

---

## MCP 服务器集成

```python
# Agent 声明 MCP 服务器（此处无认证 — 认证放在 vault 中）
agent = client.beta.agents.create(
    name="MCP Agent",
    model="claude-opus-4-8",
    mcp_servers=[
        {"type": "url", "name": "my-tools", "url": "https://my-mcp-server.example.com/sse"},
    ],
    tools=[
        {"type": "agent_toolset_20260401", "default_config": {"enabled": True}},
        {"type": "mcp_toolset", "mcp_server_name": "my-tools"},
    ],
)

# 会话挂载包含这些 MCP 服务器 URL 凭证的 vault
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    vault_ids=[vault.id],
)
```

创建 vault 和添加凭证请参见 `shared/managed-agents-tools.md` §Vaults。
