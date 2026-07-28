> **说明**：本文件为英文原文（`managed-agents-self-hosted-sandboxes.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

# Managed Agents — 自托管沙箱

使用 `config.type: "self_hosted"` 时，**agent 循环留在 Anthropic 的编排层**，但**工具执行移到你控制的基础设施上**。bash、文件操作和代码在你的容器内运行，因此文件系统内容和网络出口永远不会离开你的环境。与 `config.type: "cloud"`（Anthropic 运行容器）形成对比。连接是**仅出站**的：你的 worker 长轮询 Anthropic 的工作队列，Anthropic 永远不会拨入你的网络。

## 流程

```
1. 创建环境：             config: {type: "self_hosted"}        → env_...
2. 生成环境密钥（Console，环境页面上）                          → sk-ant-oat01-...  作为 ANTHROPIC_ENVIRONMENT_KEY
3. 运行 worker：          EnvironmentWorker.run()  或  ant beta:worker poll
4. 会话引用                environment_id=env_...  与 cloud 完全相同
```

## 创建环境

```python
client = anthropic.Anthropic()

environment = client.beta.environments.create(
    name="self-hosted", config={"type": "self_hosted"}
)
```

`{"type": "self_hosted"}` 是整个配置，没有 pool、capacity 或 networking 子字段，这些由你在自己侧控制。

## 运行 worker — SDK（主要路径）

`EnvironmentWorker` 封装了轮询 -> 分发 -> 工具执行的循环。`.run()` 是常驻循环；`.run_one()` / `.runOne()` 处理一个工作项（用于 webhook 驱动唤醒）。

**Python — 常驻：**

```python
import asyncio
import os
from anthropic import AsyncAnthropic
from anthropic.lib.environments import EnvironmentWorker


async def main() -> None:
    environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
    environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
    async with AsyncAnthropic(auth_token=environment_key) as client:
        await EnvironmentWorker(
            client,
            environment_id=environment_id,
            environment_key=environment_key,
            workdir="/workspace",
        ).run()


asyncio.run(main())
```

**TypeScript — 常驻：**

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { EnvironmentWorker } from "@anthropic-ai/sdk/helpers/beta/environments";

const environmentKey = process.env.ANTHROPIC_ENVIRONMENT_KEY!;
const environmentId = process.env.ANTHROPIC_ENVIRONMENT_ID!;
const client = new Anthropic({ authToken: environmentKey });
const ctrl = new AbortController();
process.once("SIGTERM", () => ctrl.abort());

await new EnvironmentWorker({
  client,
  environmentId,
  environmentKey,
  workdir: "/workspace",
  signal: ctrl.signal
}).run();
```

**自定义工具。** `EnvironmentWorker` 默认运行内置工具集。要添加或替换工具，使用 `AgentToolContext(workdir=, client=, session_id=)` 配合 `beta_agent_toolset(env)` / `betaAgentToolset(env)`，将生成的工具传给底层的 `tool_runner()`。附加到 agent 的 skills 在工具调用开始前下载到 `{workdir}/skills/<name>/`（`AgentToolContext` 在给定 `client` 和 `session_id` 时处理此事）。下载的 skill 文件由 CLI 和 SDK 自动标记为可执行；如果你自己实现 skills 下载，需自行设置权限。

> **运行时依赖：** SDK helper 要求 `/bin/bash` 在该确切路径。TypeScript SDK 额外要求 `unzip`、`tar` 和 Node.js 22+。这些在固定路径解析，**不**尊重 `PATH` 覆盖。

## 运行 worker — `ant` CLI（固定工具集）

`ant` CLI 提供带固定内置工具集（`bash`、`read`、`write`、`edit`、`glob`、`grep`）的 worker。按 `shared/anthropic-cli.md` 安装，然后：

```sh
export ANTHROPIC_ENVIRONMENT_KEY=sk-ant-oat01-...
ant beta:worker poll --environment-id env_... --workdir /workspace
```

- `--workdir` 是工具操作的目录（默认 `.`）；工具调用被沙箱化到该目录。
- `--environment-key` 覆盖环境变量。
- `--on-work <script>` 每个工作项运行你的脚本（例如为每个会话启动新容器，见下方容器编排）。
- `--unrestricted-paths`、`--max-idle`（默认 `60s`）、`--log-format`，参见 `ant beta:worker poll --help`。
- 标志回退到环境变量（`ANTHROPIC_ENVIRONMENT_ID`、`ANTHROPIC_ENVIRONMENT_KEY`）。
- 在 SIGTERM/SIGINT 上排干进行中的工作后干净退出。
- **固定工具集**，自定义工具请使用上方的 SDK worker。

在 `--on-work` 容器内，运行 `ant beta:worker run --workdir <dir>` 作为入口点。

## Webhook 驱动唤醒（替代常驻）

为 `session.status_run_started` 注册 webhook（参见 `shared/managed-agents-webhooks.md`），验证投递，然后用 `.run_one()` 排干一个工作项：

```python
import os
import anthropic
from anthropic.lib.environments import EnvironmentWorker

environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
client = anthropic.AsyncAnthropic(
    auth_token=environment_key,
)  # 从环境变量读取 ANTHROPIC_WEBHOOK_SIGNING_KEY 用于 webhooks.unwrap()


async def handle(raw: bytes, headers: dict[str, str]) -> dict:
    event = client.beta.webhooks.unwrap(raw.decode(), headers=headers)
    if event.data.type != "session.status_run_started":
        return {"status": "ignored"}
    await EnvironmentWorker(
        client,
        environment_id=environment_id,
        environment_key=environment_key,
        workdir="/workspace",
    ).run_one()
    return {"status": "ok"}
```

TypeScript：相同结构，使用 `client.beta.webhooks.unwrap(body, {headers})` 和 `new EnvironmentWorker({...}).runOne()`。

## 容器编排（中间层）

`EnvironmentWorker.run()` 在同一进程中轮询和执行工具。要让每个会话在**自己的**容器中运行，在轻量编排器中使用中间层轮询器。Python `client.beta.environments.work.poller(environment_id=, environment_key=, drain=, block_ms=, reclaim_older_than_ms=, auto_stop=)`；TypeScript 从 `@anthropic-ai/sdk/helpers/beta/environments` 导入 `new WorkPoller({client, environmentId, environmentKey, autoStop})`。对于每个产出的 `work` 项，启动一个注入了这些环境变量的新容器，其入口点运行 `ant beta:worker run` 或 `EnvironmentWorker(...).run_one()`。`block_ms` 为 1-999（或 `None` 表示非阻塞）；`reclaim_older_than_ms` 重新认领分配给已死 worker 的项；`drain` 在队列为空时停止；`auto_stop` 在迭代器退出后发送停止信号（当启动的容器拥有停止调用时设为 `False`）。**Go 的轮询器没有 `auto_stop` 退出选项**，它在 handler 返回时调用 `work.Stop`，因此在 handler 中阻塞直到会话完成，而非分离。

| 环境变量 | 值 |
|---|---|
| `ANTHROPIC_SESSION_ID` | `work.data.id` |
| `ANTHROPIC_WORK_ID` | `work.id` |
| `ANTHROPIC_ENVIRONMENT_ID` | `work.environment_id` |
| `ANTHROPIC_ENVIRONMENT_KEY` | 透传 |
| `ANTHROPIC_BASE_URL` | 透传 |

跳过 `work.data.type != "session"` 的项。

## 监控和控制

这些是**控制面**调用，使用 `x-api-key`（非环境密钥）认证；带 `managed-agents-2026-04-01` beta header。**从 worker 宿主外部调用**，在 worker 宿主上设置 `ANTHROPIC_API_KEY` 会将组织作用域凭证暴露给 agent 工具调用。

| SDK (`client.beta.environments.work.*`) | REST | CLI | 返回值 |
|---|---|---|---|
| `stats(environment_id)` | `GET /v1/environments/{id}/work/stats` | `ant beta:environments:work stats` | `{type:"work_queue_stats", depth, pending, oldest_queued_at, workers_polling}` |
| `stop(work_id, environment_id=)` | `POST /v1/environments/{id}/work/{work_id}/stop` | `ant beta:environments:work stop` | `work.state` |

## 与 `cloud` 相比的变化

| 关注点 | `cloud` | `self_hosted` |
|---|---|---|
| 容器生命周期、加固、网络 | Anthropic | **你**，以非 root 运行、只读 rootfs、drop caps；出口由你的 VPC/防火墙决定 |
| `file` / `github_repository` 资源挂载 | Anthropic 挂载到容器中 | **你**，通过 `sessions.create(metadata={...})` 传递指针，编排器在分发前获取/克隆 |
| `memory_store` 资源 | 支持 | **尚不支持** |
| Vault `environment_variable` 凭证 | 支持（在 Anthropic 管理的出口处替换） | **尚不支持**，出口是你自己的，没有地方替换密钥。使用 MCP 凭证或宿主侧自定义工具（`shared/managed-agents-client-patterns.md` Pattern 9） |
| 内置工具 | 通过 `agent_toolset_20260401` | 由你的 worker 提供（`EnvironmentWorker` 默认 / `beta_agent_toolset(env)` / `ant` CLI 固定集） |
| Skills 下载 | 自动 | `EnvironmentWorker` / `AgentToolContext` 获取到 `{workdir}/skills/`（需要 `client` + `session_id`） |
| Claude Platform on AWS | 支持 | **不可用** |
| SDK worker helper | 所有 SDK | **仅 Python、TypeScript、Go**（`EnvironmentWorker` / 轮询器不在 Java、Ruby、PHP 或 C# 中），使用这三种之一或 `ant` CLI |

## 凭证

| 凭证 | 格式 | 作用域 |
|---|---|---|
| `ANTHROPIC_ENVIRONMENT_KEY` | `sk-ant-oat01-...` | 一个环境的工作队列。在 Console 中生成（"Generate environment key"）。作为 client 上的 `auth_token=` / `authToken` **以及** `EnvironmentWorker` 上的 `environment_key=` / `environmentKey` 传递。存储在密钥管理器中；泄露时轮换。 |
| `ANTHROPIC_WEBHOOK_SIGNING_KEY` | `whsec_...` | Webhook 签名验证（如果使用 webhook 驱动唤醒）。SDK 自动读取此环境变量用于 `client.beta.webhooks.unwrap()`。 |

## 安全 — 你负责什么

容器加固；出口限制（没有默认限制）；`ANTHROPIC_ENVIRONMENT_KEY` 保管和轮换；运行不受信任的代码时每个信任边界一个 workspace + 环境；工具进程的最小权限；日志保留和修订。**Anthropic 无法**：快速撤销泄露的环境密钥、验证你的镜像或供应链、在你的容器内沙箱化工具执行、或在工具输出到达你的基础设施后强制保留。完整清单请参见 `shared/live-sources.md` 中的 Self-Hosted Sandboxes Security 页面。
