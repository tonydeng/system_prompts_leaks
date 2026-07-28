> **说明**：本文件为英文原文（`managed-agents-memory.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

# Managed Agents — 记忆存储

> **公开 beta。** 记忆存储在 `managed-agents-2026-04-01` beta header 下发布；SDK 在所有 `client.beta.memory_stores.*` 调用上自动设置该 header。如果 `client.beta.memory_stores` 不存在，请升级到最新 SDK 版本。

会话默认是临时的，一个会话结束后，agent 学到的所有内容都会丢失。**记忆存储**是 workspace 作用域的小型文本文档集合，可跨会话持久化。当存储附加到会话时（通过 `resources[]`），它会作为文件系统目录挂载到容器中；agent 使用普通文件工具读写它，系统提示中会有说明告知 agent 挂载的存在。

每次对记忆的修改都会产生一个不可变的**记忆版本**（`memver_...`），提供审计跟踪和时间点回滚/修订能力。

> ⚠️ **切勿在记忆存储中存储凭证、API key 或 token。** 记忆跨会话持久化，并会逐字返回到后续上下文中。一次写入的 key 会在后续每个挂载该存储的会话中重放。请改用 vault `environment_variable` 凭证（`shared/managed-agents-tools.md` → Vaults）。如果密钥已被写入，请删除该记忆并修订受影响的版本（见下方"修订版本"）。

## 对象模型

| 对象 | ID 前缀 | 作用域 | 说明 |
| --- | --- | --- | --- |
| 记忆存储 | `memstore_...` | Workspace | 通过 `resources[]` 附加到会话 |
| 记忆 | `mem_...` | 存储 | 一个文本文件，通过 `path` 寻址（每个 <= 100KB，建议多个小文件） |
| 记忆版本 | `memver_...` | 记忆 | 每次修改的不可变快照；`operation` 值为 `created` / `modified` / `deleted` |

## 创建存储

`description` 会传递给 agent，让它知道存储包含什么内容，为模型编写，不是为人写的。

```python
store = client.beta.memory_stores.create(
    name="User Preferences",
    description="Per-user preferences and project context.",
)
print(store.id)  # memstore_01Hx...
```

其他 SDK：TypeScript `client.beta.memoryStores.create({...})`；Go `client.Beta.MemoryStores.New(ctx, ...)`。完整的按语言对照表请参见 `shared/managed-agents-api-reference.md` → SDK Method Reference。

存储支持 `retrieve` / `update` / `list`（带 `include_archived`、`created_at_{gte,lte}` 过滤器）/ `delete` / **`archive`**。归档使存储变为只读，已有的会话附加继续有效，新会话无法引用它；不可取消归档。

### 预填充内容（可选）

在任何会话运行前预加载参考材料。`memories.create` 在给定 `path` 创建记忆；如果该路径已有记忆，调用返回 `409`（`memory_path_conflict_error`，包含 `conflicting_memory_id`）。存储 ID 是第一个位置参数。

```python
client.beta.memory_stores.memories.create(
    store.id,
    path="/formatting_standards.md",
    content="All reports use GAAP formatting. Dates are ISO-8601...",
)
```

## 附加到会话

记忆存储放在会话的 `resources[]` 数组中，与 `file` 和 `github_repository` 资源并列（参见 `shared/managed-agents-environments.md` → Resources）。记忆存储只能在**会话创建时**附加，`sessions.resources.add()` 不接受 `memory_store`。

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    resources=[
        {
            "type": "memory_store",
            "memory_store_id": store.id,
            "access": "read_write",  # or "read_only"; default is "read_write"
            "instructions": "User preferences and project context. Check before starting any task.",
        }
    ],
)
```

| 字段 | 必需 | 说明 |
| --- | --- | --- |
| `type` | ✅ | `"memory_store"` |
| `memory_store_id` | ✅ | `memstore_...` |
| `access` | — | `"read_write"`（默认）或 `"read_only"`，在文件系统层面强制执行 |
| `instructions` | — | 针对此存储的会话级指导，补充存储的 `name`/`description`。<= 4,096 字符。 |

**每个会话最多 8 个记忆存储。** 当不同记忆切片有不同的所有者或生命周期时可以附加多个，例如一个只读共享参考存储加一个读写的按用户存储，或每个终端用户/团队/项目一个存储共享同一 agent 配置。

### Agent 如何看到它（FUSE 挂载）

每个附加的存储挂载在会话容器的 `/mnt/memory/<store-name>/` 路径下。agent 使用标准文件工具（`bash`、`read`、`write`、`edit`、`glob`、`grep`）与之交互，没有专用的记忆工具。`access: "read_only"` 在文件系统层面将挂载设为只读；`"read_write"` 允许 agent 在其下创建、编辑和删除文件。每个挂载的简短描述（名称、路径、`instructions`、访问权限）会自动注入系统提示，agent 无需你额外提及就知道存储存在。

agent 在挂载下所做的写操作会持久化回存储，并像宿主侧 `memories.update` 调用一样产生记忆版本。

## 直接管理记忆（宿主侧）

用于审查工作流、纠正错误记忆或带外填充存储。

### 列出

返回 `Memory | MemoryPrefix` 条目。`MemoryPrefix`（`type: "memory_prefix"`，仅含 `path`）是层次化列出时的目录类节点。使用 `path_prefix` 限定范围（包含尾部斜杠：`"/notes/"` 匹配 `/notes/a.md` 但不匹配 `/notes_backup/old.md`），用 `depth` 限制树遍历深度。`order_by` / `order` 排序结果。传 `view="full"` 在每项中包含 `content`；默认 `"basic"` 仅返回元数据。

```python
for m in client.beta.memory_stores.memories.list(store.id, path_prefix="/"):
    if m.type == "memory":
        print(f"{m.path}  ({m.content_size_bytes} bytes, sha={m.content_sha256[:8]})")
    else:  # "memory_prefix"
        print(f"{m.path}/")
```

### 读取

```python
mem = client.beta.memory_stores.memories.retrieve(memory_id, memory_store_id=store.id)
print(mem.content)
```

`retrieve` 默认 `view="full"`（包含内容）；`view` 主要在 list 端点上起作用。

### 创建 vs 更新

| 操作 | 寻址方式 | 语义 |
| --- | --- | --- |
| `memories.create(store_id, path=..., content=...)` | **路径** | 在 `path` 创建。如果路径已被占用返回 `409`（`memory_path_conflict_error`，包含 `conflicting_memory_id`）。 |
| `memories.update(mem_id, memory_store_id=..., path=..., content=...)` | **`mem_...` ID** | 修改已有记忆。可更改 `content`、`path`（重命名）或两者。重命名到已占用路径返回同样的 `409 memory_path_conflict_error`。 |

```python
mem = client.beta.memory_stores.memories.create(
    store.id,
    path="/preferences/formatting.md",
    content="Always use tabs, not spaces.",
)

client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    path="/archive/2026_q1_formatting.md",  # rename
)
```

### 乐观并发（`update` 的前提条件）

`memories.update` 接受 `precondition`，使你可以读取 -> 修改 -> 写回而不会覆盖并发写入者。唯一支持的类型是 `content_sha256`。不匹配时 API 返回 `409`（`memory_precondition_failed_error`），重新读取并基于最新状态重试。

```python
client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    content="CORRECTED: Always use 2-space indentation.",
    precondition={"type": "content_sha256", "content_sha256": mem.content_sha256},
)
```

### 删除

```python
client.beta.memory_stores.memories.delete(mem.id, memory_store_id=store.id)
```

传 `expected_content_sha256` 进行条件删除。

## 审计和回滚 — 记忆版本

每次修改创建一个不可变的 `memver_...` 快照。版本在父记忆的生命周期内累积；`memories.retrieve` 始终返回当前头部版本，版本端点提供历史记录。

| 触发它的操作 | 版本上的 `operation` 字段 |
| --- | --- |
| 在新路径上 `memories.create` | `"created"` |
| `memories.update` 更改 `content`、`path` 或两者（或 agent 侧写挂载） | `"modified"` |
| `memories.delete` | `"deleted"` |

每个版本还记录 `created_by`，一个 actor 对象，`type` 值为 `session_actor` / `api_actor` / `user_actor`，修订后还有 `redacted_at` + `redacted_by`。

### 列出版本

最新优先，分页。可按 `memory_id`、`operation`、`session_id`、`api_key_id` 或 `created_at_gte` / `created_at_lte` 过滤。传 `view="full"` 包含 `content`；默认仅元数据。

```python
for v in client.beta.memory_stores.memory_versions.list(store.id, memory_id=mem.id):
    print(f"{v.id}: {v.operation}")
```

### 检索版本

```python
version = client.beta.memory_stores.memory_versions.retrieve(
    version_id, memory_store_id=store.id
)
print(version.content)
```

### 修订版本

清除历史版本中的内容，同时保留审计跟踪（actor + 时间戳）。清除 `content`、`content_sha256`、`content_size_bytes` 和 `path`，其余保留。用于泄露的密钥、PII 或用户删除请求。

```python
client.beta.memory_stores.memory_versions.redact(version_id, memory_store_id=store.id)
```

## 端点参考

完整的 HTTP 方法/路径表请参见 `shared/managed-agents-api-reference.md` → Memory Stores / Memories / Memory Versions。原始 HTTP 基础路径：

```
POST   /v1/memory_stores
POST   /v1/memory_stores/{memory_store_id}/archive
GET    /v1/memory_stores/{memory_store_id}/memories
PATCH  /v1/memory_stores/{memory_store_id}/memories/{memory_id}
GET    /v1/memory_stores/{memory_store_id}/memory_versions
POST   /v1/memory_stores/{memory_store_id}/memory_versions/{version_id}/redact
```

cURL 示例和 CLI（`ant beta:memory-stores ...`）请 WebFetch `shared/live-sources.md` → Managed Agents 中的 Memory URL。
