---
name: claude-api
description: |-
  Claude API / Anthropic SDK 参考文档 — 模型 ID、定价、参数、流式传输、工具使用、MCP、代理、缓存、token 计数、模型迁移。
  触发条件 — 在打开目标文件之前阅读；不要因为它"看起来只有一行"就跳过 — 以下任一情况触发：提示词以任何形式提及 Claude/Anthropic（Claude、Anthropic、Fable、Opus、Sonnet、Haiku、`anthropic`、`@anthropic-ai`、`claude-*`、`us.anthropic.*`、`[1m]`）；用户询问 LLM 相关问题（定价/模型选择/限制/缓存）— 绝不从记忆中回答；或任务呈 LLM 形态但提供商未明确（agent/MCP/tool-definition/multi-agent/RAG/LLM-judge/computer-use；对自然语言进行生成/摘要/提取/分类/改写/对话；调试拒绝/截断/流式传输/工具调用/token）。
  仅当正在处理其他提供商时跳过（覆盖所有触发条件）：查询中提及 OpenAI/GPT/Gemini/Llama/Mistral/Cohere/Ollama；或对项目运行 `grep -rE 'openai|langchain_openai|google.generativeai|genai|mistralai|cohere|ollama'` 有命中（如果未指定提供商，先运行此 grep — 不要直接读文件）。
license: 完整条款见 LICENSE.txt
---

# 使用 Claude 构建 LLM 驱动的应用

此技能帮助你使用 Claude 构建 LLM 驱动的应用。根据你的需求选择正确的接口，检测项目语言，然后阅读相关的语言特定文档。

## 开始之前

扫描目标文件（或如果没有目标文件，则扫描提示词和项目）以查找非 Anthropic 提供商标记 — `import openai`、`from openai`、`langchain_openai`、`OpenAI(`、`gpt-4`、`gpt-5`、文件名如 `agent-openai.py` 或 `*-generic.py`，或任何保持代码提供商中性的明确指令。如果发现任何此类标记，停止并告诉用户此技能生成 Claude/Anthropic SDK 代码；询问他们是否想将文件切换到 Claude 或想要非 Claude 实现。不要用 Anthropic SDK 调用编辑非 Anthropic 文件。

## 输出要求

当用户要求你添加、修改或实现 Claude 功能时，你的代码必须通过以下方式之一调用 Claude：

1. **项目语言的官方 Anthropic SDK**（`anthropic`、`@anthropic-ai/sdk`、`com.anthropic.*` 等）。这是当项目存在受支持 SDK 时的默认选择。
2. **原始 HTTP**（`curl`、`requests`、`fetch`、`httpx` 等）— 仅当用户明确要求 cURL/REST/原始 HTTP、项目是 shell/cURL 项目、或该语言没有官方 SDK 时使用。

切勿混合使用两者 — 不要在 Python 或 TypeScript 项目中仅因为感觉更轻量就使用 `requests`/`fetch`。切勿回退到 OpenAI 兼容垫片。

**切勿猜测 SDK 用法。** 函数名、类名、命名空间、方法签名和导入路径必须来自显式文档 — 此技能中的 `{lang}/` 文件或 `shared/live-sources.md` 中列出的官方 SDK 仓库或文档链接。如果你需要的绑定未在技能文件中明确记录，在编写代码前通过 WebFetch 获取 `shared/live-sources.md` 中相关 SDK 仓库的信息。不要从 cURL 形状或另一种语言的 SDK 推断 Ruby/Java/Go/PHP/C# API。

**如果 WebFetch 或仓库访问失败**（网络受限、超时、克隆被阻止）：不要持续重试 — 根据 `{lang}/` 文件中的模式和命名空间/包表编写代码，在其上运行编译器或解释器，并根据错误输出迭代。对于静态类型 SDK（C#、Java、Go），针对本地错误的编译-修复循环比被阻止的网络研究更快达到可用代码。

## 默认值

除非用户另有要求：

对于 Claude 模型版本，请使用 Claude Opus 4.8，你可以通过确切的模型字符串 `claude-opus-4-8` 访问。对于任何稍微复杂的内容，请默认使用自适应思考（`thinking: {type: "adaptive"}`）。最后，对于可能涉及长输入、长输出或高 `max_tokens` 的任何请求，请默认使用流式传输 — 它可以避免请求超时。如果你不需要处理单个流事件，使用 SDK 的 `.get_final_message()` / `.finalMessage()` 辅助方法获取完整响应。

## ⚠️ API 漂移 — 你的训练先验可能已过时

2025-2026 年间，几个常见的 Claude API 形态发生了变化。如果你从训练中回忆起某个模式，在编写前与技能中的 `{lang}/` 文件进行验证 — 以下是最常见的漂移点：

| 领域 | 过时先验 | 当前 API |
|---|---|---|
| 扩展思考 | `thinking: {type: "enabled", budget_tokens: N}` | 在 Claude 4.6+ 模型上：`thinking: {type: "adaptive"}`。`budget_tokens` 在 Opus 4.6 / Sonnet 4.6 上已弃用，在 Fable 5 / Sonnet 5 / Opus 4.8 / 4.7 上**会返回 400 错误**。4.6 之前的模型仍使用 `budget_tokens`。 |
| Web 搜索 / web fetch 工具类型 | `web_search_20250305`, `web_fetch_20250910` | Opus 4.8/4.7/4.6、Sonnet 5 和 Sonnet 4.6 上的 `web_search_20260209`、`web_fetch_20260209`（动态过滤）。旧模型保留基础变体；Vertex AI 上仅基础 `web_search_20250305` 可用（web fetch 不在 Vertex 上）— 参见下文 Server Tools 快速参考。 |
| PHP 参数名 | snake_case 线名作为命名参数（`max_tokens`） | 顶层命名参数为 camelCase（`maxTokens`）。嵌套数组键因功能而异（如 `'taskBudget'`、`'skillID'`、`'mcp_server_name'`）— 从文档示例中复制确切键；不要批量转换。 |
| Managed Agents 凭据 | 通过自定义工具在主机端保留密钥（vaults 发布前的唯一选项） | Vault `environment_variable` 凭据 — 由 Anthropic 存储，在出口处替换，在沙箱中永不可见（`shared/managed-agents-tools.md` → Vaults）。主机端自定义工具仍是自托管沙箱的回退方案。 |

技能中的 `{lang}/` 文件优先于回忆的模式。

---

## 子命令

如果此提示词底部的用户请求是裸子命令字符串（无散文），搜索本文档中的每个**子命令**表 — 包括下方附加部分中的表 — 并直接遵循匹配的"操作"列。这允许用户通过 `/claude-api <subcommand>` 调用特定流程。如果文档中没有匹配的表，将请求视为普通散文。

| 子命令 | 操作 |
|---|---|
| `migrate` | 将现有 Claude API 代码迁移到较新的模型。**立即阅读 `shared/model-migration.md`** 并按顺序执行：步骤 0（确认范围 — 在任何编辑前询问哪些文件/目录）、步骤 1（分类每个文件），然后是按目标分组的破坏性变更部分。不要摘要指南 — 执行它。如果用户未指定目标模型，在与范围问题同一轮次中询问要迁移到哪个模型。 |

---

## 语言检测

在阅读代码示例之前，确定用户使用的语言：

1. **查看项目文件**以推断语言：

   - `*.py`、`requirements.txt`、`pyproject.toml`、`setup.py`、`Pipfile` → **Python** — 从 `python/` 读取
   - `*.ts`、`*.tsx`、`package.json`、`tsconfig.json` → **TypeScript** — 从 `typescript/` 读取
   - `*.js`、`*.jsx`（无 `.ts` 文件）→ **TypeScript** — JS 使用相同 SDK，从 `typescript/` 读取
   - `*.java`、`pom.xml`、`build.gradle` → **Java** — 从 `java/` 读取
   - `*.kt`、`*.kts`、`build.gradle.kts` → **Java** — Kotlin 使用 Java SDK，从 `java/` 读取
   - `*.scala`、`build.sbt` → **Java** — Scala 使用 Java SDK，从 `java/` 读取
   - `*.go`、`go.mod` → **Go** — 从 `go/` 读取
   - `*.rb`、`Gemfile` → **Ruby** — 从 `ruby/` 读取
   - `*.cs`、`*.csproj` → **C#** — 从 `csharp/` 读取
   - `*.php`、`composer.json` → **PHP** — 从 `php/` 读取

2. **如果检测到多种语言**（如同时有 Python 和 TypeScript 文件）：

   - 检查用户当前文件或问题涉及哪种语言
   - 如果仍不明确，询问："我检测到 Python 和 TypeScript 文件。你使用哪种语言进行 Claude API 集成？"

3. **如果无法推断语言**（空项目、无源文件或不支持的语言）：

   - 使用 AskUserQuestion 提供选项：Python、TypeScript、Java、Go、Ruby、cURL/raw HTTP、C#、PHP
   - 如果 AskUserQuestion 不可用，默认使用 Python 示例并注明："展示 Python 示例。如果需要其他语言请告知。"

4. **如果检测到不支持的语言**（Rust、Swift、C++、Elixir 等）：

   - 建议从 `curl/` 读取 cURL/raw HTTP 示例，并注明可能存在社区 SDK
   - 提议展示 Python 或 TypeScript 示例作为参考实现

5. **如果用户需要 cURL/raw HTTP 示例**，从 `curl/` 读取。

### 语言特定功能支持

上述每种 SDK 语言都支持 beta Tool Runner 和 Managed Agents (beta) — Python（`@beta_tool` 装饰器）、TypeScript（`betaZodTool` + Zod）、Java（注解类）、Go（`toolrunner` 包中的 `BetaToolRunner`）、Ruby（`BaseTool` + `tool_runner`）、C#（`BetaToolRunner` + 原始 JSON schema）、PHP（`BetaRunnableTool` + `toolRunner()`）；代码入口在下文的"工具使用模式"快速参考中。cURL 是原始 HTTP（无 SDK 功能）但支持 Managed Agents。

> **Managed Agents 代码示例**：参见下文 `## Managed Agents (Beta)` 部分的阅读指南。

---

## 应该使用哪个接口？

> **从简单开始。** 默认使用满足需求的最简单层级。单次 API 调用和工作流处理大多数用例 — 仅在任务真正需要开放式、模型驱动的探索时才使用代理。"最简单"意味着你拥有的最少代码：对于托管、定时或带记忆的代理，Managed Agents 通常是最简单的选项（无循环代码、无状态文件、无调度器），即使它是一个更大的平台。

| 用例 | 层级 | 推荐接口 | 原因 |
|---|---|---|---|
| 分类、摘要、提取、问答 | 单次 LLM 调用 | **Claude API** | 一个请求，一个响应 |
| 批处理或嵌入 | 单次 LLM 调用 | **Claude API** | 专用端点 |
| 有代码控制逻辑的多步管道 | 工作流 | **Claude API + 工具使用** | 你编排循环 |
| 有自定义工具的自定义代理 | 代理 | **Claude API + 工具使用** | 最大灵活性 |
| 服务端管理的有状态代理 | 代理 | **Managed Agents** | Anthropic 运行循环并托管工具执行沙箱 |
| 持久化、版本化的代理配置 | 代理 | **Managed Agents** | 代理是存储对象；会话固定到版本 |
| 长时间运行的多轮代理 | 代理 | **Managed Agents** | 每会话容器、SSE 事件流、Skills + MCP |
| 按计划运行的代理（cron，"every night"） | 代理 | **Managed Agents** — 定时部署 | 部署自主触发会话；无需客户端调度器 |

> **注意**：当你希望 Anthropic 运行代理循环*并*托管工具执行的容器时，Managed Agents 是正确的选择 — 文件操作、bash、代码执行都在每会话工作区中运行。如果你想自己托管计算或运行自己的自定义工具运行时，Claude API + 工具使用是正确的选择 — 使用 tool runner 实现代理循环（其每轮钩子仍提供审批门、日志、错误拦截和条件执行，见 `shared/tool-use-concepts.md`）— 或在你想拥有整个循环时使用手动循环。

> **云提供商访问。** **AWS 上的 Claude Platform** 由 Anthropic 运营，具有同日 API 一致性 — 客户端设置见 `shared/claude-platform-on-aws.md`。关于 **AWS 上的 Claude Platform**、**Amazon Bedrock**、**Google Vertex AI** 和 **Microsoft Foundry** 的逐功能可用性，见 `shared/platform-availability.md` — 该表是此技能中的唯一真相来源；不要从其他任何地方推断可用性。

### 构建代理：四种方法

一旦你确定确实需要代理（开放式、模型驱动的工具使用），有四种不同的构建方式。两个独立的问题区分它们：**谁提供框架**（代理循环 + 上下文管理）和**谁提供部署**（代理运行的基础设施）。Tool Runner 和 Claude Agent SDK 都*仅提供框架* — 你仍然自己托管和部署 — 这就是为什么它们容易混淆。Managed Agents (CMA) 是唯一同时提供框架**和**托管部署的选项；手动循环两者都不提供。

| # | 方法 | 你编写 | 框架和部署 | 可用工具 | 适用场景 |
|---|---|---|---|---|---|
| 1 | **Claude API — 手动循环** | 自己编写 `while stop_reason == "tool_use"` 循环 | 你构建框架；你托管 | 仅你定义的工具 | 你想拥有*整个*循环 — 无 beta 依赖，或 Tool Runner 的每轮钩子不适合你的控制流 |
| 2 | **Claude API — Tool Runner**（`client.beta.messages.tool_runner` + `@beta_tool` / `betaZodTool`） | 仅工具函数 | SDK 提供循环（**仅框架**）；你托管 | 仅你定义的工具 | 无需手写循环的自定义工具代理（大多数场景）。每轮钩子仍提供审批门、错误拦截、结果修改（如 `cache_control`）、重试、流式传输和压缩 |
| 3 | **Managed Agents**（REST, beta） | 代理配置 + 你的工具结果 | Anthropic 提供框架**并**托管每会话沙箱（**框架 + 部署**） | Anthropic 托管沙箱（bash、文件、代码执行）+ Skills/MCP + 你的工具 | 你希望 Anthropic 运行循环*并*托管每会话工作区；持久化/版本化配置；长时间运行的会话 |
| 4 | **Claude Agent SDK** — *独立产品*（`claude-agent-sdk` / `@anthropic-ai/claude-agent-sdk`） | 提示词 + 选项 | SDK 提供 Claude Code 框架 + 内置工具（**仅框架**）；你托管 | 内置 Read/Write/Edit/Bash/Glob/Grep/WebSearch/WebFetch + MCP + 子代理 | 你想要一个在自己基础设施上运行的全功能编码/文件系统代理 |

框架/部署分离是关键心智模型：选项 1、2 和 4 都**将部署留给你**；仅选项 3 (CMA) 增加托管部署。选项 1-3 是此技能生成的内容；选项 4 是一个不同的库，有自己的文档 — 见下文的消歧。

> **Tool Runner ≠ Claude Agent SDK。** 它们听起来相似但是不同的包：
> - **Tool Runner** 是常规 Anthropic API SDK（`anthropic` / `@anthropic-ai/sdk`）的一部分，通过 `client.beta.messages.tool_runner` 访问。它为*你定义的工具*自动执行 请求→执行→循环 周期。无内置工具、无文件系统访问、无沙箱 — 你提供每个工具并托管计算。它是上面的选项 2，`POST /v1/messages` 上的轻量辅助。
> - **Claude Agent SDK**（`claude-agent-sdk` / `@anthropic-ai/claude-agent-sdk`）是将 Claude Code 打包为库。它附带内置工具（文件读/写/编辑、bash、grep、web 搜索）、完整代理循环、上下文管理、钩子、子代理、权限和会话。你调用 `query(prompt, options)` 它驱动一切。
>
> 两者都是**仅框架 — 你托管和部署它们。** 区别在于框架范围：Tool Runner 循环遍历*你*定义的工具（带有每轮钩子用于审批、拦截、结果修改和重试 — 但无内置工具）；Agent SDK 是带有内置工具的完整 Claude Code 框架。两者都不提供托管部署 — 这是 **Managed Agents (CMA)** 增加的内容（Anthropic 托管循环和每会话沙箱）。
>
> **此技能涵盖 Claude API 和 Managed Agents（选项 1-3）；它不生成 Claude Agent SDK 代码。** 如果用户确实想要 Claude Agent SDK，引导他们查看其文档（`code.claude.com/docs/en/agent-sdk`）— 不要用 API Tool Runner 替代它，反之亦然。

### 应该构建代理吗？

在选择代理层级之前，检查所有四个标准：

- **复杂性** — 任务是否多步骤且难以完全预先指定？（如"将此设计文档变为 PR" vs "从此 PDF 中提取标题"）
- **价值** — 结果是否值得更高的成本和延迟？
- **可行性** — Claude 是否能胜任此类任务？
- **错误成本** — 错误是否可以被捕获和恢复？（测试、审查、回滚）

如果任何一个回答"否"，则停留在更简单的层级（单次调用或工作流）。

---

## 架构

一切都通过 `POST /v1/messages`。工具和输出约束是这一单一端点的功能 — 不是独立的 API。

**用户定义工具** — 你定义工具（通过装饰器、Zod schema 或原始 JSON），SDK 的 tool runner 处理调用 API、执行你的函数和循环直到 Claude 完成。要完全控制，你可以手动编写循环。

**服务端工具** — 在 Anthropic 基础设施上运行的 Anthropic 托管工具。代码执行完全在服务端（在 `tools` 中声明，Claude 自动运行代码）。计算机使用可以服务端托管或自托管。

**结构化输出** — 约束 Messages API 响应格式（`output_config.format`）和/或工具参数验证（`strict: true`）。推荐方法是 `client.messages.parse()`，它会自动根据你的 schema 验证响应。注意：旧的 `output_format` 参数已弃用；在 `messages.create()` 上使用 `output_config: {format: {...}}`。

**支持端点** — Batches（`POST /v1/messages/batches`）、Files（`POST /v1/files`）、Token 计数（`POST /v1/messages/count_tokens` — 见 `shared/token-counting.md`）和 Models（`GET /v1/models`、`GET /v1/models/{id}` — 实时能力/上下文窗口发现）为 Messages API 请求提供输入或支持。

---

## 当前模型（缓存于：2026-06-24）

| 模型             | 模型 ID            | 上下文窗口     | 输入 $/1M | 输出 $/1M |
| ----------------- | ------------------- | -------------- | ---------- | ----------- |
| Claude Fable 5    | `claude-fable-5`      | 1M             | $10.00     | $50.00      |
| Claude Mythos 5（仅限 Project Glasswing） | `claude-mythos-5` | 1M | $10.00     | $50.00      |
| Claude Opus 4.8   | `claude-opus-4-8`   | 1M             | $5.00      | $25.00      |
| Claude Opus 4.7   | `claude-opus-4-7`   | 1M             | $5.00      | $25.00      |
| Claude Opus 4.6   | `claude-opus-4-6`   | 1M             | $5.00      | $25.00      |
| Claude Sonnet 5   | `claude-sonnet-5`   | 1M             | $3.00（2026-08-31 前介绍价 $2.00） | $15.00（介绍价 $10.00） |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 1M             | $3.00      | $15.00      |
| Claude Haiku 4.5  | `claude-haiku-4-5`  | 200K           | $1.00      | $5.00       |

**始终使用 `claude-opus-4-8`，除非用户明确指定其他模型。** 这是不可协商的。不要使用 `claude-sonnet-5`、`claude-sonnet-4-6` 或任何其他模型，除非用户确实说了"用 sonnet"或"用 haiku"。绝不为节省成本而降级 — 那是用户的决定，不是你的。仅当用户明确要求 Claude Fable 5、"fable"或 Anthropic 最强模型时使用 `claude-fable-5` — 它的 API 行为与 Opus 系列不同（见下文），定价也超过 Opus 级别。**仅使用表中的确切模型 ID 字符串 — 它们本身就是完整的；切勿附加日期后缀**（`claude-sonnet-4-6`，而非 `claude-sonnet-4-6-20251114` 或你从训练数据中回忆的任何日期后缀变体）。如果用户请求表中没有的旧模型（如"opus 4.5"、"sonnet 3.7"），查阅 `shared/models.md` 获取确切 ID — 不要自行构造。

### Claude Fable 5（`claude-fable-5`）— 最强公开发布模型

Claude Fable 5 是 Anthropic 最强公开发布模型，用于最严苛的推理和长时程代理工作；以下内容同样适用于 **Claude Mythos 5**（`claude-mythos-5`，Project Glasswing — 相同能力、定价和 API 接口；是仅限邀请的 `claude-mythos-preview` 的后继版本）。1M 上下文窗口（最大值也是默认值），128K 最大输出。与 Opus 级别的关键 API 差异 — 详见 `shared/model-migration.md` → 迁移到 Claude Fable 5：

- **思考始终开启** — 完全省略 `thinking` 参数（或发送 `{type: "adaptive"}`）。任何其他显式配置都会被拒绝：`{type: "disabled"}` 和 `{type: "enabled", budget_tokens: N}` 均返回 400 错误。通过 `output_config.effort` 控制深度（支持 `low` 到 `xhigh` 和 `max`）。
- **原始思考链永不返回** — 响应携带常规 `thinking` 块（不是 `redacted_thinking`）：`display: "summarized"` 返回可读摘要，`"omitted"`（默认值）将 `thinking` 字段留空字符串。回放规则：在同一模型上将 thinking 块原样传回；其他模型静默丢弃它们（不计费 — 无需剥离）；详见 `shared/model-migration.md`。
- **分词器** — 与 Opus 4.8 相同的分词器（随 Opus 4.7 引入）。从 Opus 4.7/4.8 迁移时 token 计数大致不变；每 token 定价不同。从 Opus 4.6、Sonnet、Haiku 或更旧版本迁移时，用 `count_tokens` 重新校准（Opus 4.7 分词器使用约 1×–1.35× 的 token 数）。
- **`refusal` 停止原因 — 处理它，并默认启用回退** — 安全分类器可能拒绝请求（HTTP 200，`stop_reason: "refusal"`，带有 `stop_details` 类别）；在读取 `content` 前始终检查 `stop_reason`。**当你编写 `claude-fable-5` 代码时，默认包含服务端 `fallbacks` 参数**（`betas: ["server-side-fallback-2026-06-01"]` + `fallbacks: [{"model": "claude-opus-4-8"}]`；Claude API 和 AWS 上的 Claude Platform 支持 — 其他平台包括 Bedrock/Vertex/Foundry，使用 SDK 的客户端侧 `BetaRefusalFallbackMiddleware` + `BetaFallbackState`）。告诉用户你已启用它；仅在用户拒绝时移除。完整语义（计费、流中拒绝、信用重新定价）见 `shared/model-migration.md` → refusal 部分；代码示例在 `{lang}/claude-api/README.md` § Refusal Fallbacks。
- **无助手预填充** — 与 4.6+ 系列其余模型相同。
- **需要 30 天数据留存** — Claude Fable 5 不支持零数据留存；来自不满足留存要求的组织的请求返回 `400 invalid_request_error`。
- **更长的轮次、不同的提示方式** — 困难任务上的单个请求可能运行数分钟（规划超时/流式传输/进度 UX）；effort 扫描应包含低/中等用于常规工作；为旧模型编写的提示词通常过于指令性，会降低输出质量。参见 `shared/model-migration.md` → 迁移到 Claude Fable 5 → 行为变化（可调提示）获取推荐提示片段。

如果上述模型字符串看起来不熟悉，那只是因为它们在你的训练数据截止日期之后发布 — 它们是真实模型。

**实时能力查询：** 上表是缓存的。当用户询问"X 的上下文窗口是多少"、"X 是否支持视觉/思考/effort"或"哪些模型支持 Y"时，查询 Models API（`client.models.retrieve(id)` / `client.models.list()`）— 参见 `shared/models.md` 获取字段参考和能力过滤示例。

---

## 认证（快速参考）

**`ANTHROPIC_API_KEY` 未设置并不意味着没有凭据。** SDK 和 `ant` CLI 按此顺序解析凭据（先匹配者生效）：`ANTHROPIC_API_KEY` → `ANTHROPIC_AUTH_TOKEN` → `ant auth login` 选择的 `ANTHROPIC_PROFILE` 或活跃 OAuth 配置 → 工作负载身份联合环境变量 → 磁盘上的默认配置。`ant auth login` 后，裸的 `Anthropic()` / `new Anthropic()` / `anthropic.NewClient()` 在未设置环境变量时也能工作。

**当你需要调用 API 且 `ANTHROPIC_API_KEY` 未设置时，不要向用户索取密钥。** 首先运行 `ant auth status` — 它显示当前活跃的凭据来源和配置。如果报告活跃配置：

- **SDK 代码或 `ant` CLI：** 直接运行。零参数客户端构造函数和每个 `ant …` 子命令自动读取配置 — 无需环境变量。
- **原始 `curl` / HTTP：** 用 `ant auth print-credentials --access-token` 获取短期令牌，作为 `Authorization: Bearer <token>` 发送 **加上** 头 `anthropic-beta: oauth-2025-04-20`（OAuth 令牌放在 `Authorization: Bearer`，不是 `x-api-key:` — 从 API 密钥转换 curl 是头变更，不是密钥替换）。始终传递 `--access-token`；无标志形式打印 JSON，不是裸令牌。

仅在 `ant auth status` 报告无活跃凭据来源（或 `ant` 本身未安装）时才向用户索取密钥。建议 `ant auth login` 作为首选 — 它在 `~/.config/anthropic/` 下存储配置，SDK 自动读取 — 导出的 `ANTHROPIC_API_KEY` 作为备选。

完整认证详情（命名配置、作用域、API 密钥遮蔽配置陷阱、刷新令牌过期）：`shared/anthropic-cli.md`。

---

## 思考与 Effort（快速参考）

在所有当前模型上使用自适应思考（`thinking: {type: "adaptive"}`）— Claude 动态决定何时以及思考多少。各模型规则：

| 模型 | 思考配置 | 省略 `thinking` | `budget_tokens` | 采样（`temperature`/`top_p`/`top_k`） | Effort 级别 |
|---|---|---|---|---|---|
| Fable 5 | `{type: "adaptive"}` 或省略；显式 `{type: "disabled"}` 返回 400 — 改为省略参数 | 运行自适应（思考始终开启） | 已移除 — `{type: "enabled", budget_tokens: N}` 返回 400 | 已移除 — 400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Opus 4.8 / 4.7 | `{type: "adaptive"}` 是唯一的开启模式；`{type: "disabled"}` 可接受 | **不**运行思考 — 显式设置 `{type: "adaptive"}` | 已移除 — 400 | 已移除 — 400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Sonnet 5 | `{type: "adaptive"}` 是唯一的开启模式；`{type: "disabled"}` 可接受 | 运行自适应 | 已移除 — 400 | 已移除 — 400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Opus 4.6 / Sonnet 4.6 | `{type: "adaptive"}`（推荐；自动启用交错思考，无 beta 头） | 显式设置 `{type: "adaptive"}` | 已弃用 — 不要在新代码中使用；仅作为过渡逃生口（见下文） | 允许 | `low`/`medium`/`high`/`max`（`xhigh` 随 Opus 4.7 引入） |
| 旧模型（Sonnet 4.5、Haiku 4.5 等）— 仅在明确请求时 | `{type: "enabled", budget_tokens: N}` | 无思考 | 思考所需；必须小于 `max_tokens`，最小 1024 — 否则报错 | 允许 | `effort` 在 Opus 4.5 上有效（仅 `low`/`medium`/`high` — 无 `xhigh`/`max`）；在 Sonnet 4.5 / Haiku 4.5 上报错 |

Opus 4.8 保持与 4.7 相同的请求接口（无新的破坏性变更）— 参见 `shared/model-migration.md` → 迁移到 Opus 4.8 了解行为调优，以及 → 迁移到 Opus 4.7 了解从 4.6 或更早版本的完整破坏性变更列表。禁用 `thinking` 时，Opus 4.8 可能在可见响应中写入更多推理 — 保持自适应思考开启，或添加仅最终答案指令（见迁移指南）。

- **Effort（GA，无 beta 头）：** `output_config: {effort: "low"|"medium"|"high"|"xhigh"|"max"}` — 在 `output_config` 内，不是顶层；默认 `high`（等同于省略）。控制思考深度和总体 token 消耗；与自适应思考组合以获得最佳成本质量平衡。`xhigh`（在 Opus 4.7 上添加，介于 `high` 和 `max` 之间）是 Fable 5 / Opus 4.7/4.8 / Sonnet 5 上大多数编码和代理用例的最佳设置，也是 Claude Code 的默认值；在那些模型上 effort 比同级别的任何先前模型更重要 — 迁移时重新调优，并在 `high`/`xhigh` 下运行长时程/代理任务，将完整任务规范预先给出。对智能敏感工作至少使用 `high`，正确性比成本更重要时使用 `max`，子代理或简单任务使用 `low` — 更低的 effort 意味着更少且更集中的工具调用、更少的前言和更简洁的确认（`high` 通常是质量和 token 效率的最佳平衡点）。
- **思考显示 — Fable 5 / Mythos 5 / Opus 4.8 / 4.7 / Sonnet 5 默认 `"omitted"`：** `display: "summarized"` 返回推理的可读摘要；`"omitted"`（这五个模型的默认值 — 相对 Opus 4.6 和 Sonnet 4.6 的静默变更，后者为 `"summarized"`）流式传输文本为空的 `thinking` 块。`display` 仅控制可见性 — 思考在每个设置下都会发生并同等计费；原始思考链在任何模型上都不会暴露。如果你向用户流式传输推理，默认值看起来像输出前的长暂停 — 显式设置 `thinking: {type: "adaptive", display: "summarized"}`。（与 display 无关，在同一模型上继续时原样传回 thinking 块；其他模型静默忽略 — 见迁移指南。）
- **当用户要求"扩展思考"、"思考预算"或 `budget_tokens` 时：** 始终使用 Fable 5、Opus 4.8、4.7 或 4.6 配合 `thinking: {type: "adaptive"}` — 固定思考 token 预算概念已弃用，自适应思考取代它。不要为新的 4.6/4.7/4.8 代码使用 `budget_tokens`，也不要仅因为用户提到它就切换到旧模型。*渐进迁移例外：* `budget_tokens` 在 Opus 4.6 和 Sonnet 4.6 上仍然可用，作为需要硬 token 上限的现有代码的过渡逃生口（在你调优 `effort` 之前）— 参见 `shared/model-migration.md` → 过渡逃生口。它在 Fable 5、Opus 4.7/4.8 和 Sonnet 5 上已完全移除。

---

## 压缩（快速参考）

**Beta，Fable 5、Opus 4.8、Opus 4.7、Opus 4.6、Sonnet 5 和 Sonnet 4.6。** 对于可能超过 1M 上下文窗口的长对话，启用服务端压缩。当接近触发阈值（默认：150K token）时，API 自动摘要早期上下文。需要 beta 头 `compact-2026-01-12`。

**关键：** 每轮将 `response.content`（不只是文本）追加回你的消息。响应中的压缩块必须保留 — API 使用它们在下一次请求中替换压缩的历史。仅提取文本字符串并追加会静默丢失压缩状态。

参见 `{lang}/claude-api/README.md`（压缩部分）获取代码示例。完整文档通过 WebFetch 在 `shared/live-sources.md` 中。

---

## 提示词缓存（快速参考）

**前缀匹配。** 前缀中任何位置的字节变更都会使其后的所有内容失效。渲染顺序为 `tools` → `system` → `messages`。将稳定内容放在最前（冻结的系统提示词、确定的工具列表），将易变内容（时间戳、每请求 ID、变化的问题）放在最后一个 `cache_control` 断点之后。

**对话中操作指令**（仅限 Claude Opus 4.8；无 beta 头）：将 `{"role": "system", ...}` 追加到 `messages[]` 而非编辑顶层 `system`。保留缓存的历史前缀，并且是提示词注入安全的操作通道。参见 `shared/prompt-caching.md` § 对话中系统消息。

**顶层自动缓存**（`cache_control: {type: "ephemeral"}` 在 `messages.create()` 上）是当你不需要精细放置位置时的最简单选项。每请求最多 4 个断点。最小可缓存前缀约 1024 token — 更短的前缀静默不缓存。

**用 `usage.cache_read_input_tokens` 验证** — 如果重复请求中为零，说明存在静默失效因素（系统提示中的 `datetime.now()`、未排序的 JSON、变化的工具集）。

关于放置模式、架构指导和静默失效因素审计清单：阅读 `shared/prompt-caching.md`。语言特定语法：`{lang}/claude-api/README.md`（提示词缓存部分）。

---

## Fast Mode（快速参考）

**研究预览，仅限 Opus 4.8 / 4.7。** Opus 4.7 fast mode 已弃用 — 移除后，4.7 上的 `speed: "fast"` 返回错误。Opus 4.8 是持久的 fast-capable 级别。Fast mode 以高达 2.5 倍的输出 token/秒运行相同模型，按高级定价收费。每个请求需要三件事：使用 **beta** messages 端点（`client.beta.messages.…`），传递 beta 标志 `fast-mode-2026-02-01`，并设置 `speed: "fast"` 作为顶层请求参数（不是头，不在 `extra_body` 中）。

```python
client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=4096,
    speed="fast", betas=["fast-mode-2026-02-01"],
    messages=[...],
)
```

| 语言 | Beta 标志 | Speed 参数 |
|---|---|---|
| Python | `betas=["fast-mode-2026-02-01"]` | `speed="fast"` |
| TypeScript / Ruby | `betas: ["fast-mode-2026-02-01"]` | `speed: "fast"` |
| Go | `[]anthropic.AnthropicBeta{anthropic.AnthropicBetaFastMode2026_02_01}` | `Speed: anthropic.BetaMessageNewParamsSpeedFast` |
| Java | `.addBeta(AnthropicBeta.FAST_MODE_2026_02_01)` | `.speed(MessageCreateParams.Speed.FAST)` |
| C# | `Betas = ["fast-mode-2026-02-01"]` | `Speed = Speed.Fast`（`Anthropic.Models.Beta.Messages`） |
| PHP | `betas: ['fast-mode-2026-02-01']` | `speed: 'fast'` |
| cURL | `anthropic-beta: fast-mode-2026-02-01` 头 | body 中 `"speed": "fast"` |

`response.usage.speed` 报告使用的速度。Fast mode 有独立于标准 Opus 的速率限制；遇到 429 时，在 `retry-after` 延迟后重试或移除 `speed` 回退到标准模式（注意：切换速度会使提示词缓存失效）。不支持 Batch API、Priority Tier、AWS 上的 Claude Platform 或第三方平台。

---

## 任务预算（快速参考）

**Beta，Fable 5 / Sonnet 5 / Opus 4.8 / 4.7。** 任务预算给 Claude 一个代理循环的 token 上限，使其自行调节并在完成时优雅收尾而非被截断 — 区别于 `max_tokens`（模型不感知的每响应强制上限）。最小 `total`：20,000。在 `client.beta.messages.stream(...)` 上的 `output_config` 中设置 `task_budget`，带 beta 标志 `task-budgets-2026-03-13` — 使用流式传输以避免大 `max_tokens` 触发 HTTP 超时（完整详情：`shared/model-migration.md` → Task Budgets）：

```python
with client.beta.messages.stream(
    model="claude-opus-4-8", max_tokens=128000,
    output_config={"effort": "high", "task_budget": {"type": "tokens", "total": 64000}},
    betas=["task-budgets-2026-03-13"],
    messages=[...], tools=[...],
) as stream:
    response = stream.get_final_message()
```

`task_budget` 字段：`type`（始终为 `"tokens"`）、`total` 和可选 `remaining`（默认为 `total`）。服务端注入一个 Claude 在生成期间可见的倒计时标记；预算计算 Claude 生成的内容和它本轮读取的工具结果 — **不**包括你每次请求重新发送的完整历史。

**观察消耗：** 如果你想显示进度，在循环迭代中累计 `response.usage.output_tokens`（加上你追加的 tool-result 块的 token 计数）。在正常循环中不要设置 `remaining` — 服务端自行跟踪倒计时，在同时重新发送完整历史时传递客户端计算的 `remaining` 会少报预算。**仅在**你在请求之间压缩或重写历史且服务端无法推导先前消耗时才传递 `remaining`。

---

## 提供商客户端（快速参考）

当在第三方平台上使用 Claude 时，使用该平台的专用客户端类 — 而非带 `base_url` 覆盖的第一方 `Anthropic()` 客户端。构造后，客户端暴露与第一方 SDK 相同的 `messages.create` / `.stream` 接口。

### Amazon Bedrock

使用 **Mantle** 客户端（Messages-API Bedrock 端点）。Bedrock 模型 ID 带 `anthropic.` 前缀（如 `"anthropic.claude-opus-4-8"`）。区域为必填项。

| 语言 | 客户端 |
|---|---|
| Python | `from anthropic import AnthropicBedrockMantle` → `AnthropicBedrockMantle(aws_region="…")` |
| TypeScript | `import { AnthropicBedrockMantle } from "@anthropic-ai/bedrock-sdk"` → `new AnthropicBedrockMantle({ awsRegion: "…" })` |
| Go | `bedrock.NewMantleClient(ctx, bedrock.MantleClientConfig{ AWSRegion: "…" })` |
| Java | `AnthropicOkHttpClient.builder().backend(BedrockMantleBackend.fromEnv()).build()`（来自 `com.anthropic.bedrock.backends`） |
| C# | `new AnthropicBedrockMantleClient(new() { AwsRegion = "…" })`（包 `Anthropic.Bedrock`） |
| PHP | `use Anthropic\Bedrock\MantleClient;` → `new MantleClient(awsRegion: '…')` |
| Ruby | `Anthropic::BedrockMantleClient.new(aws_region: "…")` |

`AnthropicBedrock` / `BedrockClient` / `BedrockBackend`（无 `Mantle`）是旧版 `bedrock-runtime` InvokeModel 路径 — 新代码优先使用 Mantle 客户端。

### Microsoft Foundry

| 语言 | 客户端 |
|---|---|
| Python | `from anthropic import AnthropicFoundry` → `AnthropicFoundry(api_key=…, resource="…")` |
| TypeScript | `import AnthropicFoundry from "@anthropic-ai/foundry-sdk"` → `new AnthropicFoundry({ … })` |
| Java | `AnthropicOkHttpClient.builder().backend(FoundryBackend.fromEnv()).build()`（来自 `com.anthropic.foundry.backends`） |
| C# | `new AnthropicFoundryClient(new AnthropicFoundryApiKeyCredentials(…))`（包 `Anthropic.Foundry`） |
| PHP | `Foundry\Client::withCredentials(…)` |

Go 和 Ruby SDK 目前不支持 Foundry。对于 Ruby，使用标准 `Anthropic::Client.new(base_url: "<foundry endpoint>")` 作为回退（Entra ID 认证非内置）。对于 AWS 上的 Claude Platform，参见 `shared/claude-platform-on-aws.md`。

### Google Cloud Vertex AI

两个必填构造参数：GCP `project_id` 和 `region`。Vertex 模型 ID **不带前缀** — 当前代模型（Opus 4.8/4.7/4.6、Sonnet 5、Sonnet 4.6）使用裸第一方 ID（如 `"claude-opus-4-8"`）；日期快照模型使用 `@` 版本分隔符（如 `claude-opus-4-5@20251101`，**而非** `claude-opus-4-5-20251101`）。认证使用 GCP ADC（`gcloud auth application-default login`）；无需 Anthropic API 密钥。`region` 可以是 `"global"`（推荐）、多区域（`"us"`/`"eu"`）或特定区域。构造后，使用相同的 `messages.create` / `.stream` 接口。

| 语言 | 客户端 |
|---|---|
| Python | `from anthropic import AnthropicVertex` → `AnthropicVertex(project_id="…", region="…")`（安装 `"anthropic[vertex]"`） |
| TypeScript | `import { AnthropicVertex } from "@anthropic-ai/vertex-sdk"` → `new AnthropicVertex({ projectId, region })` |
| Go | `import "github.com/anthropics/anthropic-sdk-go/vertex"` → `anthropic.NewClient(vertex.WithGoogleAuth(ctx, region, projectID))` |
| Java | `AnthropicOkHttpClient.builder().backend(VertexBackend.builder().region("…").project("…").build()).build()`（来自 `com.anthropic.vertex.backends`） |
| C# | `new AnthropicClient { Backend = new VertexBackend(projectId, region) }`（包 `Anthropic.Vertex`） |
| PHP | `use Anthropic\Vertex;` → `Vertex\Client::fromEnvironment(location: '…', projectId: '…')` — 注意是 `location`，不是 `region` |
| Ruby | `Anthropic::VertexClient.new(region: "…", project_id: "…")` |

---

## 上下文编辑（快速参考）

**Beta。** 上下文编辑在模型看到对话前**清除**旧工具结果或思考块；它**不是压缩**（压缩是摘要）。在 `client.beta.messages.*` 上使用 beta `context-management-2025-06-27`，传递 `context_management.edits` 和策略类型：

```python
client.beta.messages.create(
    model="claude-opus-4-8", max_tokens=4096,
    betas=["context-management-2025-06-27"],
    context_management={"edits": [{"type": "clear_tool_uses_20250919"}]},
    tools=[...], messages=[...],
)
```

策略类型：`clear_tool_uses_20250919`（清除旧工具结果；可选 `clear_tool_inputs: true` 同时清除 tool_use 参数）和 `clear_thinking_20251015`（清除思考块）。**不要**使用 `compact_20260112` 或 beta `compact-2026-01-12` — 那是单独的压缩功能。

---

## 对话中系统消息（快速参考）

**仅限 Claude Opus 4.8；无 beta 头。** 将 `{"role": "system", "content": "…"}` 追加到 `messages` 数组（不是顶层 `system` 字段），在不使缓存前缀失效的情况下添加对话中的操作指令。使用常规 `client.messages.create` — 没有 beta。对话中系统消息必须跟在 `user` 消息之后（或以服务端工具使用结尾的 `assistant` 消息之后），且必须是 `messages` 中的最后一条或后跟一个 `assistant` 轮次 — 它不能是 `messages[0]`。可用性：`shared/platform-availability.md`。参见 `shared/prompt-caching.md` § 对话中系统消息。

---

## Managed Agents（Beta）

**Managed Agents** 是第三个接口：服务端管理的有状态代理，带 Anthropic 托管的工具执行。你创建一个持久化、版本化的 Agent 配置（`POST /v1/agents`），然后启动引用它的 Session。每个会话提供一个容器作为代理的工作区 — bash、文件操作和代码执行在那里运行；代理循环本身在 Anthropic 的编排层上运行，通过工具对容器操作。会话流式传输事件；你发回消息和工具结果。

可用性：`shared/platform-availability.md`。对于 Bedrock / Vertex / Foundry 上的代理（Managed Agents 不支持的地方），使用 Claude API + 工具使用。

**强制流程：** Agent（一次）→ Session（每次运行）。`model`/`system`/`tools` 在 agent 上，从不在 session 上。参见 `shared/managed-agents-overview.md` 获取完整阅读指南、beta 头和陷阱。

**Beta 头：** `managed-agents-2026-04-01` — SDK 对所有 `client.beta.{agents,environments,sessions,vaults,memory_stores,deployments,deployment_runs}.*` 调用自动设置。Skills API 使用 `skills-2025-10-02`，Files API 使用 `files-api-2025-04-14`，但你不需要为 `/v1/skills` 和 `/v1/files` 以外的端点显式传递它们。

**子命令** — 直接用 `/claude-api <subcommand>` 调用：

| 子命令 | 操作 |
|---|---|
| `managed-agents-onboard` | 引导用户从头设置 Managed Agent。**立即阅读 `shared/managed-agents-onboarding.md`** 并遵循其访谈脚本：**描述 → 配置 agent（提议，而非审问）→ 环境 → 会话**（与 Console 快速入门相同的流程，认证推迟到会话步骤）— 默认值和内联建议完成工作，在输出任何代码前有静默可行性检查（任务 vs 工具/凭据/数据）。不要摘要 — 执行访谈。 |

**阅读指南：** 从 `shared/managed-agents-overview.md` 开始，然后是主题文件 `shared/managed-agents-*.md`（core、environments、tools、events、outcomes、multiagent、webhooks、memory、scheduled-deployments、client-patterns、onboarding、api-reference）。对于 Python、TypeScript、Go、Ruby、PHP 和 Java，阅读 `{lang}/managed-agents/README.md` 获取代码示例。对于 cURL，阅读 `curl/managed-agents.md`。**Agent 是持久的 — 创建一次，按 ID 引用。** 将 agent 和 environment 定义为版本控制的 YAML，用 `ant` CLI 应用 — 这是推荐流程（参见 `shared/anthropic-cli.md`）：CLI 拥有控制平面（创建和更新 agent），你的代码拥有数据平面（用存储的 agent ID 调用 `sessions.create`）。仅在必须以编程方式配置时才在代码中调用 `agents.create()`；无论哪种方式，存储返回的 agent ID 并传递给后续每次 `sessions.create`；切勿在请求路径中调用 `agents.create()`。如果所需绑定未在语言 README 中展示，通过 WebFetch 获取 `shared/live-sources.md` 中的相关条目而非猜测。C# 通过 `client.Beta.Agents` 和相关命名空间支持 beta Managed Agents — 参见 `csharp/claude-api/README.md` 获取详情，或 `curl/managed-agents.md` 获取原始 HTTP 参考。

**当用户想从头设置 Managed Agent 时**（如"如何开始"、"引导我创建一个"、"设置一个新 agent"）：阅读 `shared/managed-agents-onboarding.md` 并运行其访谈 — 与 `managed-agents-onboard` 子命令相同的流程。

**当用户问"如何为 X 编写客户端代码"时：** 参考 `shared/managed-agents-client-patterns.md` — 涵盖无损流重连、`processed_at` 排队/已处理门、中断、`tool_confirmation` 往返、正确的空闲/终止断开门、idle 后状态竞态、流优先排序、文件挂载陷阱等。对于凭据，优先使用 vault `environment_variable` 凭据 — 第一等机制；密钥在出口处替换，永不进入沙箱（`shared/managed-agents-tools.md` → Vaults）。通过自定义工具在主机端保留凭据是 vault 凭据不适合时的回退方案（如自托管沙箱）。

**当用户希望代理按计划运行时**（cron、"每天晚上"、"周报"）：阅读 `shared/managed-agents-scheduled-deployments.md` — 部署按 cron 节奏自主触发会话，带每次触发运行记录和生命周期控制（暂停/恢复/归档）。

---

## 服务端工具（快速参考）

服务端工具在 Anthropic 基础设施上运行 — 无客户端执行循环。在 `tools` 中声明；结果作为内容块到达同一响应中。除非特别说明，**无 beta 头**。**优先使用你的模型支持的最新工具类型变体。** 下面的 `_20260209` web search / web fetch 变体（动态过滤）需要 Opus 4.8/4.7/4.6、Sonnet 5 或 Sonnet 4.6；旧模型的基础变体列在表后。

| 工具 | `type` | `name` | 关键可选参数 | 结果块类型 |
|---|---|---|---|---|
| Web 搜索 | `web_search_20260209` | `web_search` | `max_uses`、`allowed_domains`/`blocked_domains`、`user_location` | `web_search_tool_result` → `.content` 是 `web_search_result` 列表 |
| Web 获取 | `web_fetch_20260209` | `web_fetch` | `max_uses`、`allowed_domains`/`blocked_domains`、`citations`、`max_content_tokens` | `web_fetch_tool_result` → `.content` 是带 `document` 块的 `web_fetch_result` |
| 代码执行 | `code_execution_20260521` | `code_execution` | 无 | `bash_code_execution_tool_result` → `.content.stdout` / `.stderr` / `.return_code` |
| 工具搜索（regex） | `tool_search_tool_regex_20251119` | `tool_search_tool_regex` | 将其他工具标记 `defer_loading: true` | `tool_search_tool_result` |
| 工具搜索（BM25） | `tool_search_tool_bm25_20251119` | `tool_search_tool_bm25` | 将其他工具标记 `defer_loading: true` | `tool_search_tool_result` |

`web_search_20260209` / `web_fetch_20260209` 内置动态过滤 — 代码执行在底层运行，因此**不要**在 `tools` 中单独声明 `code_execution`（第二个执行环境会混淆模型）。对于早于 Opus 4.6 / Sonnet 4.6 的模型，使用基础变体 `web_search_20250305` / `web_fetch_20250910`；Vertex AI 上仅基础 `web_search_20250305` 可用。`code_execution_20260120`（REPL 持久化 + 程序化工具调用）在 Opus 4.5+ / Sonnet 4.5+ 上运行。**仅 Go SDK**：`code_execution_20260521` 位于 `client.Beta.Messages.New` 下，带 `Betas: []anthropic.AnthropicBeta{"code-execution-2025-08-25"}`（其他语言使用普通 `client.messages.create`）；`code_execution_20260120` 在 Go 中与其他语言一样使用非 beta 的 `client.Messages.New`。Web fetch 仅获取对话中已存在的 URL。各工具的提供商可用性不同 — 参见 `shared/platform-availability.md`。参见 `shared/tool-use-concepts.md` 了解 `pause_turn` 处理。

## 文档和文件输入（快速参考）

**PDF（base64，无 beta）：** `{"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": <b64 字符串>}}` 放在用户内容中，位于文本块之前。Base64 字符串不能有换行。限制：32 MB 请求，600 页（200k 上下文模型为 100 页）。Java：`ContentBlockParam.ofDocument(DocumentBlockParam... Base64PdfSource.builder().data(...))`。

**Files API（beta `files-api-2025-04-14`）：** 通过 `client.beta.files.upload(...)` 上传 → 响应 `id` 即 `file_id`。对于 PDF/文本，引用为 `{"type": "document", "source": {"type": "file", "file_id": "..."}}`；对于图片，引用为 `{"type": "image", ...}` — 内容块类型必须匹配文件的 MIME 类型。Beta 头在上传和引用文件的 `messages.create` 上**都**需要。可用性：`shared/platform-availability.md`。

**引用（无 beta）：** 在每个 `document` 内容块上设置 `citations: {enabled: true}`（全有或全无）。响应拆分为多个 `text` 块；被引用的块携带 `citations` 数组。每个引用有 `cited_text`、`document_index`、`document_title` 和按 `type` 的位置：纯文本用 `char_location`（`start_char_index`/`end_char_index`），PDF 用 `page_location`（`start_page_number`/`end_page_number`，1 索引），自定义内容用 `content_block_location`。与 `output_config.format` 不兼容（返回 400）。

## 工具使用模式（快速参考）

**严格工具使用（无 beta）：** 在工具定义上设置 `strict: true` 作为顶层字段（与 `name`/`description`/`input_schema` 并列），**不是**在 `tool_choice` 上。Schema 必须有 `additionalProperties: false` + `required`。保证 `tool_use.input` 精确验证。Go：`Strict: anthropic.Bool(true)` + 通过 `InputSchema.ExtraFields` 设置 `additionalProperties`；Java：`.strict(true)` + `.putAdditionalProperty("additionalProperties", JsonValue.from(false))`。

**并行工具使用（默认开启）：** 一个 assistant 消息可能包含多个 `tool_use` 块。并发执行它们，然后在**单个** user 消息中返回**所有** `tool_result` 块 — 将它们拆分到多个消息中会静默训练 Claude 停止并行调用。对于失败的工具，返回带 `is_error: true` 的 `tool_result` — 不要丢弃它。

**Tool Runner（SDK beta 辅助）：** 通过 `client.beta.messages.*` 自动驱动工具调用循环。Python：`@beta_tool` 装饰器 + `client.beta.messages.tool_runner(...)` → `runner.until_done()`。TypeScript：来自 `@anthropic-ai/sdk/helpers/beta/zod` 的 `betaZodTool({...})` + `client.beta.messages.toolRunner(...)` → `await runner`。Go：`toolrunner.NewBetaToolFromJSONSchema(...)` + `client.Beta.Messages.NewToolRunner(...)` → `.RunToCompletion(ctx)`。Java 需要 `.addBeta("structured-outputs-2025-11-13")`。Ruby：`Anthropic::BaseTool` 子类 + `client.beta.messages.tool_runner(...)`。PHP：`BetaRunnableTool` + `->toolRunner(...)`。C#：原始 JSON-schema 工具 + 通过 `client.Beta.Messages.ToolRunner(...)` 的 `BetaToolRunner`。

**程序化工具调用（无 beta 头）：** Claude 从代码执行内部调用你的自定义工具。添加 `{"type": "code_execution_20260120", "name": "code_execution"}` **并**在你的自定义工具上设置 `"allowed_callers": ["code_execution_20260120"]`。Opus 4.5+ / Sonnet 4.5+（可用性：`shared/platform-availability.md`）。响应待处理的程序化调用时，user 消息必须**仅**包含 `tool_result` 块（无文本）。与 `strict: true`、`disable_parallel_tool_use`、强制 `tool_choice` 或 MCP 工具不兼容。

## 其他 API 接口（快速参考）

**Message Batches（无 beta；可用性：`shared/platform-availability.md`）：** `client.messages.batches.create(requests=[{custom_id, params}, ...])` → 轮询 `client.messages.batches.retrieve(id).processing_status` 直到 `"ended"` → 流式获取 `client.messages.batches.results(id)`。每个结果有 `.custom_id` + `.result.type`（`succeeded`/`errored`/`canceled`/`expired`）；成功时读取 `.result.message.content`。Python 将请求包装为 `Request(custom_id=..., params=MessageCreateParamsNonStreaming(...))`。结果以**任意顺序**到达 — 按 `custom_id` 索引，切勿按位置。

**Models API（无 beta；可用性：`shared/platform-availability.md`）：** `client.models.list()`（自动分页）和 `client.models.retrieve("claude-opus-4-8")`。每个模型对象有 `id`、`display_name`、`created_at` 以及 — 自 2026 年 3 月起 — `max_input_tokens`（上下文窗口）、`max_tokens`（输出上限）和 `capabilities`。没有 `context_window` 字段。

**停止详情（GA，Opus 4.7+）：** `response.stop_details` **仅在 `stop_reason == "refusal"` 时**被填充（字段：`type: "refusal"`、`category: "cyber"|"bio"|null`、`explanation`）。对其他所有 `stop_reason`（`end_turn`、`max_tokens`、`tool_use`、`pause_turn` 等）为 `null` — 读取前始终检查。

**客户端配置（无 beta）：** `timeout` 默认 10 分钟；**单位因 SDK 而异** — Python/Ruby：秒；TypeScript：**毫秒**；Go `option.WithRequestTimeout(time.Duration)`；Java `Duration`；C# `TimeSpan`。TS 对大 `max_tokens` 非流式请求将默认值放大到 60 分钟；Java 对流式请求如此（Java 非流式放大 30s–10 min）。`max_retries`/`maxRetries` 默认 2（重试 408/409/429/5xx + 连接错误）。`base_url`（或 `ANTHROPIC_BASE_URL` 环境变量）。每请求覆盖：Python `client.with_options(timeout=5.0).messages.create(...)`；TS `client.messages.create({...}, {timeout: 5_000})`；Ruby `request_options: {timeout: 5}`。超时会被重试 — 挂钟时间可达 `timeout × (max_retries+1)`。

## 工作负载身份联合（快速参考）

**GA，无 beta 头。** 构造常规零参数客户端（`Anthropic()` / `new Anthropic()` / `anthropic.NewClient()` / `AnthropicOkHttpClient.fromEnv()`）；当 `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID` 和 `ANTHROPIC_IDENTITY_TOKEN_FILE`（或 `ANTHROPIC_IDENTITY_TOKEN`）**全部**设置时，SDK 自动检测 WIF，在 `/v1/oauth/token` 交换 JWT，并自动刷新。`ANTHROPIC_WORKSPACE_ID` 不门控激活 — 仅当联合规则跨多个工作区时需要（否则 400 `workspace_id_required`），单工作区规则可选。`ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN`（即使为空）优先于 WIF，已设置的 `ANTHROPIC_PROFILE` 也优先于联合环境变量（缺失的命名配置是错误，不是穿透）— 全部取消设置这三者。

---

## 参考文档

下方 `<doc>` 标签中包含你检测到的语言的相关文档。每个标签有 `path` 属性显示其原始文件路径。用此找到正确的部分：

### 快速任务参考

> 所有 SDK 语言使用相同的按语言 `claude-api/` 目录布局（cURL：`curl/examples.md`）。并非每种语言都有每个文件 — 如果文件不存在，该功能的示例尚未为该语言文档化；回退到 cURL 形状或 WebFetch SDK 仓库。

**单次文本分类/摘要/提取/问答：**
→ 参考 `python/claude-api/README.md`

**聊天 UI 或实时响应显示：**
→ 参考 `python/claude-api/README.md` + `python/claude-api/streaming.md`

**长时间运行的对话（可能超过上下文窗口）：**
→ 参考 `python/claude-api/README.md` — 参见压缩部分

**迁移到较新模型或替换已退役模型：**
→ 参考 `shared/model-migration.md`

**提示词缓存 / 优化缓存 / "为什么我的缓存命中率低"：**
→ 参考 `shared/prompt-caching.md` + `python/claude-api/README.md`（提示词缓存部分）

**计算文件/提示词/diff 中的 token（"X 有多少 token"）：**
→ 参考 `shared/token-counting.md` — 使用 `messages.count_tokens`，切勿用 `tiktoken`

**函数调用 / 工具使用 / 代理：**
→ 参考 `python/claude-api/README.md` + `shared/tool-use-concepts.md` + `python/claude-api/tool-use.md`

**批处理（非延迟敏感）：**
→ 参考 `python/claude-api/README.md` + `python/claude-api/batches.md`

**跨多个请求的文件上传：**
→ 参考 `python/claude-api/README.md` + `python/claude-api/files-api.md`

**代理设计（工具面、上下文管理、缓存策略）：**
→ 参考 `shared/agent-design.md`

**Anthropic CLI（`ant`）— 终端访问、版本控制的 agent/environment YAML、脚本编写：**
→ 参考 `shared/anthropic-cli.md`

**Managed Agents（服务端管理的有状态代理）：**
→ 参考 `shared/managed-agents-overview.md` 和其余 `shared/managed-agents-*.md` 文件。对于 Python、TypeScript、Go、Ruby、PHP 和 Java，阅读语言文件夹中的 `managed-agents/README.md` 获取代码示例。对于 cURL，阅读 `curl/managed-agents.md`。C# 有 beta Managed Agents 支持 — 使用 `curl/managed-agents.md` 作为线路级参考（C# SDK 通过 `client.Beta.Agents` 镜像它；参见 `csharp/claude-api/README.md`）。

**错误处理：**
→ 参考 `shared/error-codes.md`

**通过 WebFetch 获取最新文档：**
→ 参考 `shared/live-sources.md` 获取 URL

## 何时使用 WebFetch

在以下情况下使用 WebFetch 获取最新文档：

- 用户要求"最新"或"当前"信息
- 缓存数据似乎不正确
- 用户询问此处未涵盖的功能

实时文档 URL 在 `shared/live-sources.md` 中。

## 常见陷阱

- 将文件或内容传递给 API 时不要截断输入。如果内容太长无法放入上下文窗口，通知用户并讨论选项（分块、摘要等），而非静默截断。
- **预填充已移除（Fable 5 和 4.6/4.7/4.8 系列）：** 助手消息预填充（最后助手轮次预填充）在 Fable 5、Opus 4.6、Opus 4.7、Opus 4.8 和 Sonnet 4.6 上返回 400 错误。改用结构化输出（`output_config.format`）或系统提示指令来控制响应格式。（一个例外：回退信用预填充声明 — 当以 `fallback_has_prefill_claim: true` 兑换信用时，服务端接受回显的助手消息；参见迁移指南的 refusal 部分。）
- **编辑前确认迁移范围：** 当用户要求将代码迁移到较新 Claude 模型而未指定文件、目录或文件列表时，**先询问应用哪个范围** — 整个工作目录、特定子目录还是特定文件集。在用户确认前不要开始编辑。"迁移我的代码库"、"将我的项目移到 X"、"升级到 Sonnet 4.6"或裸的"迁移到 Opus 4.8"等祈使措辞**仍然有歧义** — 它们告诉你做什么但不告诉你在哪里，所以要问。仅当提示词指定确切文件、特定目录或显式文件列表时才不问而直接执行（"迁移 `app.py`"、"迁移 `services/` 下所有内容"、"更新 `a.py` 和 `b.py`"）。参见 `shared/model-migration.md` 步骤 0。
- **`max_tokens` 默认值：** 不要低估 `max_tokens` — 达到上限会在思考中途截断输出并需要重试。非流式请求默认 `~16000`（保持响应在 SDK HTTP 超时内）。流式请求默认 `~64000`（超时不是问题，给模型空间）。仅在有硬性理由时降低：分类（`~256`）、成本上限、刻意短输出，或用于缓存预热的 **`max_tokens: 0`**（参见 `shared/prompt-caching.md` → Pre-warming）。
- **128K 输出 token：** Fable 5、Opus 4.6、Opus 4.7、Opus 4.8、Sonnet 5 和 Sonnet 4.6 支持最高 128K `max_tokens`，但 SDK 要求流式传输以避免 HTTP 超时。使用 `.stream()` 配合 `.get_final_message()` / `.finalMessage()`。
- **工具调用 JSON 解析（Fable 5 和 4.6/4.7/4.8 系列）：** Fable 5、Opus 4.6、Opus 4.7、Opus 4.8 和 Sonnet 4.6 可能在工具调用 `input` 字段中产生不同的 JSON 字符串转义（如 Unicode 或正斜杠转义）。始终用 `json.loads()` / `JSON.parse()` 解析工具输入 — 切勿对序列化输入做原始字符串匹配。
- **结构化输出（所有模型）：** 使用 `output_config: {format: {...}}` 而非 `messages.create()` 上已弃用的 `output_format` 参数。这是通用 API 变更，不是 4.6 专有的。
- **不要重新实现 SDK 功能：** SDK 提供高级辅助 — 使用它们而非从头构建。具体来说：使用 `stream.finalMessage()` 而非将 `.on()` 事件包装在 `new Promise()` 中；使用类型化异常类（`Anthropic.RateLimitError` 等）而非字符串匹配错误消息；使用 SDK 类型（`Anthropic.MessageParam`、`Anthropic.Tool`、`Anthropic.Message` 等）而非重新定义等效接口。
- **错误处理 — 捕获链，而非一个宽泛类。** 单个 `except APIStatusError` / `catch (AnthropicServiceException)` / `rescue APIError` 会丢失可重试（429、≥500、网络）和不可重试（400/404）失败之间的区别。写一个最具体优先的链 — 如 `NotFoundError` → `RateLimitError` → `APIStatusError` → `APIConnectionError`（或 Go 等价物：`errors.As` 转入 `*anthropic.Error` 然后 `switch apierr.StatusCode { case 404: …; case 429: …; default: … }`）。各语言类名和命名空间在 `shared/error-codes.md` 中。
- **不要研究 SDK 类型 — 先写代码。** 如果文档中未显示类型名称，从语言特定文档中的命名空间/包表编写代码文件，让编译器的错误指向正确名称。不要在编写前花轮次在 WebFetch、SDK 仓库克隆或编译运行单独的反射程序来发现类型名称 — 先生成源文件，然后修复编译器报告的问题。对已安装 SDK 快速运行 `strings` / `jar tf` / `javap` 来定位名称是可以接受的（几秒返回），但不要升级到更多。有错误类型名的文件可恢复；花在发现上而没有写出文件的会话不可恢复。
- **Bash 和文本编辑器工具是 Anthropic 定义的、无 schema 的。** 声明 `{"type": "bash_20250124", "name": "bash"}` / `{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}` — 无 `input_schema`。用你自己的 schema 命名为 `"bash"` 的自定义工具是不同的工具。处理路径和安全检查在 `shared/tool-use-concepts.md` § 客户端工具中。
- **Advisor 工具模型配对。** Advisor 工具的 `model` 必须至少与请求的顶层 `model` 一样强 — 如执行器 `claude-sonnet-5` → advisor `claude-opus-4-8` 或 `claude-opus-4-7`。无效配对返回 400。配对表在 `shared/tool-use-concepts.md` § Advisor 中。可用性：`shared/platform-availability.md`。
- **Agent Skills ≠ Managed Agents。** 要让 Claude 通过 Agent Skills 生成 `.pptx`/`.xlsx` 等，调用 `client.beta.messages.create` 带 `container={"skills": [...]}`、`code_execution_20260521` 工具以及 `code-execution-2025-08-25` + `skills-2025-10-02` betas。此处不要使用 `client.beta.agents` / `sessions` / `environments` — 那些是 Managed Agents 接口，不是 Agent Skills。
- **MCP 连接器需要两半。** 仅 `mcp_servers=[{type:"url", url, name}]` 会被作为验证错误拒绝 — 还需添加 `tools=[{type:"mcp_toolset", mcp_server_name:<相同 name>}]` 带 beta `mcp-client-2025-11-20`。可用性：`shared/platform-availability.md`。
- **`inference_geo` 是直接的顶层请求参数** — `client.messages.create(..., inference_geo="us")` / `.inferenceGeo("us")`。不要放在 `extra_body` / `putAdditionalBodyProperty` 中。在 Opus 4.6 / Sonnet 4.6 及更高版本上支持；可用性：`shared/platform-availability.md`。`response.usage.inference_geo` 报告推理运行的位置。
- **细粒度工具流式传输不是 beta 功能。** 在工具定义上设置 `eager_input_streaming: true` 并调用常规 `client.messages.stream(...)`。没有 beta 头也没有 `client.beta.*` 路径。
- **缓存诊断是 beta。** 使用 `client.beta.messages.*` 带 beta `cache-diagnosis-2026-04-07`。第一轮传递 `diagnostics: {previous_message_id: null}`，后续轮次传递 `diagnostics: {previous_message_id: <先前响应 id>}`；结果在 `response.diagnostics` 上。可用性：`shared/platform-availability.md`。
- **Memory 工具类型是 `memory_20250818`。** 声明 `{"type": "memory_20250818", "name": "memory"}`。Go 在 `client.Beta.Messages.New` 上使用 beta 命名空间类型 `{OfMemoryTool20250818: &anthropic.BetaMemoryTool20250818Param{}}`；Python/TypeScript/Ruby/PHP/C# 使用非 beta 的 `client.messages.create`；Java 同时有非 beta 的 `MemoryTool20250818` 和 beta tool-runner 路径。Python/TypeScript 提供 `BetaAbstractMemoryTool` / `betaMemoryTool` 辅助来实现后端。
- **使用功能实际支持的模型。** 某些功能限于特定模型级别 — fast mode 仅限 Opus 4.8 / 4.7，task budgets 仅限 Fable 5 / Sonnet 5 / Opus 4.8 / 4.7，advisor 工具需要有效的执行器↔advisor 配对。如果用户的提示词指定了功能不支持的模型，改用支持的模型并在输出中注明替换。
- **不要为 SDK 数据结构定义自定义类型：** SDK 为所有 API 对象导出类型。消息用 `Anthropic.MessageParam`，工具定义用 `Anthropic.Tool`，工具结果用 `Anthropic.ToolUseBlock` / `Anthropic.ToolResultBlockParam`，响应用 `Anthropic.Message`。定义你自己的 `interface ChatMessage { role: string; content: unknown }` 会重复 SDK 已提供的功能并丢失类型安全。
- **报告和文档输出：** 对于生成报告、文档或可视化的任务，代码执行沙箱预装了 `python-docx`、`python-pptx`、`matplotlib`、`pillow` 和 `pypdf`。Claude 可以生成格式化文件（DOCX、PDF、图表）并通过 Files API 返回 — 对于"报告"或"文档"类型请求考虑这种方式而非纯 stdout 文本。
- **服务端工具错误不会抛出异常。** Web 搜索和 web fetch 错误返回 HTTP 200，带 `web_search_tool_result` / `web_fetch_tool_result` 块，其 `content` 是单个错误对象（如 `{error_code: "max_uses_exceeded"}`）— 不是抛出的异常。对于 web 搜索，成功的 `content` 是*列表*；错误的 `content` 是*对象* — 在索引前据此分支。
- **代码执行输出块类型：** `code_execution_20260521` 返回 `bash_code_execution_tool_result`（带 `.content.stdout`），**不是**旧版裸 `code_execution_tool_result`。遍历 `response.content` 并匹配正确类型。
- **工具搜索：切勿全部 defer。** 搜索工具本身不能有 `defer_loading: true`，且 `tools` 中至少一个工具必须是非 defer 的，否则 API 返回 400 `All tools have defer_loading set`。