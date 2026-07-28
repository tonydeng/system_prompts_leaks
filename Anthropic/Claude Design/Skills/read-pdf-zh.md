> **说明**：本文件为英文原文（`read-pdf.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 读取 PDF

在 run_script 中读取 PDF，使用浏览器版本的 pdf-parse（锁定 @2.4.5）：

```js
const { PDFParse } = await import('https://cdn.jsdelivr.net/npm/pdf-parse@2.4.5/dist/pdf-parse/web/pdf-parse.es.js');
PDFParse.setWorker('https://cdn.jsdelivr.net/npm/pdf-parse@2.4.5/dist/pdf-parse/web/pdf.worker.min.mjs');

const blob = await readFileBinary('document.pdf');
const parser = new PDFParse({ data: new Uint8Array(await blob.arrayBuffer()) });
const result = await parser.getText();
log(result.text);
```

SRI 哈希（供参考——动态 import() 无法在运行时强制 SRI）：
- `pdf-parse.es.js` — sha384-J7LMAGioDDEBxHBcdxpU9NGtQu2/iLuSGyD3HsO5aYDJ0BAisPtpTYGc5XcB7UcI
- `pdf.worker.min.mjs` — sha384-zdw/VQhL/JrSgvr/Omai4B8USJUC6AQXr/4YW01OlVWutKoGvg34AOFCRsO1dGJr
