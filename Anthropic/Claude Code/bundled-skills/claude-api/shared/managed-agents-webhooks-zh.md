# 托管智能体 — Webhooks

当托管智能体资源状态变更时，Anthropic 可以向你的 HTTPS 端点发送 POST 请求——这是保持 SSE 流或轮询的替代方案。载荷是**精简的**（仅事件类型 + 资源 ID）；收到后，获取资源以了解当前状态。每次投递都经过 HMAC 签名。

> **方向很重要。** 本页涵盖 *Anthropic → 你* 关于会话/vault 状态的通知。**不**涵盖 *第三方 → 你* 的 webhook（即*触发*会话的 webhook，例如调用 `sessions.create()` 的 GitHub push 处理器）——那是你这边的普通应用代码，没有 Anthropic 特定的传输格式。

---

## 注册端点（仅限 Console）

Console → **Manage → Webhooks**。目前没有编程方式的端点管理 API。同一页面支持密钥轮换。

| 字段 | 约束 |
|---|---|
| URL | HTTPS 端口 443，公开可解析的主机名 |
| 事件类型 | 按 `data.type` 订阅——只接收已订阅类型（以及测试事件） |
| 签名密钥 | `whsec_` 前缀，32 字节，**创建时仅显示一次**——请妥善保存 |

---

## 验证签名

每次投递都经过 HMAC 签名。**使用 SDK 的 `client.beta.webhooks.unwrap()`**——它验证签名、拒绝超过约 5 分钟的载荷，并返回解析后的事件。它从 `ANTHROPIC_WEBHOOK_SIGNING_KEY` 读取 `whsec_` 密钥。

```python
import anthropic
from flask import Flask, request

client = anthropic.Anthropic()  # 从环境变量读取 ANTHROPIC_WEBHOOK_SIGNING_KEY
app = Flask(__name__)


@app.route("/webhook", methods=["POST"])
def webhook():
    try:
        event = client.beta.webhooks.unwrap(
            request.get_data(as_text=True),
            headers=dict(request.headers),
        )
    except Exception:
        return "invalid signature", 400

    if event.id in seen_event_ids:  # 去重重试——id 是按事件的，不是按投递的
        return "", 204
    seen_event_ids.add(event.id)

    match event.data.type:
        case "session.status_idled":
            session = client.beta.sessions.retrieve(event.data.id)
            notify_user(session)
        case "vault_credential.refresh_failed":
            alert_oncall(event.data.id)

    return "", 204
```

将**原始请求体**传给 `unwrap()`——重新序列化 JSON 的框架（Express `.json()`、Flask `.get_json()`）会改变字节并破坏 MAC。其他语言请查阅 SDK 仓库中的 `beta.webhooks.unwrap` 绑定（`shared/live-sources.md`）；不要手写验证。

---

## 载荷信封

```json
{
  "type": "event",
  "id": "event_01ABC...",
  "created_at": "2026-03-18T14:05:22Z",
  "data": {
    "type": "session.status_idled",
    "id": "session_01XYZ...",
    "organization_id": "8a3d2f1e-...",
    "workspace_id": "c7b0e4d9-..."
  }
}
```

根据 `data.type` 分发，按 `data.id` 获取资源，返回任意 **2xx** 确认。`created_at` 是*状态转换*发生的时间，不是 webhook 触发的时间。

---

## 支持的 `data.type` 值

| `data.type` | 触发条件 |
|---|---|
| `session.status_scheduled` | 会话创建完成，可接受事件 |
| `session.status_run_started` | 智能体执行启动（每次转为 `running`） |
| `session.status_idled` | 智能体等待输入（工具审批、自定义工具结果或下一条消息） |
| `session.status_terminated` | 会话遇到终止性错误 |
| `session.thread_created` | 多智能体：协调器打开了新的子智能体线程 |
| `session.thread_idled` | 多智能体：子智能体线程等待输入 |
| `session.outcome_evaluation_ended` | 结果评分器完成一次迭代 |
| `vault.archived` | Vault 已归档 |
| `vault.created` | Vault 已创建 |
| `vault.deleted` | Vault 已删除 |
| `vault_credential.archived` | Vault 凭据已归档 |
| `vault_credential.created` | Vault 凭据已创建 |
| `vault_credential.deleted` | Vault 凭据已删除 |
| `vault_credential.refresh_failed` | MCP OAuth vault 凭据刷新失败 |
| `agent.created` | 智能体已创建 |
| `agent.updated` | 新智能体版本已发布。不创建新版本的更新**不**触发此事件。 |
| `agent.archived` | 智能体已归档 |
| `agent.deleted` | 智能体已永久删除——无对象可获取；将事件本身视为最终状态 |
| `deployment.created` | 计划部署已创建 |
| `deployment.updated` | 部署属性已变更（如计划已编辑） |
| `deployment.paused` | 部署已暂停——按请求暂停，或当计划运行因**不可恢复**错误（归档的智能体、缺失的环境）自动暂停。可恢复的故障（包括限流）**不**自动暂停。 |
| `deployment.unpaused` | 部署已取消暂停；计划恢复 |
| `deployment.archived` | 部署已归档——直接归档，或因智能体归档/删除而归档 |
| `deployment.deleted` | 部署已永久删除——无对象可获取；将事件本身视为最终状态 |
| `deployment_run.started` | **计划**运行已启动。手动运行**不**发出 `deployment_run.*` 事件。 |
| `deployment_run.succeeded` | 计划运行已创建其会话。与运行的 `.started` 事件相同的 `data.id`（运行 ID）——获取部署运行以得到其 `session_id`，然后订阅会话事件以跟踪工作。 |
| `deployment_run.failed` | 计划运行未创建会话。与运行的 `.started` 事件相同的 `data.id`——获取部署运行以得到 `error.type` / `error.message`。 |

> 这些是 **webhook** 的 `data.type` 值——与 SSE 事件类型（`session.status_idle`、`span.outcome_evaluation_end` 等，见 `shared/managed-agents-events.md`）是不同的命名空间。不要在 webhook 处理器中复用 SSE 常量。

---

## 投递行为与陷阱

- **无顺序保证。** `session.status_idled` 可能在 `session.outcome_evaluation_ended` 之前到达，即使评估先完成。如果顺序重要，按信封的 `created_at` 排序。
- **重试携带相同的 `event.id`。** 非 2xx 至少重试一次。按 `event.id` 去重。
- **3xx 视为失败。** 不跟随重定向——如果端点迁移，请在 Console 中更新 URL。
- **自动禁用**：连续约 20 次投递失败后，或主机名解析为私有 IP 或返回重定向时立即禁用。需在 Console 中手动重新启用。
- **精简载荷是有意为之。** 不要期望 webhook 体中有 `stop_reason`、`outcome_evaluations`、凭据密钥等——请获取资源。
