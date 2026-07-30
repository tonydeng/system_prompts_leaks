> **说明**：本文件为英文原文（`anthropic-cli.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Anthropic CLI (`ant`)

`ant` CLI 将每个 Claude API 资源作为 shell 子命令暴露。与 `curl` 相比：请求体由类型化标志或管道 YAML 构建而非手写 JSON，`@path` 将文件内容内联到任何字符串字段，`--transform` 用 GJSON 路径提取字段（无需 `jq`），列表端点自动分页（用 `--max-items N` 限制总结果数；`--limit` 仅设置服务器页面大小），`beta:` 前缀自动设置正确的 `anthropic-beta` 头。

## 何时使用 CLI vs SDK

**CLI 用于控制面，SDK 用于数据面。** 智能体和环境是相对静态的资源，你用 `ant` 定义、配置和调试它们，将 YAML 签入仓库、从 CI 应用、从终端检查。会话是动态的，由你的应用通过 SDK 驱动，按任务创建、流式传输事件、响应工具调用、集成到你的产品中。两者都访问同一个 API；区别在于调用在哪里，而不在于什么是可能的。

| | 控制面 → `ant` | 数据面 → SDK |
|---|---|---|
| 资源 | 智能体、环境、技能、保险库、文件 | 会话、事件 |
| 频率 | 每次部署一次 / 临时 | 每个任务 / 每个回合 |
| 存在于 | 仓库中的 `*.yaml` + CI + 终端 | 应用代码 |
| 典型调用 | `create < agent.yaml`, `update --version N`, `list`, `retrieve`, `archive`, `--debug` | `sessions.create()`, `events.stream()`, `events.send()` |

## 安装和认证

```sh
# macOS
brew install anthropics/tap/ant
xattr -d com.apple.quarantine "$(brew --prefix)/bin/ant"

# Linux / WSL — 从 github.com/anthropics/anthropic-cli/releases 选择版本
curl -fsSL "https://github.com/anthropics/anthropic-cli/releases/download/v${VERSION}/ant_${VERSION}_$(uname -s | tr A-Z a-z)_$(uname -m | sed -e s/x86_64/amd64/ -e s/aarch64/arm64/).tar.gz" \
  | sudo tar -xz -C /usr/local/bin ant

# 或从源码构建 (Go 1.22+)
go install github.com/anthropics/anthropic-cli/cmd/ant@latest
```

**认证** — CLI 以与 SDK 相同的方式解析凭据（先匹配者优先）：显式标志，然后 `ANTHROPIC_API_KEY`，然后 `ANTHROPIC_AUTH_TOKEN`，然后 `ANTHROPIC_PROFILE` 选择的或活动的配置文件，然后 Workload Identity Federation 环境变量，最后磁盘上的默认配置文件。用 `ANTHROPIC_BASE_URL` 或 `--base-url` 覆盖主机。

- **API 密钥**：在环境中设置 `ANTHROPIC_API_KEY`。
- **OAuth 配置文件**（无需管理静态密钥）：`ant auth login` 打开浏览器，交换短期令牌，并在 `$ANTHROPIC_CONFIG_DIR` 下存储配置文件（Linux/macOS 默认 `~/.config/anthropic/`，Windows 上 `%APPDATA%\Anthropic`，`configs/<profile>.json` 存设置，`credentials/<profile>.json` 存令牌）。后续的 `ant`（和 SDK）调用会自动拾取它，登录后一个裸的 `Anthropic()` 客户端即可工作，但直接读取 `ANTHROPIC_API_KEY` 的脚本不会。Claude Code 和 Claude Agent SDK 遵循相同的配置文件解析。`ant auth status` 显示哪个凭据源和配置文件胜出（它只报告状态，不要将其退出代码脚本化为健康检查）；`ant auth logout` 清除活动配置文件（`--all` 清除所有配置文件）。在没有浏览器的远程主机上，`ant auth login --no-browser` 打印授权 URL 并在终端中接受返回的代码。
- **非交互式工作负载**（CI、服务器、容器）：交互式登录用于在你自己的机器上开发，请改用 Workload Identity Federation（通过 `shared/live-sources.md` 查看认证文档）。

> **#1 认证陷阱：** 配置文件仅在未设置 API 密钥时才会被查询。一个过期的导出 `ANTHROPIC_API_KEY` 会静默覆盖每个配置文件，请求会发送到该密钥所绑定的任何组织/工作区。`ant auth status` 显示哪个源胜出；在依赖配置文件之前取消设置该密钥（或按命令：`env -u ANTHROPIC_API_KEY ant …`）。要真正**取消设置**它，`ANTHROPIC_API_KEY=""` 仍然占据其优先级槽位并以空密钥认证。同样的影子效应反过来也适用于 Claude Code：在 `ant auth login` 之后，Claude Code 可能会警告配置文件与其自己的 `/login` 凭据之间的认证冲突，保留一个（使用配置文件并在 Claude Code 中 `/logout`，或 `ant auth logout` 保留 Claude Code 自己的登录）。

**命名配置文件** — 交互式登录令牌绑定到单个组织+工作区，API 只显示属于该工作区的资源。如果你创建的智能体、会话或文件"消失了"，通常的原因是令牌绑定到了与创建它的工作区不同的工作区（`ant auth status` 显示活动工作区）。多工作区工作意味着每个工作区一个配置文件：

```sh
ant auth login --profile <name>                  # 如果配置文件不存在则创建；在浏览器中选择组织/工作区
ant auth login --profile <name> --workspace-id wrkspc_01...   # 直接绑定，跳过选择器
ant profile activate <name>                      # 切换默认配置文件
ant --profile <name> models list                 # 一次性使用；等价于：ANTHROPIC_PROFILE=<name> ant models list
ant profile list                                 # 查看
ant profile set workspace_id wrkspc_01... --profile <name>    # 编辑配置键（workspace_id, base_url, organization_id, …）
```

`ant profile set` 编辑现有配置文件的配置，它从不创建配置文件，也**不会**重新绑定已颁发的凭据；在该配置文件下重新运行 `ant auth login` 以新目标铸造令牌。将 `ANTHROPIC_PROFILE` 指向不存在的配置文件会报错，而不是回退。刷新令牌最终会硬过期（它们不会随使用而滑动），当之前正常工作的配置文件开始认证失败时，在调试其他任何东西之前重新运行 `ant auth login`。

**作用域** — 配置文件的 OAuth 作用域集在登录时请求（`--scope`）并持久化在配置文件上（`scope` 也是一个 `profile set` 配置键；与其他配置编辑一样，更改它需要重新 `ant auth login` 才能生效）。特权作用域，例如用于组织管理端点的 `org:admin`，**不在**默认作用域集中：显式传递你想要的完整集合（`ant auth login --profile admin --scope "... org:admin"`），服务器仅在你的角色确实拥有特权作用域时才授予它。因为作用域集搭在配置文件铸造的每个令牌上，所以将特权工作保留在专用配置文件上（`admin` vs `default`），日常推理在非特权配置文件上进行，用 `--profile`/`ANTHROPIC_PROFILE` 切换。查看 `ant auth login --help` 获取当前作用域列表，`ant auth status` 查看活动令牌携带的内容。

要将活动凭据传递给子进程或原始 HTTP 脚本：

```sh
# 裸访问令牌 — 用于 curl 的 Authorization 头
curl https://api.anthropic.com/v1/messages \
  -H "Authorization: Bearer $(ant auth print-credentials --access-token)" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: oauth-2025-04-20" \
  -H "content-type: application/json" \
  -d '{"model": "claude-opus-4-8", "max_tokens": 1024, "messages": [{"role": "user", "content": "Hello"}]}'

# .env 格式 — 设置 ANTHROPIC_AUTH_TOKEN（如果配置文件有则设置 ANTHROPIC_BASE_URL）。
# 输出是裸 KEY=value（无 `export`），所以用 `set -a` 为子进程自动导出：
set -a; eval "$(ant auth print-credentials --env)"; set +a
python my_script.py   # SDK 拾取 ANTHROPIC_AUTH_TOKEN
```

OAuth 令牌放在 `Authorization: Bearer` 上（不是 `x-api-key:`）**加上 `anthropic-beta: oauth-2025-04-20` 头**，将原始 curl/httpx 脚本从 API 密钥转换是头的更改，不是密钥替换。Beta 头要求是端点相关的（某些端点碰巧不需要它也能工作；`/v1/messages` 不行），始终发送它，这样在切换端点时请求不会中断。令牌是短期的，通过环境变量传递时不会自动刷新，所以对于长时间运行的脚本，在令牌过期前重新运行 `print-credentials`（`print-credentials` 本身会在需要时刷新令牌）。如果同时设置了 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN`，SDK 会同时发送两者，API 会拒绝请求，在 `eval` `--env` 输出之前取消设置 `ANTHROPIC_API_KEY`。

**脚枪：** `ant auth print-credentials` **不带标志**会打印整个凭据 JSON，而不是裸令牌，将其放入 `Authorization` 头会产生空响应或 HTTP/2 协议错误。始终使用 `--access-token` 来获取头（它始终读取命名/活动配置文件；设置的 `ANTHROPIC_API_KEY` 不会覆盖凭据打印）。

## 命令结构

```
ant <resource>[:<subresource>] <action> [flags]
```

Beta 资源（智能体、会话、环境、部署、技能、保险库、内存存储）位于 `beta:` 下，CLI 自动发送正确的 `anthropic-beta` 头，所以除非用 `--beta <header>` 覆盖，否则不要自己传递。对于自托管环境，`ant beta:worker poll/run` 和 `ant beta:environments:work stats/stop` 驱动和监控工作队列，参见 `shared/managed-agents-self-hosted-sandboxes.md`。

```sh
ant models list
ant messages create --model claude-opus-4-8 --max-tokens 1024 --message '{role: user, content: "Hello"}'
ant beta:agents retrieve --agent-id agent_01...
ant beta:sessions:events list --session-id session_01...
```

`ant --help` 列出资源；在任何子命令后追加 `--help` 查看其标志。

## 全局标志

| 标志 | 用途 |
| --- | --- |
| `--format` | `auto`（默认：TTY 时 pretty，管道时 compact）、`json`、`jsonl`、`yaml`、`pretty`、`raw`、`explore`（交互式 TUI） |
| `--transform` | 应用于响应的 GJSON 路径（列表端点上按项目应用）。`--format raw` 时不应用。 |
| `-r`, `--raw-output` | 如果转换结果是字符串，打印时不加引号（jq 语义）。与 `--transform` 配合用于标量捕获。 |
| `--max-items` | 限制自动分页列表端点返回的总结果数（不同于 `--limit`，后者是服务器页面大小）。 |
| `--format-error` / `--transform-error` | 与 `--format`/`--transform` 相同，应用于错误响应。`-r` 不适用于错误路径，使用 `--format-error yaml` 获取不带引号的错误标量。 |
| `--base-url` | 覆盖 API 主机 |
| `--debug` | 将完整 HTTP 请求 + 响应打印到 stderr（API 密钥已脱敏） |

## 输出 — `--transform` + `--format`

`--transform` 接受 [GJSON 路径](https://github.com/tidwall/gjson/blob/master/SYNTAX.md)。在列表端点上它按**项目**运行，而不是在信封上。

```sh
ant beta:agents list --transform '{id,name,model}' --format jsonl
```

**为 shell 使用提取标量：** 将 `--transform` 与 `-r` 配合（`--raw-output`，打印不带引号的字符串，jq 风格）：

```sh
AGENT_ID=$(ant beta:agents create --name "My Agent" --model '{id: claude-sonnet-5}' \
  --transform id -r)
```

## 输入 — 标志、stdin、`@file`

**标志** — 标量字段直接映射。结构化字段接受宽松 YAML 语法（不带引号的键）或严格 JSON。可重复标志构建数组（每个 `--tool`、`--event`、`--message` 追加一个元素）：

```sh
ant beta:agents create \
  --name "Research Agent" \
  --model '{id: claude-opus-4-8}' \
  --tool '{type: agent_toolset_20260401}' \
  --tool '{type: custom, name: search_docs, input_schema: {type: object, properties: {query: {type: string}}}}'
```

**Stdin** — 管道传输完整 JSON 或 YAML 体。与标志合并；冲突时标志优先（对于数组字段，任何标志**完全替换** stdin 数组，不会追加）。引用 heredoc 分隔符（`<<'YAML'`）以禁用体内部的 shell 展开：

```sh
ant beta:agents create <<'YAML'
name: Research Agent
model: claude-opus-4-8
system: |
  You are a research assistant. Cite sources for every claim.
tools:
  - type: agent_toolset_20260401
YAML
```

**`@file` 引用** — 将文件内容内联到任何字符串值字段。在结构化标志值内部，引用路径。二进制文件自动 base64 编码；用 `@file://`（文本）或 `@data://`（base64）强制。将字面的前导 `@` 转义为 `\@`。

```sh
ant beta:agents create --name "Researcher" --model '{id: claude-sonnet-5}' --system @./prompts/researcher.txt

ant messages create --model claude-opus-4-8 --max-tokens 1024 \
  --message '{role: user, content: [
    {type: document, source: {type: base64, media_type: application/pdf, data: "@./scan.pdf"}},
    {type: text, text: "Extract the text from this scanned document."}
  ]}' \
  --transform 'content.0.text' -r
```

原生接受文件路径的标志（例如 `beta:files upload` 上的 `--file`）接受不带 `@` 的裸路径。

## 版本控制的 Managed Agents 资源

这是定义智能体和环境的推荐流程，将 YAML 签入仓库并通过 `create`（首次）/ `update`（之后）同步。参见 `shared/managed-agents-core.md` 获取字段参考。

```yaml
# summarizer.agent.yaml
name: Summarizer
model: claude-sonnet-5
system: |
  You are a helpful assistant that writes concise summaries.
tools:
  - type: agent_toolset_20260401
```

```sh
# 创建（一次性）— 捕获 ID
AGENT_ID=$(ant beta:agents create < summarizer.agent.yaml --transform id -r)

# 更新 (CI) — 需要 ID + 当前版本（乐观锁）
ant beta:agents update --agent-id "$AGENT_ID" --version 1 < summarizer.agent.yaml
```

环境使用相同的模式（`ant beta:environments create|update < env.yaml`），然后用两个 ID 启动会话：

```sh
ant beta:sessions create --agent "$AGENT_ID" --environment-id "$ENV_ID" --title "Task"
ant beta:sessions:events send --session-id "$SID" \
  --event '{type: user.message, content: [{type: text, text: "Summarize X"}]}'
ant beta:sessions:events list --session-id "$SID" --transform 'content.0.text' -r
ant beta:sessions:events stream --session-id "$SID"   # 实时事件流
```

### 交互式会话循环（先流后发）

`ant beta:sessions:events stream` 仅传递在流打开*之后*发出的事件，所以在发送启动消息*之前*打开它以避免丢失早期事件。使用进程替换将流保持在文件描述符上，发送，然后读取：

```sh
exec {stream}< <(ant beta:sessions:events stream --session-id "$SID" \
  --transform '{type,text:content.#(type=="text").text,err:error.message}' --format yaml)

ant beta:sessions:events send --session-id "$SID" > /dev/null <<'YAML'
events:
  - type: user.message
    content:
      - type: text
        text: Summarize the repo README
YAML

type=
while IFS= read -r -u "$stream" line; do
  case "$line" in
    type:\ session.status_idle) break ;;
    type:\ session.error)
      IFS= read -r -u "$stream" next || next=
      case "$next" in err:\ *) msg=${next#err: } ;; *) msg=unknown ;; esac
      printf '\n[Error: %s]\n' "$msg"; break ;;
    type:\ *) type=${line#type: } ;;
    text:*)
      [[ $type == agent.message ]] || continue
      val=${line#text: }
      case "$val" in '|-'|'|') ;; *) printf '%s' "$val" ;; esac ;;
    \ \ *)
      if [[ $type == agent.message ]]; then printf '%s\n' "${line#  }"; fi ;;
  esac
done
exec {stream}<&-
```

这适用于交互式探索和演示。对于需要响应 `agent.tool_use` / `agent.custom_tool_use` 事件、在断开后重连或对 `events.list` 去重的应用代码，请使用 SDK，参见 `shared/managed-agents-client-patterns.md`。

## 脚本模式

列表端点上的 `--transform id -r` 每行输出一个裸 ID，用 `xargs` 组合，或用 `--max-items N` 限制结果集而无需通过 `head` 管道：

```sh
FIRST=$(ant beta:agents list --transform id -r --max-items 1)
ant beta:agents:versions list --agent-id "$FIRST" --transform '{version,created_at}' --format jsonl
```

错误塑形镜像成功路径（注意：`-r` 不适用于错误输出，这里用 `--format-error yaml` 获取不带引号的标量）：

```sh
ant beta:agents retrieve --agent-id bogus --transform-error error.message --format-error yaml 2>&1
```

Shell 补全：`ant @completion {zsh|bash|fish|powershell}`。

如需完整的、始终最新的参考（包括每个端点的标志），WebFetch `shared/live-sources.md` 中的 **Anthropic CLI** URL。
