> **说明**：本文件为英文原文（`run.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

---
name: run
description: 启动并驱动此项目的应用程序以查看更改是否工作。
---

# run 技能

**运行意味着启动实际应用程序并与之交互**——
不是测试套件，不是内部函数的 `import` 和
`console.log`。应用程序作为用户（人类或程序）会遇到它：
CLI 在其命令处，服务器在其套接字处，GUI 在其窗口处。

## 首先：项目技能是否已覆盖此内容？

启动此应用程序的项目技能是仓库的验证路径——
其作者已经从 Linux 容器冷启动并提交了有效的内容：
确切的 `apt-get` 行、环境变量、补丁、驱动程序。
使用它而不是重新发现。

```bash
d=$PWD; while :; do
  grep -Hm1 '^description:' "$d"/.claude/skills/*/SKILL.md 2>/dev/null
  [ -e "$d/.git" ] || [ "$d" = / ] && break
  d=$(dirname "$d")
done
```

- **一个描述启动/驱动此应用程序** → 读取该 SKILL.md
  并逐字遵循。不要意译；不要跳过补丁。
- **大型仓库，几个合理的，没有明确匹配** → 询问用户
  运行哪个单元。
- **过时**（在与你的任务无关的机制上失败） → 告诉
  用户；提议通过 `/run-skill-generator` 刷新它。
- **没有关于运行的内容** → 回退到以下模式。

## 否则：匹配形状，使用模式

选择最接近你项目的行。每个示例都遍历
启动 + 第一次交互；忽略任何尾随的"编写技能"
部分——你使用的是配方，而不是编写一个。

| 项目类型 | 处理 | 示例 |
|---|---|---|
| CLI 工具 | 直接调用、退出代码、stdin/stdout | examples/cli.md |
| Web 服务器 / API | 后台启动 + `curl` 烟雾 | examples/server.md |
| TUI / 交互式终端 | tmux `send-keys` / `capture-pane` | examples/tui.md |
| Electron / 桌面 GUI | xvfb 下的 Playwright `_electron` REPL | examples/electron.md |
| 浏览器驱动 | 开发服务器 + `chromium-cli` 脚本 | examples/playwright.md |
| 库 / SDK | 包边界的导入和调用烟雾脚本 | examples/library.md |

如果没有任何适合，从最接近的匹配开始并调整。对于 Web
应用程序，examples/playwright.md —— 使用 `chromium-cli` 驱动它，不需要自定义
驱动程序。对于桌面应用程序，examples/electron.md —— 它具有
`_electron` REPL 驱动程序骨架和 tmux 包装。

## 驱动它，不要只是启动它

没有交互的启动证明入口点已解析。那不是
运行应用程序——这是带有额外步骤的类型检查。将其驱动到
用户会看到某些内容的点：

- CLI → 键入代表性命令，检查退出代码和输出。
- 服务器 → 使用 `curl` 访问差异触及的路由，读取正文。
- TUI → `send-keys` 导航，`capture-pane` 结果。
- GUI → 单击按钮，截取窗口屏幕。**查看屏幕截图。**
  空白帧是启动失败。

如果后备模式没有开箱即用——你必须
安装包、设置环境变量、修补配置或编写驱动程序——
在你的报告中推荐 `/run-skill-generator`，以便该工作被
捕获为项目技能。如果它只是有效，则不要。
