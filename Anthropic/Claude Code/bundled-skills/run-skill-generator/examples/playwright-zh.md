> **说明**：本文件为英文原文（`playwright.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 示例：浏览器驱动的 web 应用

你有一个向浏览器提供 HTML 的开发服务器。无头容器中的智能体无法打开浏览器窗口 — 所以"运行应用"意味着启动开发服务器，针对它驱动无头 Chromium，并生成证明页面已渲染的截图。

不要编写浏览器驱动程序。使用 `chromium-cli`。

## 开发服务器

找到开发命令（`package.json` `scripts.dev`、`Makefile`、README），在后台启动它，并等待它实际服务：

```bash
npm run dev &   # 或 yarn dev, pnpm dev, make serve, ./dev.sh
echo $! > /tmp/dev.pid
timeout 30 bash -c 'until curl -sf http://localhost:3000 >/dev/null; do sleep 1; done'
```

不要 `sleep 5` — 轮询端口。在重新启动之前用 `kill $(cat /tmp/dev.pid)`（或 `pkill -f 'npm run dev'`）停止，否则下一次运行会遇到 `EADDRINUSE`。

## 驱动

`chromium-cli` 是无头 Chromium REPL。将脚本管道传输到 stdin：

```bash
chromium-cli --session app <<'EOF'
nav http://localhost:3000
wait-for text=Dashboard
screenshot
click button:has-text("New item")
fill input[name="title"] Smoke test
press Enter
wait-for text=Smoke test
screenshot
console --errors
EOF
```

截图落在 `chromium_cli/sessions/app/screenshots/` 中（最新符号链接为 `screenshot.png`）。这就是整个循环：`nav` → `wait-for` 你需要的元素 → 操作（`click` / `fill` / `type` / `press`）→ `screenshot` → `console --errors` 检查没有任何抛出。完整命令参考：`chromium-cli` skill，或提示符处的 `help`。

对于迭代调试，在 tmux 下运行它并一次 `send-keys` 一个命令 — 相同的命令，相同的会话。

**如果 `chromium-cli` 不可用：**调整 [electron.md](electron.md) 的 REPL 驱动程序 — 结构和命令转移，但它是 `_electron` 特定的：改为导入 `{ chromium }`，用 `chromium.launch({ args: ['--no-sandbox'] })` 启动，通过 `(await app.newContext()).newPage()` 获取页面，然后 `goto()` 你的开发 URL，并删除仅 Electron 的窗口内省（`.windows()`/`.firstWindow()`/`windows` 命令）。

## 在 skill 中放什么

仅项目特定的位。`chromium-cli` 处理机制。

- **开发命令 + 端口 + 停止。**确切的启动行，它需要的任何环境变量，以及停止它的 `kill`/`pkill`。
- **身份验证。**任何获得登录会话的东西 — `set-cookie` 行、`fill`/`click` 登录序列，或执行 API 舞蹈并发出 cookie 的助手脚本。
- **一个代表性交互。**不是整个应用 — 一条证明它正在运行的路径，以截图结束。
- **应用特定的 gotchas。**只有你实际遇到的那些。

## 经常出现的 Gotchas

- **React 受控输入。** `eval el.value = '…'` 不会触发 React 的 onChange。使用 `fill` / `type` — 它们通过 Playwright 的输入管道。
- **Websockets / 长轮询。** `wait-idle` 永远不会解决。`wait-for` 你实际需要的元素。
- **慢首次绘制。** Vite/Next 按需编译路由；第一次 `nav` 可能需要 10s+。`wait-for` 处理它；原始 `sleep` 不处理。
- **`screenshot-element <sel>`** 裁剪到一个元素 — 当差异在特定组件中，而不是整个页面时使用它。
- **在声明成功之前检查 `console --errors`。**页面可以渲染其外壳，而每个数据获取都 500。
