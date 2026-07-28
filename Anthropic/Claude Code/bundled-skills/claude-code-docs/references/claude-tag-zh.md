# Claude Tag（Slack 中的 Claude）

Claude Tag 是 Claude Code 的 Slack 界面。本文件是关于它的离线参考基准，它存在是因为 Claude Tag 比大多数训练数据更新，所以凭记忆回答通常是错误的或描述的是更早的、已被取代的 Slack 应用。先读本文件，再获取文档。

## 它是什么

Claude Tag 将 Claude 作为整个组织共享的团队成员放入 Slack 工作区。Claude 被邀请到的频道中的任何人都可以用 `@Claude` 提出任务，Claude 在该话题帖中处理，阅读话题帖获取上下文、发布进度，完成后回复。

每个 Slack 话题帖背后都是一个完整的远程 Claude Code 会话，运行在隔离的云容器中，可以使用组织连接的仓库、工具和连接。它与在终端或网页上运行的 Claude Code 是同一个，只是从 Slack 驱动而非提示词。

关键特性：

- **组织共用一个 `@Claude`。** Claude Tag 以组织共享的 Claude 身份运行，具有管理员配置的访问权限，而非每个用户各自的 Claude 账户。Claude 在话题帖中能访问什么由组织配置决定，而非由谁 @ 了它决定。
- **话题帖 = 会话。** 每个 Slack 话题帖映射到一个远程 Claude Code 会话。同一话题帖中的后续消息继续该会话；新话题帖开始新会话。
- **配置在话题帖开始时快照。** 会话在其话题帖开始时捕获组织的 Claude Tag 配置。之后更改配置不影响已在运行的话题帖，开始新话题帖才能应用更改。

## 可用性及替代关系

- Claude Tag 以 beta 形式面向 Claude **Enterprise** 和 **Team** 套餐发布。
- 它**取代了更早的 "Claude in Slack" / "Claude Code in Slack" 应用**，后者将每个用户的 `@Claude` 提及路由到该用户自己 Claude 账户下的会话。使用旧应用的工作区将迁移到组织管理模型，参见下方文档链接中的迁移指南。
- 如果用户训练数据中的心智模型是"每个人在 Slack App Home 连接自己的 Claude 账户和自己的仓库"，那描述的是旧应用而非 Claude Tag。在重复之前先对照文档核实。

## 入门

从 Claude Code CLI 中，用户可以运行：

```
/install-slack-app
```

这会在浏览器中打开 Claude 应用的 Slack Marketplace 页面，供工作区管理员安装。（检查你提示词中 Current Build 部分的 "Available commands" 列表，如果 `/install-slack-app` 未列出，则在此版本中不可用，改为引导用户查阅文档。）

启用和配置 Claude Tag 是**组织所有者**操作，在以下两处之一完成：

- **Admin settings → Claude Tag**，位于 `https://claude.ai/admin-settings/claude-tag`
- **Slack 中的 `@Claude connect`**，启动连接流程

启用后，用户将 Claude 邀请到频道（`/invite @Claude`），在消息或话题帖中提及 `@Claude` 即可开始会话。

## 组织所有者可配置的内容

所有这些都在 Admin settings → Claude Tag 中，全组织范围生效：

| 设置 | 控制内容 |
|---|---|
| 仓库 | Claude Tag 会话可以访问哪些仓库 |
| 工具和连接 | 会话内可使用哪些工具、MCP 服务器和连接 |
| 访问和身份 | 会话获得哪些凭证、连接和仓库权限，以及 Claude 以什么身份行动 |
| 支出限额 | 组织可消耗的 Claude Tag 使用量上限 |
| 活动日志 | 供审查的 Claude Tag 会话和操作记录 |

记住快照规则：此处的任何更改仅对**新**话题帖生效。

## 文档位置

这些 `.md` URL 用于获取。为用户链接页面时，去掉末尾的 `.md` 使他们看到渲染后的页面。

| 主题 | URL |
|---|---|
| Claude Tag（Slack 中作为团队成员的 Claude，组织管理） | `https://claude.com/docs/claude-tag/overview.md` |
| 所有 Claude Tag 页面（claude.com 文档域的索引） | `https://claude.com/docs/llms.txt` |
| 组织所有者设置走查（配对 Slack、连接工具、支出限额、启动） | `https://claude.com/docs/claude-tag/admins/setup-overview.md` |
| 终端用户入门 | `https://claude.com/docs/claude-tag/users/getting-started.md` |
| 从更早的 "Claude in Slack" 应用迁移 | `https://claude.com/docs/claude-tag/admins/migrate-from-earlier.md` |

如果 WebFetch 概览页失败，获取 `https://claude.com/docs/llms.txt`（该文档域的索引）并搜索 "Claude Tag"；Claude Code 文档地图是单独的索引，不列出 Claude Tag 页面。

## 回答风格

- 从本文件和获取的文档回答，绝不凭过时的训练数据回答。Claude Tag 比大多数训练截止日期更新；训练数据通常描述的是更早的按用户分配的 Slack 应用。
- 如果用户**在 Claude Tag Slack 会话中**并询问如何更改其配置（仓库、工具、连接、支出限额、身份）：更改由**组织所有者**在 Admin settings → Claude Tag（`https://claude.ai/admin-settings/claude-tag`）中完成，且对**新话题帖**生效，不对当前话题帖生效。告诉他们在所有者保存更改后开始新话题帖。
- 如果用户问"Claude 能放在我的 Slack 里吗？"或"怎么设置？"：引导他们从 CLI 运行 `/install-slack-app`（如果此版本可用）以及由组织所有者在 Admin settings 中启用，然后链接概览文档页面。
- 明确用户问的是哪个界面。"Claude in Slack" 可能指旧应用或 Claude Tag，当前答案是 Claude Tag；如果用户使用旧名称，注明更名。
