# HANDOFF — 跨会话交接文档

> 最后更新：2026-07-31 | 会话主题：补全 7 个缺失的 -zh.md 中文镜像翻译

## 当前任务

为 `system_prompts_leaks` 仓库补全 7 个缺失的 `-zh.md` 中文镜像翻译。

**整体进度**：✅ 全部 7 个翻译已完成、验证通过并提交到 main。5 个 commits 待 push 到 origin。

## 已完成工作

### 7 个翻译全部完成并提交

| # | 文件 | 大小 | 比率 | Fences | 提交 |
|---|------|------|------|--------|------|
| 1 | `Anthropic/claude-design-zh.md` | 191.5KB | 95% | 160/160 | `399b758` |
| 2 | `Anthropic/claude-fable-5-zh.md` | 209.6KB | 90.9% | 142/142 | `a4df403`（重译） |
| 3 | `Anthropic/claude-opus-5-zh.md` | 206.4KB | 92.7% | 134/134 | `9fd3ba3`（修复重复 fence） |
| 4 | `Anthropic/raw/claude-opus-4.6-raw-zh.md` | 161.9KB | 70.8% | 28/28 | `399b758` |
| 5 | `Anthropic/raw/claude-sonnet-4.6-raw-zh.md` | 164.2KB | 70.7% | 27/27 | `399b758` |
| 6 | `Anthropic/claude-cowork-zh.md` | 249.7KB | 90.2% | 4/4 | `9fd3ba3` |
| 7 | `OpenAI/Codex/codex-full-zh.md` | 356.7KB | 98.5% | 492/492 | `9fd3ba3` |

### 本会话提交
- `399b758` — 5 个翻译（design, fable-5, opus-5, opus-4.6-raw, sonnet-4.6-raw）
- `a48bb8d` — merge 5 translations
- `9fd3ba3` — cowork + codex-full 新增，opus-5 fence 修复
- `a4df403` — fable-5 重译（并行会话产出，通过验证后提交）

### 翻译策略
- **大文件拆分**：>200KB 文件按代码块 fence 边界拆分为多段并行翻译
  - `claude-cowork.md`（276.9KB）→ 2 段（Part1: 1173行, Part2: 2172行）
  - `codex-full.md`（362KB, 11104行）→ 4 段（Part1: 2565行, Part2: 2605行, Part3: 2522行, Part4: 3412行）
- **合并规则**：Part1 加声明头，Part2-4 不加；合并后全量验证
- **并发限制**：max 2-3 个并行 deep 后台任务
- **类别选择**：`deep` 类别用于长翻译（`writing` 类别对 >100KB 文件易崩溃 EOF）

### 验证标准（全部通过）
- 文件存在性 ✓
- 大小比率 60-130% ✓
- 声明头存在（产品系统提示词）✓
- 代码 fence 数量与源文件匹配 ✓
- 无末行截断 ✓
- 结构保真度（工具头数量、关键标识符）✓

## 当前约束（活跃限制，非历史经验）

- ⛔ **原文照贴原则**：系统提示词文件必须原样粘贴，禁止摘要/改写/润色。`{model_name}` 等占位符必须保留
- ⛔ **元文档无声明头规则**：README/CONTRIBUTING/AGENTS/SKILL-zh.md 等元文档不应有声明头；产品系统提示词 `-zh.md` 必须有声明头
- ⛔ **`.handoff/` 绝不能加入 .gitignore**：按 AGENTS.md 规则须版本控制
- ⛔ **README 不链接 -zh.md**：README 只链接英文原文，`-zh.md` 镜像在目录中共存即可，无需更新 README

## 当前卡点

- 无。所有翻译已完成并提交。

## 下一步计划

1. **Push 到 origin**（优先级：高）：5 个 commits 待推送到 `tonydeng/system_prompts_leaks`（需用户明确授权）
2. **(可选) 创建 PR 到 upstream**（优先级：低）：向 `asgeirtj/system_prompts_leaks` 提 PR
3. **清理临时文件**（优先级：低）：`C:\Users\ADMINI~1\AppData\Local\Temp\opencode\` 下的 codex-part*/cowork-part* 文件可删除

## 踩坑记录（绝对不要再踩）

- **PowerShell `temp` 变量陷阱**：`temp` 不是有效 cmdlet 名，必须用 `Join-Path $temp` 构建路径，不能写 `(temp 'file.md')`
- **PowerShell CJK 格式串陷阱**：`-f` 格式串中含 CJK 字符会触发 parser error，用纯 ASCII 输出
- **`writing` 类别对长翻译不可靠**：>100KB 文件易崩溃 EOF，改用 `deep` 类别
- **Agent 路径丢失问题**：后台 agent 上下文压缩后会把文件写到错误位置——必须在 prompt 中显式指定绝对路径
- **fence 数量验证**：翻译后必须检查 `^\s*`` ` 行数与源文件匹配，可发现重复/遗漏的代码块标记
- **`container`/`reasoning` 等词的误报**：这些词在 prose 中翻译为中文（容器/推理），在代码块中保留英文——分别统计中英文出现次数可区分
- **并行会话修改**：另一会话可能修改已提交文件（如 fable-5 被重译），提交前检查 `git status` 和 `git diff`

## 关键上下文

### 仓库信息
- **仓库**：`D:\workspace\github\system_prompts_leaks`
- **性质**：纯 Markdown 内容仓库，收集 AI 产品系统提示词泄露。无构建/测试/lint/源代码
- **Git**：`origin`→tonydeng/system_prompts_leaks（fork），`upstream`→asgeirtj/system_prompts_leaks，分支 `main`
- **Main HEAD**：`a4df403`，clean，5 commits ahead of origin/main
- **.gitattributes**：`*.md -whitespace`（.md 空白警告是有意为之，保留原文保真度）

### 声明头格式（标准）
```
> **说明**：本文件为英文原文（`{原文件名}`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。
```
- **头部位置规则**：产品系统提示词 `-zh.md` 必须有声明头（首行）；元文档（README/CONTRIBUTING/AGENTS/SKILL-zh.md）不应有声明头

### 中文镜像约定（AGENTS.md 文档化）
- **命名**：`{filename}.md` → `{filename}-zh.md`（同目录）
- **系统提示词 -zh.md**：有声明头，译注版辅助理解，英文原文为权威来源
- **元文档（README/CONTRIBUTING 等）**：无声明头，完整翻译，可独立阅读
- **已经是中文的 .md**（如 AGENTS.md）：无需镜像
- **纯中文翻译**，占位符/代码块不翻译，Markdown 格式保留

### HANDOFF 机制（借自 Scorpius）
- **位置**：`<repo-root>/.handoff/current.md`
- **归档**：会话结束前复制到 `.handoff/archive/{YYYY-MM-DD}.md`（同日重名加 `-N`）
- **读取验证**：新会话读取后必须输出 📋 HANDOFF 已恢复确认块
- **版本控制**：`.handoff/` 纳入 Git，不加入 .gitignore
- **机制分工**：.handoff/current.md（跨会话短期）vs compress（会话内）vs Supermemory（跨会话长期）
