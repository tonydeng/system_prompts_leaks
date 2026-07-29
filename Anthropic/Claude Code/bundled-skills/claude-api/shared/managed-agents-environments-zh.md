# 托管智能体 — 环境与资源

## 环境

创建会话需要 `environment_id`。环境是用于在 Anthropic 基础设施中启动容器的**可复用配置模板** — 你可以为不同用例创建不同的环境（例如数据可视化 vs Web 开发，使用不同的包集合）。Anthropic 负责扩缩容、容器生命周期和工作编排。

**环境名称必须唯一。** 使用已有名称创建环境会返回 409。

### 网络

| 网络策略         | 描述                                                           |
| ---------------- | ------------------------------------------------------------- |
| `unrestricted`   | 完全出站（法律黑名单除外）                                      |
| `limited`        | 默认拒绝；通过 `allowed_hosts` / `allow_package_managers` / `allow_mcp_servers` 按需开放 |

```json
{
  "networking": {
    "type": "limited",
    "allow_package_managers": true,
    "allow_mcp_servers": true,
    "allowed_hosts": ["api.example.com"]
  }
}
```

`limited` 的三个字段都是可选的。`allow_package_managers`（默认 `false`）允许 PyPI/npm 等；`allow_mcp_servers`（默认 `false`）允许智能体配置的 MCP 服务器端点，无需在 `allowed_hosts` 中列出。

**MCP 注意事项：** 在 `limited` 网络下，要么设置 `allow_mcp_servers: true`，要么将每个 MCP 服务器域名添加到 `allowed_hosts`。否则容器无法访问它们，工具会静默失败。

### 创建环境

SDK 会自动添加 `managed-agents-2026-04-01`。TypeScript：

```ts
const env = await client.beta.environments.create({
  name: "my_env",
  config: {
    type: "cloud",
    networking: { type: "unrestricted" },
  },
});
```

### 自托管沙箱

要在**你自己的基础设施**而非 Anthropic 的基础设施中运行工具执行，设置 `config: {type: "self_hosted"}` — 智能体循环仍在 Anthropic 侧运行，但 `bash` / 文件操作 / 代码执行在你通过出站轮询 worker 控制的容器中进行。`networking` 块不适用（你控制出站）。资源挂载（`file`、`github_repository`）和内存存储的行为不同 — 参见 `shared/managed-agents-self-hosted-sandboxes.md` 了解 worker、凭据和云 vs 自托管的对比。

### 环境 CRUD

| 操作             | 方法     | 路径                                       | 备注 |
| ---------------- | -------- | ------------------------------------------ | ----- |
| 创建             | `POST`   | `/v1/environments`                         | |
| 列表             | `GET`    | `/v1/environments`                         | 分页（`limit`, `after_id`, `before_id`） |
| 获取             | `GET`    | `/v1/environments/{id}`                    | |
| 更新             | `POST`   | `/v1/environments/{id}`                    | 变更仅应用于**新**容器；现有会话保持其原始配置 |
| 删除             | `DELETE` | `/v1/environments/{id}`                    | 返回 204。 |
| 归档             | `POST`   | `/v1/environments/{id}/archive`            | 使其变为**只读**；现有会话继续运行，新会话无法引用它。不可取消归档 — 终态。 |

---

## 资源

将文件、GitHub 仓库和内存存储附加到会话。**会话创建会阻塞直到所有资源挂载完成** — 容器在所有文件和仓库就位前不会进入 `running` 状态。每个会话最多 **999 个文件资源**。每个会话支持多个 GitHub 仓库。关于 `type: "memory_store"` 资源（跨会话持久化内存 — 每个会话最多8个），参见 `shared/managed-agents-memory.md`。

### 文件上传（输入 — 主机 → 智能体）

先通过 Files API 上传文件，然后通过 `file_id` + `mount_path` 引用：

```ts
// 1. 上传
const file = await client.beta.files.upload({
  file: fs.createReadStream("data.csv"),
  purpose: "agent",
});

// 2. 作为会话资源附加
const session = await client.beta.sessions.create({
  agent: agent.id,
  environment_id: envId,
  resources: [
    { type: "file", file_id: file.id, mount_path: "/workspace/data.csv" }
  ],
});
```

**`mount_path` 是必需的**且必须是绝对路径。父目录会自动创建。智能体工作目录默认为 `/workspace`。文件以只读方式挂载 — 智能体将修改后的版本写入新路径。

### 会话输出（输出 — 智能体 → 主机）

智能体可以在会话期间将文件写入 `/mnt/session/outputs/`。这些文件会被 Files API 自动捕获，之后可以列出和下载：

```ts
// 在轮次完成后，列出此会话范围内的输出文件：
for await (const f of client.beta.files.list({
  scope_id: session.id,
  betas: ["managed-agents-2026-04-01"],
})) {
  console.log(f.filename, f.size_bytes);
  const resp = await client.beta.files.download(f.id);
  const text = await resp.text();
}
```

**要求：**
- 必须为智能体启用 `write` 工具（或 `bash`）才能创建输出文件。
- 会话范围的 `files.list` / `files.download` 捕获写入 `/mnt/session/outputs/` 的输出。
- 过滤参数是 **`scope_id`**（REST 查询参数 `?scope_id=<session_id>`）。SDK 的 files 资源仅自动添加 `files-api-2025-04-14` 头，因此需显式传递 `betas: ["managed-agents-2026-04-01"]`（或在原始 HTTP 上同时传两个头）— 没有它 API 可能会将 `scope_id` 拒绝为未知字段。需要 `@anthropic-ai/sdk` ≥ 0.88.0 / `anthropic`（Python）≥ 0.92.0 — 旧版本不识别 `scope_id` 类型。`ant` CLI **尚未**暴露此标志；使用 SDK 或 curl。
- 将 `sessions.create()` 返回的会话 ID 原样传递（如 `sesn_011CZx...`）— API 会验证前缀。
- 在 `session.status_idle` 和输出文件出现在 `files.list` 之间有短暂的索引延迟（约1-3秒）。如果为空，重试一两次。

> **`scope_id` 过滤不可用时的后备方案**（旧版 SDK，或端点返回错误）：发送后续 `user.message` 要求智能体 `read` `/mnt/session/outputs/` 下的每个文件并返回内容。智能体会将文件内容作为 `agent.message` 文本流式传回。这仅适用于文本文件且消耗输出 token — 用于临时解阻，不作为主路径。

这提供了一个双向文件桥：上传参考数据进去，下载智能体制品出来。

### GitHub 仓库

在初始化期间将 GitHub 仓库克隆到会话容器中，在智能体开始执行之前。智能体可以通过 `bash`（`git`）读取、编辑、提交和推送。每个会话支持多个仓库 — 每个仓库添加一个 `resources` 条目。仓库会被缓存，因此使用相同仓库的未来会话启动更快。

仓库在会话生命周期内保持附加 — 要更改挂载的仓库，请创建新会话。你**可以**在运行中的会话上通过 `client.beta.sessions.resources.update(resource_id, {session_id, authorization_token})` 轮换仓库的 `authorization_token`；资源 `id` 在会话创建时和 `resources.list()` 返回。

**字段：**

| 字段 | 必需 | 备注 |
|---|---|---|
| `type` | ✅ | `"github_repository"` |
| `url` | ✅ | GitHub 仓库 URL |
| `authorization_token` | ✅ | 具有仓库访问权限的 GitHub 个人访问令牌。**从不在 API 响应中回显。** |
| `mount_path` | ❌ | 仓库将被克隆到的路径。默认为 `/workspace/<repo-name>`。 |
| `checkout` | ❌ | `{type: "branch", name: "..."}` 或 `{type: "commit", sha: "..."}`。默认为仓库的默认分支。 |

**令牌权限级别**（细粒度 PAT）：
- `Contents: Read` — 仅克隆
- `Contents: Read and write` — 推送变更和创建拉取请求

**认证工作原理：** `authorization_token` 永远不会放入容器内。`git pull` / `git push` 和针对附加仓库的 GitHub REST 调用通过 Anthropic 侧的 git 代理路由，该代理在请求离开沙箱后注入令牌。容器中运行的代码 — 包括智能体写入的任何内容 — 无法读取或窃取它。

> ‼️ **要生成拉取请求**，你还需要 GitHub **MCP 服务器**访问权限 — `github_repository` 资源仅提供文件系统 + git 访问。参见 `shared/managed-agents-tools.md` → MCP 服务器。PR 工作流是：在挂载的仓库中编辑文件 → 通过 `bash` 推送分支（通过 git 代理使用 `authorization_token` 认证）→ 通过 MCP `create_pull_request` 工具创建 PR（通过 vault 认证）。

**TypeScript：**

```ts
// 1. 创建智能体 — 声明 GitHub MCP（此处无认证）
const agent = await client.beta.agents.create(
  {
    name: 'GitHub Agent',
    model: 'claude-opus-4-8',
    mcp_servers: [
      { type: 'url', name: 'github', url: 'https://api.githubcopilot.com/mcp/' },
    ],
    tools: [
      { type: 'agent_toolset_20260401', default_config: { enabled: true } },
      { type: 'mcp_toolset', mcp_server_name: 'github' },
    ],
  },
);

// 2. 启动会话 — 为 MCP 认证附加 vault + 挂载仓库
const session = await client.beta.sessions.create({
  agent: agent.id,
  environment_id: envId,
  vault_ids: [vaultId],  // vault 包含 GitHub MCP OAuth 凭据
  resources: [
    {
      type: 'github_repository',
      url: 'https://github.com/owner/repo',
      authorization_token: process.env.GITHUB_TOKEN,  // 仓库克隆令牌（≠ MCP 认证）
      checkout: { type: 'branch', name: 'main' },
    },
  ],
});
```

**Python：**

```python
import os

agent = client.beta.agents.create(
    name="GitHub Agent",
    model="claude-opus-4-8",
    mcp_servers=[{
        "type": "url",
        "name": "github",
        "url": "https://api.githubcopilot.com/mcp/",
    }],
    tools=[
        {"type": "agent_toolset_20260401", "default_config": {"enabled": True}},
        {"type": "mcp_toolset", "mcp_server_name": "github"},
    ],
)

session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=env_id,
    vault_ids=[vault_id],  # vault 包含 GitHub MCP OAuth 凭据
    resources=[{
        "type": "github_repository",
        "url": "https://github.com/owner/repo",
        "authorization_token": os.environ["GITHUB_TOKEN"],  # 仓库克隆令牌（≠ MCP 认证）
        "checkout": {"type": "branch", "name": "main"},
    }],
)
```

---

## Files API

上传和管理文件作为会话资源，以及下载智能体写入 `/mnt/session/outputs/` 的文件。

| 操作             | 方法     | 路径                                  | SDK |
| ---------------- | -------- | ------------------------------------- | --- |
| 上传             | `POST`   | `/v1/files`                           | `client.beta.files.upload({ file })` |
| 列表             | `GET`    | `/v1/files?scope_id=...`              | `client.beta.files.list({ scope_id, betas: ["managed-agents-2026-04-01"] })` |
| 获取元数据       | `GET`    | `/v1/files/{id}`                      | `client.beta.files.retrieveMetadata(id)` |
| 下载             | `GET`    | `/v1/files/{id}/content`              | `client.beta.files.download(id)` → `Response` |
| 删除             | `DELETE` | `/v1/files/{id}`                      | `client.beta.files.delete(id)` |

List 上的 `scope_id` 过滤器将结果范围限定为该会话写入 `/mnt/session/outputs/` 的文件。不带过滤器时，你将获得上传到你账户的所有文件。
