# HANDOFF — 跨会话交接文档

> 最后更新：2026-07-30 | 会话主题：5-agent 审查 + P0/P1/P2 缺陷修复

## 当前任务

为 `system_prompts_leaks` 仓库的 427 个 `-zh.md` 中文镜像文件执行 5-agent 审查，修复审查报告中发现的所有 P0/P1/P2 问题。

**整体进度**：✅ 所有修复已完成并验证，待合并提交到 main。

## 已完成工作

### 审查阶段
- 5-lane 并行审查完成，编译报告：3 P0 + 6 P1 + 8 P2 问题

### P0 修复（已合并推送，commit `ddc16e6`）
- **P0 #1**：`run-zh.md` 删除混入的污染内容
- **P0 #2**：`gpt-5.5-thinking-zh.md` 删除重复章节
- **P0 #3**：`claude-3.7-sonnet-w-tools-zh.md` 恢复丢失的占位符

### P1 修复（在 fix-all worktree，待提交）
- **P1 #5**：3 个 SKILL-zh.md 文件移除不应有的声明头（元文档无声明头规则）
- **P1 #4+#6**：批量为 65 个 `-zh.md` 文件添加声明头（32 产品系统提示词 + 33 bundled-skills）

### P2 修复（在 fix-all worktree，待提交）
- **P2 #7**：`anthropic_reminders-zh.md` "种族或种族"→"种族或民族"
- **P2 #8**：`grok-4.5-zh.md` L18 "withholding"→"隐瞒"
- **P2 #9**：跳过——`security-review-zh.md` 编号 16 重复是英文原文 bug，保持与原文一致
- **P2 #11**：跳过——`CONTRIBUTING-zh.md` 元文档添加翻译声明合适
- **P2 #12**：跳过——`batch-zh.md` 经调查是单文件 skill（系统提示词），有声明头正确
- **P2 #13**：全量扫描发现 184 个非标准头变体（原审查仅标记 2 个），全部标准化
- **P2 #10**：`README-zh.md` 全量重写以匹配 `README.md` 结构

### 最终验证（PowerShell 脚本 UTF-8 编码执行）
- 总计 427 个 -zh.md 文件
- **382 个标准头** ✓
- **0 个非标准变体** ✓（P2 #13 标准化已确认）
- **45 个无声明头** = 全部为元文档（README/CONTRIBUTING/SKILL-zh.md/slash-command 文档/设计技能文档）✓ 符合规则

## 当前约束（活跃限制，非历史经验）

- ⛔ **原文照贴原则**：系统提示词文件必须原样粘贴，禁止摘要/改写/润色。`{model_name}` 等占位符必须保留
- ⛔ **元文档无声明头规则**：README/CONTRIBUTING/AGENTS/SKILL-zh.md 等元文档不应有声明头；产品系统提示词 `-zh.md` 必须有声明头
- ⛔ **batch-zh.md 判定**：是单文件 skill（系统提示词），不是元文档，有声明头是正确的——P2 #12 为误报
- ⛔ **Worktree 工作约定**：所有开发在 `spl-fix-all` worktree 进行，main 保持干净，通过合并更新
- ⛔ **`.handoff/` 绝不能加入 .gitignore**：按 AGENTS.md 规则须版本控制

## 当前卡点

- 无。所有修复已完成并验证。

## 下一步计划

1. **提交所有修复到 fix-all 分支**（优先级：高）：254 个文件已修改未提交
2. **合并 fix-all 到 main**（优先级：高）：切回主仓库 merge
3. **Push 到 origin**（优先级：高）：推送 main 到 `tonydeng/system_prompts_leaks`
4. **清理 worktree**（优先级：中）：`git worktree remove ../spl-fix-all`
5. **(可选) 创建 PR 到 upstream**（优先级：低）：向 `asgeirtj/system_prompts_leaks` 提 PR

## 踩坑记录（绝对不要再踩）

- **PowerShell `-match` 正则中文陷阱**：`-match` 操作符遇到含 `[^` 的中文正则会报 "Unterminated [] set"；需用 `-like` 通配符匹配或 `Select-String` 替代，或对正则中的 `[` 进行转义 `\[`
- **PowerShell 数组陷阱**：`@('a','b')` 含空格元素会塌缩为单元素，用 `System.Collections.ArrayList` + `[void]$list.Add()` 代替
- **Windows PowerShell 中文输出乱码**：控制台编码问题，不影响文件内容；用 Read 工具验证文件实际内容
- **验证脚本须写入 UTF-8 文件执行**：含中文的正则在 PowerShell 命令行直接执行会因控制台编码失败，将脚本写入 .ps1 文件（UTF-8）后执行可绕过
- **`.gitattributes` whitespace 警告是预期的**：`*.md -whitespace`，LF→CRLF 警告不要修复
- **commit-msg hook 格式**：要求 `:emoji: type(scope): subject`；`i18n` 不是有效 type，用 `docs(i18n)`
- **大文件(>40KB)子智能体易写入截断**：必须外部验证文件大小比率（60-130% 为正常区间）
- **超时不等于失败**：必须用外部脚本验证，不能仅凭任务状态判断

## 关键上下文

### 仓库信息
- **仓库**：`D:\workspace\github\system_prompts_leaks`
- **性质**：纯 Markdown 内容仓库，收集 AI 产品系统提示词泄露。无构建/测试/lint/源代码
- **Git**：`origin`→tonydeng/system_prompts_leaks（fork），`upstream`→asgeirtj/system_prompts_leaks，分支 `main`
- **Main HEAD**：`ddc16e6`（P0 修复已合并并推送），clean
- **活跃 worktree**：`D:\workspace\github\spl-fix-all`，分支 `fix-all`，HEAD=`ddc16e6`，254 个文件已修改未提交
- **.gitattributes**：`*.md -whitespace`（.md 空白警告是有意为之，保留原文保真度）

### 声明头格式（标准）
```
> **说明**：本文件为英文原文（`{原文件名}`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。
```
- **头部位置规则**：产品系统提示词 `-zh.md` 必须有声明头（首行或 YAML front matter 之后）；元文档（README/CONTRIBUTING/AGENTS/SKILL-zh.md）不应有声明头

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
