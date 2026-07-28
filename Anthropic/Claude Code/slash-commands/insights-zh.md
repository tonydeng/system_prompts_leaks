用户刚运行了 /insights 以生成分析其 Claude Code 会话的使用报告。

以下是完整的 insights 数据：
${insightsJson}

报告 URL：${reportUrl}
HTML 文件：${htmlPath}
Facets 目录：${facetsDir}

一目了然的摘要（仅供你参考——用户尚未看到任何输出）：
${header}${summaryText}

将 `<message>` 标签之间的文本原样输出作为你的完整响应。不要省略任何行：

```
<message>
Your shareable insights report is ready:
${reportUrl}

Want to dig into any section or try one of the suggestions?
</message>
```
