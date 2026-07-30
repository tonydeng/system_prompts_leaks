> **说明**：本文件为英文原文（`artifact-capabilities.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

---
name: artifact-capabilities
description: 已发布 Artifact 可声明的运行时能力，包括从页面调用用户的 claude.ai 连接器（MCP）以及未来能力。在将 `capabilities` 传递给 Artifact 工具或编写任何 `window.claude.mcp` 代码之前，请先加载此技能。
---

# Artifact 运行时能力

已发布的 Artifact 页面可以通过向 Artifact 工具传递 `capabilities: {name: config}` 来声明**运行时能力**，即 claude.ai 查看器在打开时授予页面的能力。控制面是有效名称和配置结构的权威来源。声明动作的含义：在重新部署时**省略** `capabilities` 会原样保留已存储的声明（并保留 artifact 的已存储合约固定）；**空对象** `{}` 是显式的全部清除；**非空对象**是全量声明（已存储但未重新声明的任何内容都会被撤销）。移动重新发布 artifact 的运行时版本是一个有意的动作，传递 `contract: 'latest'` 来升级，或传递特定版本来固定或回滚，绝不是编辑的副作用。

运行时合约 0.1.12


--- capability: downloads ---

`downloads` 能力允许已发布的页面向查看器提供生成的文件：声明 `capabilities: {downloads: true}`，然后调用 `window.claude.downloads.save({filename, data})`。查看器会看到确认提示并可以拒绝，保存绝不会静默执行或保证成功，因此应在查看者明确有意时提供此功能，并处理拒绝情况。首次使用前检查 `window.claude.downloads` 是否存在；类型定义对调用合约和错误代码具有权威性。


--- capability: mcp ---

`mcp` 允许已发布的页面通过 `window.claude.mcp` 调用查看器的 claude.ai 连接器；调用使用查看器的凭据运行，从不暴露令牌。声明 `capabilities: {mcp: {servers: [{server, tools}]}}`，其中 `server` 是连接器的显示名称。保持清单最小化：这是查看者授权的许可，声明此能力的页面无法公开分享。两个分支：展示数据的部分注册 `watchTool(server, tool, input, handler, opts?)`，它会重放缓存、在过期时刷新、仅通过 `refetchInterval` 轮询，并返回一个同步的取消订阅函数供存储；操作部分调用 `callTool` 一次并读取 `result.payload`。工具失败会 REJECT（`tool_error`）；watch 失败以 handler 错误事件的形式到达。首先阅读类型定义：按错误代码分支用户体验，仅重试 `retryable` 错误，在授权拒绝时丢弃数据，从 `result.cache.storedAt` 驱动新鲜度 UI。它们省略了参数名称和结果编码：观察每个工具的一组真实请求/响应对，或者在发布时说明情况，绝不要猜测。


**本次会话中的连接器。** 连接器工具在你的工具列表中显示为 `mcp__<connector>__<toolName>`。将 `server` 设为 `<connector>` 段，即 `mcp__` 和下一个 `__` 之间的所有内容（对于 `mcp__claude_ai_Slack_beta__search`，`server` 是 `claude_ai_Slack_beta`）。精确复制该段，包括大小写；发布时会自动解析为连接器的显示名称。只有 claude.ai 连接器有效，本地配置的 MCP 服务器无效。清单的 `tools` 数组采用连接器的上游工具名称（由 `listTools()` / `/v1/mcp_servers` 返回），当上游名称包含 `.` 或空格时，可能与规范化的 `<toolName>` 段不同。在连接器未加载但设置了 `$CLAUDE_CODE_OAUTH_TOKEN` 的隔离/CI 会话中，通过 Bash 获取列表：`curl -H 'anthropic-version: 2023-06-01' -H 'anthropic-beta: mcp-servers-2025-12-04' -H "Authorization: Bearer $CLAUDE_CODE_OAUTH_TOKEN" https://api.anthropic.com/v1/mcp_servers?limit=1000`；在这种情况下，使用每个条目的 `display_name` 作为 `server` 值（确切的显示名称始终与工具前缀段一起被接受）。

**调用合约**（运行时合约 0.1.12）。平台提供的此合约的 `window.claude` 类型定义提取在 `<skill-dir>` 下：`0.1.12/downloads.d.ts`、`0.1.12/mcp.d.ts`。在编写任何 `window.claude.mcp` 调用之前，请先阅读 `<skill-dir>/0.1.12/mcp.d.ts`，它对于此合约版本具有超越任何记忆中 API 形状的权威性。类型定义仅涵盖调用信封，不会告诉你连接器工具的参数名称或其结果编码。绝不要在未于本次会话中观察到该工具的一组真实请求/响应对的情况下发布调用连接器工具的页面；如果无法安全观察（例如连接器在此处未认证，或调用工具会有副作用），在发布时明确告知用户，在你的回复中说明而非作为已发布页面内的注释，而不是发布一个猜测的结构。观察到的响应载荷是用户的真实数据：从中学习结构，但绝不要将观察到的值作为示例或占位数据嵌入已发布的页面中。
