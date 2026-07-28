# 斜杠命令提示词

Claude Code 内置斜杠命令背后的模型面提示词——用户通过输入 `/name` 调用的那些。这些与捆绑技能（见 `../bundled-skills/`）不同：它们注册为 `type:"prompt"` 或 `type:"local"` 命令，而非技能。

| 文件 | 命令 | 功能 |
|---|---|---|
| `btw.md` | `/btw` | 派生一个轻量级侧问智能体，在不打断主线程的情况下回答 |
| `compact.md` | `/compact` | 摘要对话以释放上下文 |
| `compact-continuation-message.md` | — | 压缩后注入到续接会话中的消息 |
| `compact-rewind-summarization.md` | — | 用于回退/自动压缩的摘要提示词 |
| `recap.md` | `/recap` | 一行式"你离开了"复述（与自动离开摘要共享） |
| `insights.md` | `/insights` | 生成会话分析报告 |
| `team-onboarding.md` | `/team-onboarding` | 从使用数据构建队友入门指南 |
| `session-title.md` | `/rename` | 自动生成会话标题（也用于自动命名） |

从 Claude Code 2.1.211 二进制中逐字捕获。运行时数据槽以真实语义名称显示（如 `${question}`、`${insightsJson}`）。
