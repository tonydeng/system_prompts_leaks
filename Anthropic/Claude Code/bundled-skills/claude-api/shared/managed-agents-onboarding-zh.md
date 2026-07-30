> **说明**：本文件为英文原文（`managed-agents-onboarding.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Managed Agents — 上手流程

> **通过 `/claude-api managed-agents-onboard` 调用？** 你来对地方了。运行下面的访谈，不要向用户总结它，直接问问题。

Claude Managed Agents 是一种托管 agent：Anthropic 运行 agent 循环并为每个会话提供沙箱化容器来执行 agent 的工具（或使用你自己的 worker，配合 `self_hosted` 环境，参见 `shared/managed-agents-self-hosted-sandboxes.md`）。你提供 **agent 配置**（工具、skills、模型、系统提示，可复用、有版本）和**环境配置**（沙箱，可跨 agent 复用）。每次运行是一个**会话**。

流程分四步：**描述 -> agent -> 环境 -> 会话**，与 Console 快速入门相同的脉络，相同的理念：**凭证之前先出价值**。用户在任何认证要求之前从想法走到可运行的会话；每个凭证在设计使其相关的时刻（§2）被*标记*，在会话设置时（§4）*收集*一次，在那里绑定（`sessions.create()`）并被测试（冒烟测试）。将 `shared/managed-agents-core.md` 与此文档一起阅读，它有每个旋钮的完整细节；此文档是访谈脚本。

---

## 1. 描述任务

**以一句简短的引导语和一个开放式提示开始，不要猜测，不要问卷。** 用你自己的话：

> Managed Agents 是托管的，Anthropic 运行 agent 循环、沙箱和基础设施，你只需定义 agent。我们分三步走：agent、它运行的环境、然后一个实时测试会话。所以：描述你想要的 agent，它该做什么，什么触发它（一个人、一个事件、一个计划）？

让他们完整回答后再配置任何东西。

## 2. 配置 agent — 提议，而非审问

他们的描述完成访谈的工作。从中起草 agent 配置并**作为内嵌你建议的提案呈现**，用户对具体配置做出反应而非回答问题列表。至多一个批量追问来填补真正的空缺。在描述给出切入点的地方提出建议：

- **工具** — 默认启用完整的预构建工具集（`agent_toolset_20260401`：`bash`、`read`、`write`、`edit`、`glob`、`grep`、`web_fetch`、`web_search`）。为工作提到的任何第三方服务**建议 MCP 服务器**（GitHub、Linear、Slack 等），并在建议时标记每个服务暗示的凭证（"Linear MCP -> 启动时需要 Linear API token"），使 §4 的认证步骤成为手续而非意外。收集本身等到 §4。仅在用户自己的应用必须应答调用时使用自定义工具（名称、描述、输入 schema，其 handler 代码由他们编写，不要生成）。
- **Skills** — 当工作产出 `xlsx`/`docx`/`pptx`/`pdf` 文件时**建议**预构建 skills；通过 `skill_id` 自定义（每个 agent 最多 20 个，预构建 + 自定义合计）。
- **Outcome** — 如果描述暗示可检查的"完成"标准（或你可以在追问中引出：不是"一份好报告"而是"一个每个 SKU 有数字 `price` 列的 CSV"），**建议 Outcome 启动**，框架会根据评分标准评分和迭代（`shared/managed-agents-outcomes.md`）。
- **手头资源** — 磁盘上的仓库（`github_repository`：URL，可选 `mount_path`/`checkout`；token 在 §4 来），要填充的文件（Files API 上传 -> `{type: "file", file_id, mount_path}`；只读），如果工作引用它们。
- **模型** — 默认 `claude-opus-4-8`；最难的长程任务用 `claude-fable-5`（`shared/model-migration.md` -> Migrating to Claude Fable 5）。

> ‼️ **创建 PR 也需要 GitHub MCP 服务器** — `github_repository` 挂载仅是文件系统层面的。在挂载中编辑 -> 通过 `bash` 推送分支 -> 通过 MCP `create_pull_request` 工具发起 PR。

每个旋钮的完整细节：`shared/managed-agents-tools.md`（工具集、MCP、自定义工具、skills）、`shared/managed-agents-environments.md`（仓库、文件）。

## 3. 环境

通常零或一个问题：

- **复用还是创建？** 环境跨 agent 共享，先检查是否已有。
- **网络** — 默认不受限出口。仅在用户想要出口控制时切换到 `limited`，然后设置 `allow_mcp_servers: true` 或在 `allowed_hosts` 中列出每个 MCP 服务器域名，否则这些工具会静默失败。
- **建议 `self_hosted`** 当信号出现时：工具必须在自己的基础设施上运行、密钥不能离开它、或需要云容器没有的二进制文件/数据（`shared/managed-agents-self-hosted-sandboxes.md`；Claude Platform on AWS 上不可用）。否则用 `cloud`，不要为简单工作主动提及。

## 4. 会话 — 认证，然后测试运行

**认证在此发生 — 收集 §2 中标记的凭证，现在配置已定：** 一个 vault（已有的或 `vaults.create()`）+ 为 §2 中声明的每个 MCP 服务器创建 `vaults.credentials.create()`，为工作使用的 API key 创建 `environment_variable` 凭证（在出口处替换；沙箱看到占位符），以及每个仓库挂载的 `authorization_token`。凭证只写；MCP 凭证按 URL 匹配服务器并自动刷新。参见 `shared/managed-agents-tools.md` -> Vaults。

**静默可行性门控 — 在发出任何内容前自己运行此检查，只暴露缺口。** 逐条检查工作：每个动词映射到已启用的工具或 MCP 服务器（"开 PR" -> GitHub MCP，不只是挂载）；每个 MCP 服务器和仓库挂载都有认证步骤的凭证；每个外部主机在网络选择下可达；工作引用的每个文件/仓库/数据集都已挂载；"完成"是可检查的。如果缺什么，说出来并解决，不要发出你已知资源不足的配置。

**启动方式 — 选一种，绝不同时用：**
- `user.message` — 对话式。
- `user.define_outcome` + 评分标准 — 当 §2 确定了 Outcome 时；框架迭代和评分直到评分标准通过。
- **计划形态？** 完全跳过每会话启动，创建一个**部署**（`deployments.create()` 带 `schedule` + `initial_events`）；每次触发自主创建会话。参见 `shared/managed-agents-scheduled-deployments.md`。

要编入运行时代码的机制：会话创建在资源挂载前阻塞（错误的挂载在那里暴露，在 token 之前）；在发送启动事件*之前*打开事件流；在 `session.status_terminated` 或带终止 `stop_reason` 的 `session.status_idle` 上退出，除 `requires_action` 外的任何情况（`shared/managed-agents-client-patterns.md` Pattern 5）；usage 落在 `span.model_request_end` 上；产物落在 `/mnt/session/outputs/`（`files.list({scope_id: session.id, ...})`）。

## 5. 集成 — 生成代码

从最后一个答案直接到代码，没有前言，没有关于设置 vs 运行时的说教；两块结构展示了它。生成**两个清晰分离的块**：

**块 1 — 设置（运行一次，存储 ID）。** 优先使用 **YAML 文件 + `ant` CLI**，agent 和 environment 是用户应该签入并从 CI 应用的版本控制定义：

1. `<name>.agent.yaml`（扁平结构：`name`、`model`、`system`、`tools`、`mcp_servers`、`skills`）和 `<name>.environment.yaml`
2. ```sh
   AGENT_ID=$(ant beta:agents create < <name>.agent.yaml --transform id -r)
   ENV_ID=$(ant beta:environments create < <name>.environment.yaml --transform id -r)
   # CI sync: ant beta:agents update --agent-id "$AGENT_ID" --version N < <name>.agent.yaml
   ```

SDK 回退当用户要求时，且在 **Claude Platform on AWS 上必需**（认证为 SigV4，`ant` CLI 没有 SigV4 模式，使用 `shared/claude-platform-on-aws.md` 中的平台客户端）：标记为 `# ONE-TIME SETUP — run once, save the IDs` 并调用 `environments.create()` -> `agents.create()`。

> ⚠️ **Deployments 比 MA 表面的其他部分更新。** 在发出 `ant beta:deployments …` 或 `client.beta.deployments` / `client.beta.deployment_runs` 调用前，验证用户安装的 CLI/SDK 暴露了它们（`ant beta:deployments --help`；`hasattr(client.beta, "deployments")`）。如果没有，使用 `managed-agents-2026-04-01` beta header 发出原始 HTTP 调用 `POST /v1/deployments`（使用 `ant auth print-credentials` 的 Bearer token 认证时加 `oauth-2025-04-20`），并留下升级备注标记什么可以简化为 SDK 调用。

**计划形态？部署是设置，不是运行时。** 在块 1 中创建它，在 agent/环境 ID 存在后（`deployments.create()` 带 `schedule` + `initial_events`）。块 2 就**不是**会话循环，没有每次运行的启动要发送。改为发出：手动运行触发器（`POST /v1/deployments/{id}/run`）让用户现在可以测试而非等待第一次触发（手动运行兼作冒烟测试），加上一个获取 helper（最新 `deployment_runs` 条目 -> `session_id` -> Console URL + 产物的 `files.list(scope_id=session_id)`）。

**块 2 — 运行时（每次调用；对话式和 Outcome 形态）。** 检测到的语言的 SDK 代码（Python/TS/cURL，SKILL.md -> Language Detection），不要在这里发出 shell 循环：

1. 从配置/环境加载 `agent_id` + `env_id`
2. `sessions.create(agent=AGENT_ID, environment_id=ENV_ID, resources=[...], vault_ids=[...])`，然后打印 Console URL 让用户可以实时观看：`https://platform.claude.com/workspaces/default/sessions/{session.id}`（将 `default` 替换为他们的 workspace slug）
3. **当工作依赖 MCP 服务器、凭证或锁定的主机时冒烟测试**，这些失败不在 `sessions.create()` 上暴露，只在首次使用时。一个低成本的探测回合（"确认你可以到达 <service> 并列出 1-2 项；不要开始任务"），验证，然后发送真正的启动。没有外部依赖时跳过。
4. 打开流 -> 发送 §4 启动 -> 用 §4 的终止门控循环。

> ⚠️ **绝不在同一个未防护的块中发出 `agents.create()` 和 `sessions.create()`**，那会教导每次运行创建新 agent，这是头号反模式。单脚本请求：将创建包裹在 `if not os.getenv("AGENT_ID"):` 中。

从你检测到的语言的 `{lang}/managed-agents/README.md` 中提取确切语法（cURL 和 C#：使用 `curl/managed-agents.md` 作为线路级参考）。不要发明字段名。
