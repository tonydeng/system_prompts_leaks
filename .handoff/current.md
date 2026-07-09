# HANDOFF — 跨会话交接文档

> 最后更新：2026-07-09 | 会话主题：system_prompts_leaks 中文镜像（-zh.md）批量生成

## 当前任务

为 `D:\workspace\github\system_prompts_leaks` 仓库的所有英文 .md 文件创建中文镜像 `-zh.md`（同目录，加 `-zh` 后缀）。这是 AGENTS.md 中文档化的约定，本次会话执行批量生成。

**整体进度**：114/289 文件已完成（39.4%），覆盖 8 个小厂商试点 + Anthropic 大部分。剩余 OpenAI/Google/xAI/Microsoft/Misc 全部 + Anthropic 大文件修复。

## 已完成工作

### 基础设施
- **AGENTS.md 创建**（D:\workspace\github\system_prompts_leaks\AGENTS.md，约 127 行）：仓库协作准则，含中文镜像约定 + .handoff 机制（借自 Scorpius）+ .gitignore 规则
- **.gitignore 创建**：忽略 `.omo/`、编辑器/IDE 临时文件、OS 垃圾文件。**注意：`.handoff/` 不被忽略**（须版本控制）
- **.handoff/ 目录初始化**：本文件 + archive/ 子目录

### 中文镜像生成统计

| 厂商 | 英文 .md | 已完成 -zh.md | 状态 |
|------|---------|--------------|------|
| Anthropic | 118 | 103 | ⚠️ 15 缺失 + 8 截断待修复 |
| OpenAI | 87 | 0 | ❌ 未开始 |
| Google | 22 | 0 | ❌ 未开始 |
| xAI | 11 | 0 | ❌ 未开始 |
| Microsoft | 5 | 0 | ❌ 未开始 |
| Perplexity | 3 | 3 | ✅ 完成 |
| Cursor | 1 | 1 | ✅ 完成 |
| Meta | 1 | 1 | ✅ 完成（已修复 README 垃圾内容混入） |
| Mistral | 2 | 2 | ✅ 完成 |
| Notion | 1 | 1 | ✅ 完成 |
| Qwen | 1 | 1 | ✅ 完成 |
| Misc | 23 | 0 | ❌ 未开始 |
| 根目录 | 2 (README + AGENTS) | 1 (README-zh) | ✅ AGENTS.md 已是中文无需镜像 |
| .github | 1 (CONTRIBUTING) | 1 | ✅ 完成 |
| **合计** | **277** | **114** | **39.4%** |

### Anthropic 详细状态

**已完成且验证通过**：95 文件（波次1 的 84 个 <40KB 小文件 + 波次2 的 11 个 40-200KB 文件正常）

**15 个缺失文件**（MISSING，未生成）：

| # | 文件路径 | 大小 | 原因 |
|---|---------|------|------|
| 1 | Anthropic\claude-cowork.md | 276.9KB | 巨型文件跳过（用户决策） |
| 2 | Anthropic\claude-design.md | 463.6KB | 巨型文件跳过（用户决策） |
| 3 | Anthropic\Claude Code\claude-code-2.1.172-fable-5.md | 94.2KB | 批次9 未完成 |
| 4 | Anthropic\Claude Code\claude-code-2.1.172-opus-4.6.md | 117.4KB | 批次10 失败重启未完成 |
| 5 | Anthropic\Claude Code\deferred-tools.md | 83.9KB | 批次8 未完成 |
| 6 | Anthropic\Claude Code\bundled-skills\claude-api.md | 536.4KB | 巨型文件跳过（用户决策） |
| 7 | Anthropic\Claude Code\bundled-skills\update-config.md | 144.7KB | 批次10 失败重启未完成 |
| 8 | Anthropic\Official\all.md | 389.3KB | 巨型文件跳过（用户决策） |
| 9 | Anthropic\old\claude-3.7-full-system-message-with-all-tools.md | 110.7KB | 批次10 失败重启未完成 |
| 10 | Anthropic\old\claude-3.7-sonnet-full-system-message-humanreadable.md | 112.3KB | 批次10 失败重启未完成 |
| 11 | Anthropic\old\claude-4.1-opus-thinking.md | 105.3KB | 批次9 未完成 |
| 12 | Anthropic\old\claude-4.5-sonnet.md | 142.9KB | 批次10 失败重启未完成 |
| 13 | Anthropic\old\claude-opus-4.5.md | 90.8KB | 批次8 未完成 |
| 14 | Anthropic\raw\claude-opus-4.6-raw.md | 228.7KB | 巨型文件跳过（用户决策） |
| 15 | Anthropic\raw\claude-sonnet-4.6-raw.md | 232.1KB | 巨型文件跳过（用户决策） |

**8 个截断文件**（已生成但比率<60%，需修复重做）：

| # | 文件路径 | 比率 | 原→译 |
|---|---------|------|-------|
| 1 | Anthropic\claude-opus-4.6-zh.md | 5.1% | 179KB→9.2KB |
| 2 | Anthropic\claude-opus-4.7-zh.md | 3.7% | 181.5KB→6.6KB |
| 3 | Anthropic\claude-opus-4.8-zh.md | 4.1% | 182.9KB→7.4KB |
| 4 | Anthropic\claude-sonnet-4.6-zh.md | 20.3% | 173.8KB→35.4KB |
| 5 | Anthropic\claude-sonnet-5-zh.md | 18.7% | 187.8KB→35.2KB |
| 6 | Anthropic\claude-fable-5-zh.md | 41.8% | 187KB→78.2KB |
| 7 | Anthropic\Claude Code\claude-code-2.1.172-opus-4.8-zh.md | 50.2% | 89.9KB→45.1KB |
| 8 | Anthropic\old\claude-3.7-sonnet-zh.md | 32.6% | 110.6KB→36.1KB |

### 6 个巨型文件（用户已确认跳过）
- Anthropic\claude-cowork.md (276.9KB)
- Anthropic\claude-design.md (463.6KB)
- Anthropic\Official\all.md (389.3KB)
- Anthropic\Claude Code\bundled-skills\claude-api.md (536.4KB)
- Anthropic\raw\claude-opus-4.6-raw.md (228.7KB)
- Anthropic\raw\claude-sonnet-4.6-raw.md (232.1KB)

（注：gpt-5.5-full 362KB 在 OpenAI 目录，尚未开始）

## 当前约束（活跃限制，非历史经验）

- ⛔ **巨型文件（>200KB）跳过策略**：用户明确决策跳过 6 个 Anthropic 巨型文件 + gpt-5.5-full。如需处理需用户重新确认，且子智能体几乎无法一次性完成，需分段策略
- ⛔ **大文件（>40KB）子智能体高失败率**：试点和波次2证明，>40KB 文件子智能体易写入截断、超时、甚至混入垃圾内容。8 个截断文件 + 多个超时批次印证此约束
- ⛔ **不可信子智能体自我报告**：必须用外部脚本验证文件存在性 + 大小比率（60-130% 为正常区间）+ 标题数对比。批次10 曾报告 completed 但 5 文件全部 0 创建
- ⛔ **`.handoff/` 绝不能加入 .gitignore**：按 AGENTS.md 规则须版本控制
- ⚠️ **未提交 git**：本次会话所有产出（AGENTS.md + .gitignore + 114 个 -zh.md + .handoff/）均未 commit，等待用户确认

## 当前卡点

- **Anthropic 大文件处理策略未定**：8 个截断文件 + 9 个缺失文件（排除 6 个巨型）需修复/重做，但 >80KB 文件子智能体失败率高。需考虑：①分段翻译再拼接 ②用户手动处理 ③跳过并标注
- **后续厂商未开始**：OpenAI(87)/Google(22)/xAI(11)/Microsoft(5)/Misc(23) 共 148 文件待处理

## 下一步计划

1. **修复 Anthropic 截断文件**（优先级：高）：8 个截断文件需删除后重做。建议对 >80KB 文件采用分段策略：先让子智能体翻译前半部分，验证后继续后半部分
2. **补全 Anthropic 缺失文件**（优先级：高）：9 个非巨型缺失文件（83-145KB），需重新启动批次。注意批次10 的失败教训：禁止子智能体委托给 explore agent
3. **推进 OpenAI**（优先级：高）：87 文件，排除 gpt-5.5-full 362KB。按大小分波次，<40KB 先做
4. **推进其他厂商**（优先级：中）：Google 22 + xAI 11 + Microsoft 5 + Misc 23 = 61 文件
5. **用户确认后提交 git**（优先级：中）：所有产出待提交

## 踩坑记录（绝对不要再踩）

- **大文件(>40KB)子智能体易写入截断**：Meta(56.9KB)和Notion(46.7KB)试点就遇到截断，Anthropic 波次2 的 8 个 170-190KB 文件全部严重截断（比率 3.7-50.2%）。必须外部验证
- **子智能体可能委托给只读 agent 导致 0 产出**：批次10 (bg_42d82c88) 遇到 111KB 文件后走偏，试图委托给 explore agent（只读），5 文件全部 0 创建。Prompt 中必须明确禁止委托
- **子智能体可能混入其他文件内容**：试点 Meta 文件修复时错误将 README 内容写入 meta-ai-zh.md 开头（行3-276）。必须检查文件开头和末尾内容是否对应原文
- **超时不等于失败**：批次5 (bg_3840ab20) 超时 ERROR 但外部验证显示 16 文件全部已写入磁盘。必须用外部脚本验证，不能仅凭状态判断
- **background_cancel 可能失败**：部分任务无法取消（已完成或状态异常），需检查返回状态
- **Windows PowerShell 中文输出乱码**：控制台编码问题，但不影响实际文件内容。验证脚本输出乱码时需注意数据本身是正确的

## 关键上下文

### 仓库信息
- **仓库**：`D:\workspace\github\system_prompts_leaks`
- **性质**：纯 Markdown 内容仓库，收集 AI 产品系统提示词泄露。无构建/测试/lint/源代码
- **Git**：`origin`→tonydeng/system_prompts_leaks（fork），`upstream`→asgeirtj/system_prompts_leaks，分支 `main`
- **.gitattributes**：`*.md -whitespace`（.md 空白警告是有意为之，保留原文保真度）
- **原文照贴原则**：系统提示词文件必须原样粘贴，禁止摘要/改写/润色。`{model_name}` 等占位符必须保留

### 中文镜像约定（AGENTS.md 文档化）
- **命名**：`{filename}.md` → `{filename}-zh.md`（同目录）
- **系统提示词 -zh.md**：第一行声明头 `> **说明**：本文件为英文原文（\`{原文件名}\`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 \`{model_name}\`）保持原样不译。`
- **元文档（README/CONTRIBUTING 等）**：无声明头，完整翻译
- **已经是中文的 .md**（如 AGENTS.md）：无需镜像
- **纯中文翻译**，占位符/代码块不翻译，Markdown 格式保留

### 委派策略（writing 类别，run_in_background=true）
- **小文件（<40KB）**：每批 15-20 文件，子智能体可可靠完成
- **中文件（40-100KB）**：每批 5-6 文件，需外部验证
- **大文件（100-200KB）**：每批 2-5 文件，高失败风险，需分段策略
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
