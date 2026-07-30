# HANDOFF — 跨会话交接文档

> 最后更新：2026-07-30 | 会话主题：system_prompts_leaks 中文镜像（-zh.md）批量生成

## 当前任务

为 `D:\workspace\github\system_prompts_leaks` 仓库的所有英文 .md 文件创建中文镜像 `-zh.md`（同目录，加 `-zh` 后缀）。这是 AGENTS.md 中文档化的约定，本次会话执行批量生成。

**整体进度**：✅ 全部可翻译文件（<200KB）已完成。8 个 >200KB 巨型文件按用户决策跳过。

## 已完成工作

### 基础设施
- **AGENTS.md 创建**：仓库协作准则，含中文镜像约定 + .handoff 机制 + .gitignore 规则
- **.gitignore 创建**：忽略 `.omo/`、编辑器/IDE 临时文件、OS 垃圾文件。`.handoff/` 不被忽略
- **.handoff/ 目录初始化**：current.md + archive/ 子目录

### 中文镜像生成统计

| 厂商 | 英文 .md | 已完成 -zh.md | 状态 |
|------|---------|--------------|------|
| Anthropic | 118 | 110 | ✅ 完成（8 个 >200KB 巨型文件跳过） |
| OpenAI | 87 | 87 | ✅ 完成（codex-full 362KB 跳过） |
| Google | 22 | 22 | ✅ 完成 |
| xAI | 11 | 11 | ✅ 完成 |
| Microsoft | 5 | 5 | ✅ 完成 |
| Perplexity | 3 | 3 | ✅ 完成 |
| Cursor | 1 | 1 | ✅ 完成 |
| Meta | 1 | 1 | ✅ 完成 |
| Mistral | 2 | 2 | ✅ 完成 |
| Notion | 1 | 1 | ✅ 完成 |
| Qwen | 1 | 1 | ✅ 完成 |
| Misc | 23 | 23 | ✅ 完成 |
| 根目录 | 2 (README + AGENTS) | 1 (README-zh) | ✅ AGENTS.md 已是中文无需镜像 |
| .github | 1 (CONTRIBUTING) | 1 | ✅ 完成 |
| **合计** | **277** | **269** | **✅ 全部可翻译文件完成** |

### 最终验证结果
- 所有已提交 -zh.md 文件 ratio 在 60-130% 正常区间（多数 86-107%）
- 系统提示词文件首行声明头正确
- 元文档（README/CONTRIBUTING/SKILL/docs）无声明头，完整翻译
- 占位符（`${insightsJson}`、`{{Region}}`、`{model_name}` 等）原样保留

### 8 个 >200KB 巨型文件（用户已确认跳过）
- Anthropic\claude-cowork.md (276.9KB)
- Anthropic\claude-design.md (201.6KB)
- Anthropic\claude-fable-5.md (230.6KB)
- Anthropic\claude-opus-5.md (222.7KB)
- Anthropic\Official\all.md (433.6KB)
- Anthropic\raw\claude-opus-4.6-raw.md (228.7KB)
- Anthropic\raw\claude-sonnet-4.6-raw.md (232.1KB)
- OpenAI\Codex\codex-full.md (362KB)

### 本会话提交记录（translate-continue 分支）
- ff8fb14, 9811b6a, 60e84f2, f6ca710, 1854dbc, 63ac2a5, b5b8a53, 9d589f8, c9111e1
- 总计 ~240+ 个 -zh.md 文件已提交

## 当前约束（活跃限制，非历史经验）

- ⛔ **巨型文件（>200KB）跳过策略**：用户明确决策跳过 8 个巨型文件。如需处理需用户重新确认，且子智能体几乎无法一次性完成，需分段策略
- ⛔ **不可信子智能体自我报告**：必须用外部脚本验证文件存在性 + 大小比率（60-130% 为正常区间）+ 标题数对比
- ⛔ **`.handoff/` 绝不能加入 .gitignore**：按 AGENTS.md 规则须版本控制
- ⛔ **Worktree 工作约定**：所有开发在 `spl-translate-continue` worktree 进行，main 保持干净，通过合并更新

## 当前卡点

- 无。翻译任务已完成。

## 下一步计划

1. **合并 worktree 到 main**（优先级：高）：切换到主仓库 `D:\workspace\github\system_prompts_leaks`，merge `translate-continue` 分支
2. **Push 到 origin**（优先级：高）：推送 main 到 `tonydeng/system_prompts_leaks`
3. **(可选) 创建 PR 到 upstream**（优先级：中）：向 `asgeirtj/system_prompts_leaks` 提 PR
4. **(可选) 处理 8 个 >200KB 巨型文件**（优先级：低）：需用户确认，需分段翻译策略

## 踩坑记录（绝对不要再踩）

- **大文件(>40KB)子智能体易写入截断**：Meta(56.9KB)和Notion(46.7KB)试点就遇到截断，Anthropic 波次2 的 8 个 170-190KB 文件全部严重截断（比率 3.7-50.2%）。必须外部验证
- **子智能体可能委托给只读 agent 导致 0 产出**：批次10 (bg_42d82c88) 遇到 111KB 文件后走偏，试图委托给 explore agent（只读），5 文件全部 0 创建。Prompt 中必须明确禁止委托
- **子智能体可能混入其他文件内容**：试点 Meta 文件修复时错误将 README 内容写入 meta-ai-zh.md 开头（行3-276）。必须检查文件开头和末尾内容是否对应原文
- **超时不等于失败**：批次5 (bg_3840ab20) 超时 ERROR 但外部验证显示 16 文件全部已写入磁盘。必须用外部脚本验证，不能仅凭状态判断
- **单文件任务模式对大文件更可靠**：>100KB 文件采用 1 file per task 模式，Wave 1-4 全部成功（86-107% ratio）
- **background_cancel 可能失败**：部分任务无法取消（已完成或状态异常），需检查返回状态
- **Windows PowerShell 中文输出乱码**：控制台编码问题，但不影响实际文件内容。验证脚本输出乱码时需注意数据本身是正确的
- **commit-msg hook 格式**：要求 `:emoji: type(scope): subject`；`i18n` 不是有效 type，用 `docs(i18n)`
- **`.gitattributes` whitespace 警告是预期的**：`*.md -whitespace`，LF→CRLF 警告不要修复

## 关键上下文

### 仓库信息
- **仓库**：`D:\workspace\github\system_prompts_leaks`
- **性质**：纯 Markdown 内容仓库，收集 AI 产品系统提示词泄露。无构建/测试/lint/源代码
- **Git**：`origin`→tonydeng/system_prompts_leaks（fork），`upstream`→asgeirtj/system_prompts_leaks，分支 `main`
- **Worktree**：`D:\workspace\github\spl-translate-continue`，分支 `translate-continue`
- **.gitattributes**：`*.md -whitespace`（.md 空白警告是有意为之，保留原文保真度）
- **原文照贴原则**：系统提示词文件必须原样粘贴，禁止摘要/改写/润色。`{model_name}` 等占位符必须保留

### 中文镜像约定（AGENTS.md 文档化）
- **命名**：`{filename}.md` → `{filename}-zh.md`（同目录）
- **系统提示词 -zh.md**：第一行声明头 `> **说明**：本文件为英文原文（\`{原文件名}\`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 \`{model_name}\`）保持原样不译。`
- **元文档（README/CONTRIBUTING 等）**：无声明头，完整翻译
- **已经是中文的 .md**（如 AGENTS.md）：无需镜像
- **纯中文翻译**，占位符/代码块不翻译，Markdown 格式保留

### 委派策略（writing 类别，run_in_background=true）— 已验证有效的模式
- **小文件（<40KB）**：每批 15-20 文件，子智能体可可靠完成
- **中文件（40-100KB）**：每批 5-6 文件，需外部验证
- **大文件（100-200KB）**：1 file per task 模式，成功率高（86-107% ratio）
- **巨型文件（>200KB）**：跳过，用户已确认

### 验证脚本模板（PowerShell）
```powershell
# 验证 -zh.md 文件完整性和比率
$enFiles = Get-ChildItem -Path $vendorPath -Filter "*.md" -Recurse -File | Where-Object { $_.Name -notmatch '-zh\.md$' }
foreach ($f in $enFiles) {
    $zhPath = $f.FullName -replace '\.md$', '-zh.md'
    if (-not (Test-Path $zhPath)) { Write-Output "MISSING: $($f.Name)" }
    else {
        $zh = Get-Item $zhPath
        $ratio = [math]::Round(($zh.Length / $f.Length) * 100, 1)
        if ($ratio -lt 60) { Write-Output "TRUNCATED: $($zh.Name) ${ratio}%" }
    }
}
```

### HANDOFF 机制（借自 Scorpius）
- **位置**：`<repo-root>/.handoff/current.md`
- **归档**：会话结束前复制到 `.handoff/archive/{YYYY-MM-DD}.md`（同日重名加 `-N`）
- **读取验证**：新会话读取后必须输出 📋 HANDOFF 已恢复确认块
- **版本控制**：`.handoff/` 纳入 Git，不加入 .gitignore
- **机制分工**：.handoff/current.md（跨会话短期）vs compress（会话内）vs Supermemory（跨会话长期）
