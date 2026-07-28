# 注入式提醒

Claude Code 将这些模型可见的提示词包裹在 `<system-reminder>` 中，当某个功能或模式触发时注入到对话回合里——它们不作为任何命令的提示词注册，也不通过输入 `/name` 调用。这是第三种提示词通道（与技能和斜杠命令提示词并列）；它是最容易被遗漏的通道，因为这些提示词没有命令入口。

| 文件 | 触发时机 |
|---|---|
| `teammate.md` | 会话加入智能体团队时（团队协调指令） |
| `remote-plan.md` | 远程/云端规划会话被触发时 |
| `plan-multiagent.md` | 计划模式运行多智能体实现计划流程时 |
| `non-interactive.md` | 会话以非交互/打印模式运行时 |
| `container-restart.md` | 容器重启且后台任务被停止时 |
| `model-switched.md` | 会话模型被切换时（`/model`） |
| `brief-mode.md` | 简洁模式启用时（输出通过工具重定向） |

从 Claude Code 2.1.211 二进制文件中原样捕获。运行时数据槽使用其真实的语义名称（如 `${agentName}`、`${model}`、`${stoppedTasks}`）。
