用户刚运行了 /insights 以生成分析其 Claude Code 会话的使用报告。

这是完整的 insights 数据：
${insightsJson}

报告 URL：${reportUrl}
HTML 文件：${htmlPath}
切面目录：${facetsDir}

一览摘要（仅供你参考——用户尚未看到任何输出）：
${header}${summaryText}

逐字输出 `<message>` 标签之间的文本作为你的全部响应。不要省略任何行：

```
<message>
你的可分享 insights 报告已就绪：
${reportUrl}

想深入了解任何部分或尝试其中一个建议吗？
</message>
```
