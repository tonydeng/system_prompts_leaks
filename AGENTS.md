# AGENTS.md — system_prompts_leaks

> 仓库协作准则。每条都回答："若无此提示，智能体是否会踩坑？" 否则不写入。
> 本文件为元文档（中文），不参与 `-zh.md` 镜像约定。

## 仓库性质

**纯 Markdown 内容仓库**，收集各 AI 产品（Claude / ChatGPT / Gemini / Grok 等）的系统提示词泄露。**无构建、无测试、无 lint、无源代码**。不要尝试运行 build / test / typecheck / lint——会全部失败或无意义。

## 核心完整性规则（最高优先级）

1. **原文照贴**：系统提示词文件必须原样粘贴，禁止摘要、改写、润色。`{model_name}` 等占位符必须保留。参见 `.github/CONTRIBUTING.md`："Paste the raw system prompt as-is. Don't summarize or paraphrase — the full, unedited text is the point."
2. **`.gitattributes` 规则**：`*.md -whitespace` —— .md 文件的尾部空白/不一致空白**不被标记为错误**，这是有意为之以保留原文保真度。看到 whitespace 警告被忽略不要惊讶，也不要"修复"已有 .md 的空白。
3. **新增提示词必须同步更新 `README.md`**：
   - 对应 vendor 章节的主表格添加链接行
   - 顶部 "Recently Updated" 表格添加条目（日期 + 链接，最新置顶）
   - README.md 是全仓库索引，漏更新 = 新内容无人发现

## 目录结构

按厂商分目录，每个 .md 文件 = 一个产品/模型的系统提示词：

| 目录 | 覆盖 | 子目录 |
|------|------|--------|
| `Anthropic/` | Claude 系列 | `Claude Code/`（含 `bundled-skills/`）、`Official/`、`old/`、`raw/` |
| `OpenAI/` | ChatGPT / GPT / o-series | `API/`、`Codex/`、`Old/` |
| `Google/` | Gemini | — |
| `xAI/` | Grok | — |
| `Perplexity/` `Microsoft/` | Perplexity / Copilot | — |
| `Cursor/` `Meta/` `Mistral/` `Notion/` `Qwen/` | 单产品目录 | — |
| `Misc/` | 其他（Zed / Warp / Brave 等） | — |

新厂商提示词归入最贴近的目录；都不匹配则放 `Misc/`。文件名用描述性 kebab-case，匹配模型或产品名。

## Git 配置（fork 工作流）

- `origin` → `tonydeng/system_prompts_leaks`（个人 fork）
- `upstream` → `asgeirtj/system_prompts_leaks`（上游原仓库）
- 主分支：`main`，直接在 `main` 工作（纯内容仓库，无需 worktree 隔离）
- PR 目标：上游 `asgeirtj/system_prompts_leaks`
- **`.gitignore`**：已配置，忽略 `.omo/`（OpenCode 会话元数据）、编辑器/IDE 临时文件、OS 垃圾文件。**注意：`.handoff/` 不被忽略**（见下文跨会话交接章节，须纳入版本控制）。

## 中文镜像约定（-zh.md）

每个**英文** `.md` 文件都维护一份中文镜像，放在**同目录**下，文件名加 `-zh` 后缀。

**命名**：
- `Cursor/cursor.md` → `Cursor/cursor-zh.md`
- `OpenAI/gpt-5.5-thinking.md` → `OpenAI/gpt-5.5-thinking-zh.md`
- `README.md` → `README-zh.md`
- `.github/CONTRIBUTING.md` → `.github/CONTRIBUTING-zh.md`

**内容要求**：
- **元文档**（README / CONTRIBUTING 等）：完整中文化翻译，可独立阅读。
- **系统提示词原文**：`-zh.md` 为**中文译注版**，保留英文原文为权威来源。译注版辅助理解，不可替代原文。任何冲突以英文原文为准；占位符（如 `{model_name}`）保持原样不译。
- 新增/修改英文 .md 时，必须同步创建/更新对应 `-zh.md`。
- `README.md` 顶部 "Recently Updated" 表格只在英文版维护，`README-zh.md` 可滞后。
- 已经是中文的 .md（如本 AGENTS.md）无需镜像。

## 跨会话交接（.handoff/）

借自 Scorpius 项目的跨会话记忆机制。AI 智能体记忆以会话为界，项目推进以周为界，`.handoff/` 弥合这一落差。

### 目录结构

```
<repo-root>/.handoff/
├── current.md          ← 当前交接文档（每次会话覆盖更新）
└── archive/
    └── YYYY-MM-DD.md   ← 按日期归档的历史版本
```

### 机制分工（避免与 OpenCode 内置机制重叠）

| 机制 | 作用域 | 何时用 | 不何时用 |
|------|--------|--------|----------|
| `.handoff/current.md` | 跨会话短期 | 会话结束写入当前项目状态 | 不用于会话内上下文压缩 |
| `compress` | 会话内 | 上下文膨胀时压缩历史对话 | 不用于跨会话信息传递 |
| `Supermemory` | 跨会话长期 | 项目知识、模式沉淀 | 不用于临时任务状态 |

### 规则

1. **会话结束前**（长会话或任务阶段性完成时）：将 `current.md` 复制到 `archive/{YYYY-MM-DD}.md`（同日重名加 `-N` 后缀），再覆盖写入 `current.md` 反映最新状态。
2. **新会话启动时**：第一步读取 `.handoff/current.md`（如存在）。
3. **读取验证（强制）**：读取后在首次回复输出以下确认块，未输出 = 未读取：
   ```
   📋 HANDOFF 已恢复：
   - 当前任务：[一句话]
   - 下一步：[最高优先级行动]
   - 关键约束：[最需要注意的限制]
   ```
4. **纳入版本控制**：`.handoff/` 不加入 `.gitignore`，随项目提交。`current.md` 每次覆盖更新后提交，`archive/` 保留所有历史版本。
5. **current.md 必含**：当前任务 / 已完成工作 / 当前约束（活跃限制，非历史经验）/ 当前卡点 / 下一步计划 / 踩坑记录（绝对不要再踩）/ 关键上下文（写给无先验知识的新会话）。

### current.md 模板

```markdown
# HANDOFF — 跨会话交接文档
> 最后更新：{日期} | 会话主题：{主题}

## 当前任务
{目标}

## 已完成工作
- {项}：产出在 {路径}

## 当前约束（活跃限制，非历史经验）
- ⛔ {约束}：{描述}

## 当前卡点
- {卡点}：{描述}

## 下一步计划
1. {步骤}（优先级：高）

## 踩坑记录（绝对不要再踩）
- {坑}：{解决方案}

## 关键上下文
{写给新会话的背景，不假设先验知识}
```

## 其他

- **图片资源**：`.github/banner-dark.png` / `banner-light.png` 用于 README 明暗主题切换。新增图片资源放 `.github/`。
- **LICENSE**：仓库有 LICENSE 文件；泄露的系统提示词版权归各厂商，本仓库为归档目的收录。
- **无 worktree 约定**：纯内容仓库，直接在 `main` 分支工作即可。
