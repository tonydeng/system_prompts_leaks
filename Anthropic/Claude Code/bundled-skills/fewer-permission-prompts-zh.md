> **说明**：本文件为英文原文（`fewer-permission-prompts.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

---
name: fewer-permission-prompts
description: 扫描你的转录以查找常见的只读 Bash 和 MCP 工具调用，然后将优先允许列表添加到项目 .claude/settings.json 以减少权限提示。
---

# 更少的权限提示

查看我的转录的 MCP 和 bash 工具调用，并基于这些，制作一个优先模式列表，我应该将其添加到我的权限允许列表中以减少权限提示。专注于只读命令。

权限的格式是：`Bash(foo*)`、`Bash(foo)`、`Bash(foo bar *)`、`mcp__slack__slack_read_thread` 等。

然后，将这些添加到项目 `.claude/settings.json` 下的 `permissions.allow`。

## 步骤

1. **定位转录。** 会话转录位于 `~/.claude/projects/<sanitized-cwd>/*.jsonl`。每行是一个 JSON 对象。工具调用显示为 `assistant` 消息，带有 `type: "tool_use"` 的 `message.content[]` 条目。`name` 字段标识工具（例如 `"Bash"`、`mcp__slack__slack_read_thread"`）；对于 Bash，`input.command` 是 shell 字符串。

    扫描用户项目目录中的最近转录——不仅仅是当前项目——以便允许列表反映他们的实际使用。将扫描限制在合理数量的最近会话（例如 50 个最近修改的 JSONL 文件），以便保持快速。

2. **提取工具调用频率。**
    - 对于 `Bash` 调用：解析 `input.command`，采用前导命令标记（处理 `sudo`、`timeout`、管道、`&&`、环境变量前缀）。记录命令 + 第一个子命令对（例如 `git status`、`gh pr view`、`ls`、`cat`）。
    - 对于 MCP 调用：记录完整工具名称（例如 `mcp__slack__slack_read_thread`）。
    - 计算扫描转录中的出现次数。

3. **过滤到只读。** 仅保留不改变状态的命令。只读示例：`ls`、`cat`、`pwd`、`git status`、`git log`、`git diff`、`git show`、`git branch`、`rg`、`grep`、`find`、`head`、`tail`、`wc`、`file`、`which`、`echo`、`date`、`gh pr view`、`gh pr list`、`gh pr diff`、`gh issue view`、`gh issue list`、`gh run list`、`gh run view`、`gh api` (GET)、`bun run typecheck`、`bun run lint`、`bun run test`（对于不改变的测试）、`docker ps`、`docker logs`、`kubectl get`、`kubectl describe`、`ps`、`top`、`df`、`du`、`env`、`printenv`、名称中带有 `read`/`get`/`list`/`search`/`view` 的任何 MCP 工具。

    删除任何写入、删除、重命名、推送、合并、安装或运行具有副作用的构建/测试的任何内容。如有疑问，将其排除。

    **永远不要允许列表授予任意代码执行的模式。** 其中任何一个的通配符规则（例如 `Bash(python3:*)`）等同于允许任意代码执行。此列表并非详尽无遗——对同一类别中的任何内容应用相同规则：
    - 解释器：`python`/`python3`、`node`、`bun`、`deno`、`ruby`、`perl`、`php`、`lua` 等。
    - Shell：`bash`、`sh`、`zsh`、`fish`、`eval`、`exec`、`ssh` 等。
    - 包运行器：`npx`、`bunx`、`uvx`、`uv run` 等。
    - 任务运行器通配符：`npm run *`、`yarn run *`、`pnpm run *`、`bun run *`、`make *`、`just *`、`cargo run *`、`go run *` 等——精确的 `Bash(bun run typecheck)` 可以，`Bash(bun run *)` 不行
    - `gh api *`、`docker run`/`exec`、`kubectl exec`、`sudo` 和类似

4. **删除 Claude Code 已经自动允许的命令。** 这些不需要允许列表条目——它们从不提示。如果你在转录中看到其中任何一个，跳过它们；不要向用户建议它们。

    - **始终自动允许（任何参数）：** `cal`、`uptime`、`cat`、`head`、`tail`、`wc`、`stat`、`strings`、`hexdump`、`od`、`nl`、`id`、`uname`、`free`、`df`、`du`、`locale`、`groups`、`nproc`、`basename`、`dirname`、`realpath`、`cut`、`paste`、`tr`、`column`、`tac`、`rev`、`fold`、`expand`、`unexpand`、`fmt`、`comm`、`cmp`、`numfmt`、`readlink`、`diff`、`true`、`false`、`sleep`、`which`、`type`、`expr`、`test`、`getconf`、`seq`、`tsort`、`pr`、`echo`、`printf`、`ls`、`cd`、`find`。
    - **仅零参数自动允许：** `pwd`、`whoami`、`alias`。
    - **自动允许的精确形式：** `claude -h`、`claude --help`、`node -v`、`node --version`、`python --version`、`python3 --version`、`ip addr`。
    - **仅使用安全标志自动允许（已验证）：** `xargs`、`file`、`sed`（只读表达式）、`sort`、`man`、`help`、`netstat`、`ps`、`base64`、`grep`、`egrep`、`fgrep`、`sha256sum`、`sha1sum`、`md5sum`、`tree`、`date`、`hostname`、`info`、`lsof`、`pgrep`、`tput`、`ss`、`fd`、`fdfind`、`aki`、`rg`、`jq`、`uniq`、`history`、`arch`、`ifconfig`、`pyright`。
    - **所有 git 只读子命令：** `git status`、`git log`、`git diff`、`git show`、`git blame`、`git branch`、`git tag`、`git remote`、`git ls-files`、`git ls-remote`、`git config --get`、`git rev-parse`、`git describe`、`git stash list`、`git reflog`、`git shortlog`、`git cat-file`、`git for-each-ref`、`git worktree list` 等。
    - **所有 gh 只读子命令：** `gh pr view`、`gh pr list`、`gh pr diff`、`gh pr checks`、`gh pr status`、`gh issue view`、`gh issue list`、`gh issue status`、`gh run view`、`gh run list`、`gh workflow list`、`gh workflow view`、`gh repo view`、`gh release view`、`gh release list`、`gh api` (GET)、`gh auth status` 等。
    - **Docker 只读子命令：** `docker ps`、`docker images`、`docker logs`、`docker inspect`。

    真实来源：`src/tools/BashTool/readOnlyValidation.ts`（`READONLY_COMMANDS`、`READONLY_NOARGS`、`READONLY_EXACT`、`COMMAND_ALLOWLIST`）和 `src/utils/shell/readOnlyCommandValidation.ts`（`GIT_READ_ONLY_COMMANDS`、`GH_READ_ONLY_COMMANDS`、`DOCKER_READ_ONLY_COMMANDS`、`RIPGREP_READ_ONLY_COMMANDS`、`PYRIGHT_READ_ONLY_COMMANDS`）。如果用户在此仓库中并且你不确定命令是否被覆盖，请 grep 这些文件而不是猜测。

5. **选择模式形式。** 使用仍然覆盖观察到的使用的最窄模式：
    - 如果用户运行许多变体（`git log`、`git log --oneline`、`git log main..HEAD`）：使用 `Bash(git log *)`——注意 `*` 之前的空格，这是前缀匹配正确工作所必需的。
    - 如果单个精确调用很常见：使用没有通配符的 `Bash(foo)`。
    - 对于 MCP：逐字使用完整工具名称（不需要通配符；它们已经很具体）。
    - 永远不要将模式扩大到与上述规则冲突的程度（没有任意代码执行，没有改变/副作用）。

6. **优先级。** 按计数降序排列。删除出现少于约 3 次的任何内容——不值得允许列表条目。将列表限制在前约 20 个，以便用户可以浏览。

7. **向用户展示优先列表** 作为带有列的 markdown 表格：排名、模式、计数、一行描述。示例：

    | # | 模式 | 计数 | 注释 |
    |---|---------|-------|-------|
    | 1 | `Bash(git status *)` | 142 | 仓库状态检查 |
    | 2 | `Bash(gh pr view *)` | 87 | PR 检查 |
    | 3 | `mcp__slack__slack_read_thread` | 54 | Slack 线程读取 |

8. **合并到 `.claude/settings.json`** 在当前项目中（不是 `~/.claude/settings.json`，不是 `.claude/settings.local.json`）。如果文件不存在则创建。保留现有键和 `permissions.allow` 中的现有条目；与已经存在的内容去重；不要删除任何内容；不要重新排序不相关字段。

9. **报告回来。** 告诉用户你添加了什么（计数 + 几个示例），允许列表中已经有什么，以及你跳过了什么以及为什么（例如"删除了 `rm` 和 `git push`——不是只读；删除了 `cat`/`ls`/`git status`——已经自动允许，不需要规则"）。

不要向 `permissions.deny` 或 `permissions.ask` 添加任何内容。不要触摸任何其他设置字段。