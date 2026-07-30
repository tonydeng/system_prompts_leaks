> **说明**：本文件为英文原文（`managed-agents-outcomes.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 托管智能体 — 结果（Outcomes）

**结果（outcome）** 将会话从*对话*提升为*工作*：你声明"完成"的标准，框架运行 迭代 → 评分 → 修订 循环，直到产物满足评分标准、达到 `max_iterations` 上限，或被中断。一个独立的**评分器**（独立上下文窗口）根据你的评分标准对每次迭代打分，并将各标准的差距反馈给智能体。

SDK 在所有 `client.beta.sessions.*` 调用上自动设置 `managed-agents-2026-04-01` beta 头；结果功能无需额外头部。

---

## `user.define_outcome` 事件

结果不是 `sessions.create()` 的字段。你先创建一个普通会话，然后发送 `user.define_outcome` 事件。智能体收到后即开始工作——**不要再发送 `user.message`** 来启动它。

```python
session = client.beta.sessions.create(
    agent=AGENT_ID,
    environment_id=ENVIRONMENT_ID,
    title="Financial analysis on Costco",
)

client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.define_outcome",
            "description": "Build a DCF model for Costco in .xlsx",
            "rubric": {"type": "text", "content": RUBRIC_MD},
            # or: "rubric": {"type": "file", "file_id": rubric.id}
            "max_iterations": 5,  # optional; default 3, max 20
        }
    ],
)
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `type` | `"user.define_outcome"` | |
| `description` | string | 任务描述。智能体朝此目标工作——无需单独的 `user.message`。 |
| `rubric` | `{type: "text", content}` \| `{type: "file", file_id}` | **必填。** Markdown 格式，包含明确的、可独立评分的标准。通过 `client.beta.files.upload(...)` 上传一次（beta `files-api-2025-04-14`），即可在多个会话中复用。 |
| `max_iterations` | int | 可选。默认 **3**，最大 **20**。 |

该事件会在流上回显，附带服务器分配的 `outcome_id` 和 `processed_at`。

> **编写评分标准。** 使用明确的、可评分的标准（如"CSV 有一个数值类型的 `price` 列"），而非模糊描述（如"数据看起来不错"）——评分器会独立评估每条标准，模糊标准会产生噪声循环。如果你没有评分标准，可以让 Claude 分析一个已知合格的产物，将其分析转化为评分标准。

---

## 结果特有事件

这些事件出现在标准事件流（`sessions.events.stream` / `.list`）上，与常规的 `agent.*` / `session.*` 事件并存。

| 事件 | 载荷要点 | 含义 |
|---|---|---|
| `span.outcome_evaluation_start` | `outcome_id`, `iteration`（从 0 开始） | 评分器开始对第 *N* 次迭代评分。 |
| `span.outcome_evaluation_ongoing` | `outcome_id` | 评分器运行期间的心跳。评分器的推理过程不可见——你只能看到它*正在工作*，看不到*在想什么*。 |
| `span.outcome_evaluation_end` | `outcome_evaluation_start_id`, `outcome_id`, `iteration`, `result`, `explanation`, `usage` | 评分器完成一次迭代。`result` 决定下一步（见下表）。 |

### `span.outcome_evaluation_end.result`

| `result` | 下一步 |
|---|---|
| `satisfied` | 会话 → `idle`。此结果终止。 |
| `needs_revision` | 智能体开始下一次迭代。 |
| `max_iterations_reached` | 不再有评分循环。智能体可执行最后一次修订，然后会话 → `idle`。 |
| `failed` | 会话 → `idle`。评分标准与任务根本不匹配（如描述与标准矛盾）。 |
| `interrupted` | 仅在 `user.interrupt` 到达前 `_start` 已触发时才发出。 |

```json
{
  "type": "span.outcome_evaluation_end",
  "id": "sevt_01jkl...",
  "outcome_evaluation_start_id": "sevt_01def...",
  "outcome_id": "outc_01a...",
  "result": "satisfied",
  "explanation": "All 12 criteria met: revenue projections use 5 years of historical data, ...",
  "iteration": 0,
  "usage": { "input_tokens": 2400, "output_tokens": 350, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 1800 },
  "processed_at": "2026-03-25T14:03:00Z"
}
```

---

## 检查状态与获取交付物

**状态**——在流上监听 `span.outcome_evaluation_end`，或轮询会话并读取 `outcome_evaluations`：

```python
session = client.beta.sessions.retrieve(session.id)
for ev in session.outcome_evaluations:
    print(f"{ev.outcome_id}: {ev.result}")  # outc_01a...: satisfied
```

**交付物**——智能体写入 `/mnt/session/outputs/`。会话 idle 后，通过 Files API 用 `scope_id=session.id` 获取。这与 `shared/managed-agents-environments.md` → Session outputs 中记录的会话输出机制相同（包括 `files.list` 上需要双重 beta 头的要求）。

---

## 交互规则与陷阱

- **同一时间只有一个结果。** 链式调用时，在前一个结果的终止事件 `span.outcome_evaluation_end`（`satisfied` / `max_iterations_reached` / `failed` / `interrupted`）之后，才发送下一个 `user.define_outcome`。会话在链式结果之间保留历史记录。
- **引导允许但非必需。** 你*可以*在结果执行过程中发送 `user.message` 事件来引导方向，但智能体已经知道要持续工作直到终止——不要发送"继续"提示。
- **`user.interrupt` 暂停当前结果**——它会标记 `result: "interrupted"` 并将会话置为 `idle`，准备接受新结果或对话轮次。
- **终止后会话可复用**——继续对话或定义新结果。
- **结果 ≠ 会话创建字段。** 不要将 `outcome`、`rubric` 或 `description` 放在 `sessions.create()` 上——结果始终通过 `user.define_outcome` 事件发送。
- **Idle-break 门控不变。** 在排空循环中，继续使用 `event.type === 'session.status_idle' && event.stop_reason?.type !== 'requires_action'`——不要仅以 `span.outcome_evaluation_end` 作为门控（在 `needs_revision` 时会话继续运行）。参见 `shared/managed-agents-client-patterns.md` Pattern 5。

如需 Python 以外语言的原始 HTTP 形状和 SDK 绑定，请 WebFetch `https://platform.claude.com/docs/en/managed-agents/define-outcomes.md`（参见 `shared/live-sources.md`）。
