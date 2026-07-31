# HANDOFF — 跨会话交接文档

> 最后更新：2026-07-31 | 会话主题：合并 upstream + 补全 desktop-fable-5 翻译

## 当前任务

合并 upstream/main 到 origin/main，处理 3 个语义影响点，补全新文件中文镜像。

**整体进度**：✅ 全部完成并推送到 origin/main。

## 已完成工作

### 7 个翻译补全（早期会话，commits `399b758`-`a4df403`）
详见 archive/2026-07-31.md。文件：claude-design, claude-fable-5, claude-opus-5, claude-opus-4.6-raw, claude-sonnet-4.6-raw, claude-cowork, codex-full。

### Upstream 合并（commit `e043d1b`）
- `git merge upstream/main` —— 干净合并，无冲突
- Upstream 4 commits 合入：Latitude 赞助商 banner、删除 `Claude Code/README.md`、新增 `claude-code-desktop-fable-5.md`（303KB）

### 3 个语义影响点处理（commit `a20fd94`）
- **A. 孤儿 README-zh.md**：删除 `Anthropic/Claude Code/README-zh.md`（跟随 upstream 删除原文）
- **B. README 赞助商同步**：`README-zh.md` 顶部加 Latitude sponsor banner（与 upstream README.md 一致）
- **C. 新文件翻译**：`claude-code-desktop-fable-5-zh.md` 翻译完成
  - 源文件 311.4KB / 7192 行 / 384 fences
  - 译文 297.4KB / 7196 行 / 384 fences（比率 95.5%）
  - 拆分策略：Part1 (180KB) + Part2a (64KB) + Part2b (67KB) 3 段并行
  - Part2 首次超时截断（仅完成 19%），拆成 Part2a+Part2b 重试成功

### 已推送 commits
- `e043d1b` — merge upstream/main
- `a20fd94` — 3 个语义点处理 + desktop-fable-5 翻译

## 当前约束（活跃限制，非历史经验）

- ⛔ **原文照贴原则**：系统提示词文件必须原样粘贴，禁止摘要/改写/润色。`{model_name}` 等占位符必须保留
- ⛔ **元文档无声明头规则**：README/CONTRIBUTING/AGENTS/SKILL-zh.md 等元文档不应有声明头；产品系统提示词 `-zh.md` 必须有声明头
- ⛔ **`.handoff/` 绝不能加入 .gitignore**：按 AGENTS.md 规则须版本控制
- ⛔ **大文件翻译拆分阈值**：>150KB 需拆分，>200KB 必须拆分成 <100KB 的段并行翻译（deep 类别，单任务 <100KB 成功率高）

## 当前卡点

- 无。

## 下一步计划

1. **(可选) 创建 PR 到 upstream**（优先级：低）：向 `asgeirtj/system_prompts_leaks` 提 PR 贡献中文翻译

## 踩坑记录（绝对不要再踩）

- **deep 类别 >130KB 文件超时风险高**：Part2（131.6KB）超时仅完成 19%；拆成 65KB 左右的子部分成功率 100%。阈值：单任务 <100KB
- **commit-msg hook 格式**：要求 `:emoji: type(scope): subject`；merge commit 默认 message 会被拒绝，需手动指定规范格式
- **PowerShell CJK 格式串陷阱**：`-f` 格式串中含 CJK 字符会触发 parser error，用纯 ASCII 输出
- **PowerShell `temp` 变量陷阱**：`temp` 不是有效 cmdlet 名，必须用 `Join-Path $temp` 构建路径
- **`writing` 类别对长翻译不可靠**：>100KB 文件易崩溃 EOF，改用 `deep` 类别
- **fence 数量验证**：翻译后必须检查 `^\s*`` ` 行数与源文件匹配，可发现重复/遗漏的代码块标记
- **拆分点选择**：必须在 fence 闭合点拆分（fence count 为偶数处），否则合并后 fence 错位

## 关键上下文

### 仓库信息
- **仓库**：`D:\workspace\github\system_prompts_leaks`
- **性质**：纯 Markdown 内容仓库，收集 AI 产品系统提示词泄露。无构建/测试/lint/源代码
- **Git**：`origin`→tonydeng/system_prompts_leaks（fork），`upstream`→asgeirtj/system_prompts_leaks，分支 `main`
- **Main HEAD**：`a20fd94`，clean，已与 origin 同步，与 upstream 同步
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
