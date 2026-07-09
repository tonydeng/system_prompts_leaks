# Claude Code `/code-review` 斜杠命令

Claude Code 内置 `/code-review` 技能（v2.1.198）背后的提示模板。磁盘上没有此命令的 skill.md——文本被编译到 Claude Code 二进制文件中，并在命令运行时作为用户消息块注入到对话中。此处内容是从二进制文件中提取的，并针对实时 API 捕获（MITM 代理，2026-07-02）进行了字节验证。

## 用法

`/code-review [level] [target]`——level 是 `low`、`medium`、`high`（默认）、`xhigh`、`max` 之一；target 是可选的 PR 编号、分支、引用范围或路径。`--comment` 将发现作为内联 PR 评论发布；`--fix` 将它们应用到工作树。`ultra` 运行单独的多代理云审查。

当传递参数时，注入的块以 `Review target: `<args>`` 为前缀，后跟空行；没有参数时，它直接从 effort 标题开始。

## 文件

| 文件 | 流水线 |
|---|---|
| `low.md` | 1 次 diff 传递，无子代理，无验证 → ≤4 个发现 |
| `medium.md` | 8 个查找角度 × 6 个候选，1 票验证（精确度调整）→ ≤8 个发现 |
| `high.md` | 8 个查找角度 × 6 个候选，1 票验证（召回偏向）→ ≤10 个发现——默认 |
| `xhigh.md` | 10 个查找角度 × 8 个候选，验证，缺口扫描 → ≤15 个发现 |
| `max.md` | 与 xhigh 相同，只是标题措辞不同；仅 API 推理努力不同 |
| `report-findings-tool.md` | 附加到主机 UI 呈现类型化发现的会话的 `ReportFindings` 工具（描述 + JSON 模式）——JSON 数组的替代输出通道 |

查找角度：A 逐行 diff 扫描，B 移除行为审计器，C 跨文件追踪器，D 语言陷阱专家（仅 xhigh/max），E 包装器/代理正确性（仅 xhigh/max），加上复用、简化、效率、高度和约定（CLAUDE.md 违规）。

二进制文件还包含此处未重现的兄弟变体：通过 `ReportFindings` 工具调用而不是 JSON 数组报告的输出模式、工件发布步骤（发现呈现为可共享的 HTML 页面），以及在启用工作流时在 high/xhigh/max 使用的基于工作流的编排（每个正确性角度一个查找器，一个合并的清理查找器，每个不同的 file:line 一个验证器，然后综合）。

当前规范副本：`Anthropic/Claude Code/bundled-skills/code-review.md`（带有 frontmatter 的 high 变体）。
