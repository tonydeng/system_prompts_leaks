> **说明**：本文件为英文原文（`electron.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 示例：Electron / 桌面 GUI 应用

Electron 应用有一个窗口。无头容器中的未来智能体无法看到窗口。所以你在这里的交付物不是说"`npm start` 打开一个窗口"的 markdown 文件 — 它是一个**驱动程序脚本**，它在 xvfb 下启动应用，暴露命令 REPL（点击、类型、截图），并让智能体通过发送文本行来操作 UI。

然后 skill 的 `SKILL.md` 成为该驱动程序的简短手册。

## 你在构建什么

```
apps/desktop/
  .claude/skills/run-desktop/
    SKILL.md               ← 简短。"运行驱动程序，这里是命令"
    driver.mjs             ← REPL：stdin 命令 → Playwright 操作
```

驱动程序就是产品。没有它，skill 描述了一个智能体永远无法触摸的 GUI。

**升级路径：**如果驱动程序成长为项目的真实 e2e 套件想要共享的启动助手，将其移动到 `e2e-playwright/driver.mjs`（或 `scripts/drive.mjs`）并更新 skill 的路径。skill 保持在 `.claude/skills/run-desktop/`；驱动程序找到更好的家。

## 步骤 1 — 让应用在 xvfb 下完全启动

这通常是最难的部分，并产生大多数 Gotchas。README 将说"仅 macOS/Windows"。忽略那个。安装 xvfb + Chromium 共享库，找到 Electron 二进制文件，并启动它：

```bash
apt-get install -y xvfb libnss3 libgbm1 libasound2t64 libgtk-3-0 \
  libxss1 libxkbcommon0 libatk-bridge2.0-0 libcups2 libdrm2

# 首先构建应用。通常"dev"脚本是 electron-forge，它执行 Vite/webpack 构建
# 然后启动。你只需要构建：
npm install
npx electron-forge start &   # 构建 .vite/build/ 或 dist/
sleep 20 && kill %1          # 构建后杀死它 — 你将自己启动

# 现在尝试原始启动
xvfb-run -a node -e "
  const { _electron } = require('playwright-core');
  _electron.launch({
    executablePath: './node_modules/electron/dist/electron',
    args: ['--no-sandbox', '.'],
    timeout: 30000,
  }).then(app => {
    console.log('launched, windows:', app.windows().map(w => w.url()));
    return app.close();
  });
"
```

迭代直到它启动。每个缺少的 `.so` → 再一个 `apt-get` 包 → 先决条件中再一行。每个启动超时 → 检查 `nodeCliInspect` 熔丝未禁用，检查构建输出存在。

**`--no-sandbox` 在容器中几乎总是需要的。** Electron 的沙箱需要 CAP_SYS_ADMIN 或用户命名空间。默认情况下两者都没有。

## 步骤 2 — 构建 REPL 驱动程序

一旦你可以启动它，将那个一次性脚本变成 REPL。开始最小 — 你将根据需要添加命令。**REPL 是正确的形状**，因为智能体可以在 tmux 内运行它并在每次交互时迭代而无需重新启动（慢）应用。

```javascript
// .claude/skills/run-<unit>/driver.mjs
// <app> 的 REPL 驱动程序。在无头 Linux 上的 xvfb 下运行。
// 为智能体设计：包装在 tmux 中，send-keys 命令，capture-pane 输出。
import { _electron as electron } from 'playwright-core';
import * as readline from 'node:readline';
import * as fs from 'node:fs';
import * as path from 'node:path';

const APP_DIR = path.resolve(import.meta.dirname, '../../..');
const SHOT_DIR = process.env.SCREENSHOT_DIR || '/tmp/shots';
fs.mkdirSync(SHOT_DIR, { recursive: true });

let app = null;
let page = null;   // 你实际交互的窗口/页面

const electronBin = process.platform === 'darwin'
  ? path.join(APP_DIR, 'node_modules/electron/dist/Electron.app/Contents/MacOS/Electron')
  : path.join(APP_DIR, 'node_modules/electron/dist/electron');

const COMMANDS = {
  async launch() {
    if (app) return console.log('already launched');
    app = await electron.launch({
      executablePath: electronBin,
      args: ['--no-sandbox', APP_DIR],
      env: { ...process.env, DISPLAY: process.env.DISPLAY || ':99' },
      timeout: 30_000,
    });
    // Electron 没有干净的"已加载"信号 — 这个睡眠是一个盲目的猜测。
    // 一旦你知道此应用的就绪看起来像什么，就替换为轮询：
    // 等待直到 windows() 包括预期的 URL，或在 firstWindow() 上 waitForSelector。
    await new Promise(r => setTimeout(r, 8_000));
    // 找到真正的 UI 页面。通常不是 firstWindow() — 可能是
    // 启动屏幕，或者真实内容在 BrowserView 覆盖中。
    page = app.windows().find(w => !w.url().startsWith('devtools://'))
        ?? await app.firstWindow();
    console.log('launched.', app.windows().length, 'windows:');
    for (const w of app.windows()) console.log(' ', w.url());
  },

  async ss(name) {
    if (!page) return console.log('ERROR: launch first');
    const f = path.join(SHOT_DIR, (name || `ss-${Date.now()}`) + '.png');
    await page.screenshot({ path: f });
    console.log('screenshot:', f);
  },

  // 通过 evaluate() 点击，而不是 locator.click()。如果内容存在于
  // 主窗口之上的 BrowserView 中，Playwright 的坐标数学会击中错误的层。
  // DOM .click() 总是有效。
  async click(sel) {
    if (!page) return console.log('ERROR: launch first');
    const r = await page.evaluate(s => {
      const el = document.querySelector(s);
      if (!el) return 'NOT_FOUND';
      el.click(); return 'OK';
    }, sel);
    console.log('click', sel, '→', r);
  },

  async 'click-text'(text) {
    if (!page) return console.log('ERROR: launch first');
    const r = await page.evaluate(t => {
      const els = [...document.querySelectorAll('button, a, [role="button"]')];
      const el = els.find(e => e.textContent?.trim() === t)
              ?? els.find(e => e.textContent?.includes(t));
      if (!el) return 'NOT_FOUND';
      el.click(); return 'OK: ' + el.tagName;
    }, text);
    console.log('click-text', JSON.stringify(text), '→', r);
  },

  async type(text)  { if (page) await page.keyboard.type(text, { delay: 30 }); },
  async press(key)  { if (page) await page.keyboard.press(key); },

  async wait(sel) {
    if (!page) return console.log('ERROR: launch first');
    try { await page.waitForSelector(sel, { timeout: 10_000 }); console.log('found:', sel); }
    catch { console.log('TIMEOUT:', sel); }
  },

  async eval(expr) {
    if (!page) return console.log('ERROR: launch first');
    try { console.log(JSON.stringify(await page.evaluate(expr))); }
    catch (e) { console.log('ERROR:', e.message); }
  },

  async text(sel) {
    if (!page) return console.log('ERROR: launch first');
    console.log(await page.evaluate(
      s => (s ? document.querySelector(s) : document.body)?.innerText ?? '(null)',
      sel || null));
  },

  // 内省：对于弄清楚哪个窗口/webContents 实际具有 UI 至关重要。
  // Electron 应用经常生成几个。
  async windows() {
    if (!app) return console.log('ERROR: launch first');
    for (const w of app.windows()) console.log(' ', w.url());
    const wcs = await app.evaluate(({ webContents }) =>
      webContents.getAllWebContents().map(w => ({ id: w.id, type: w.getType(), url: w.getURL() })));
    console.log('webContents:');
    for (const w of wcs) console.log(` [${w.id}] ${w.type}: ${w.url}`);
  },

  async quit() { if (app) await app.close().catch(()=>{}); app = null; page = null; },
  help() { console.log('commands:', Object.keys(COMMANDS).join(', ')); },
};

// 阻止 Electron 窃取 stdin — 使用原始 fd。
const stdin = fs.createReadStream(null, { fd: fs.openSync('/dev/stdin', 'r') });
const rl = readline.createInterface({ input: stdin, output: process.stdout, prompt: 'driver> ' });

rl.on('line', async line => {
  const [cmd, ...rest] = line.trim().split(/\s+/);
  if (!cmd) return rl.prompt();
  const fn = COMMANDS[cmd];
  if (!fn) { console.log('unknown:', cmd, '— try: help'); return rl.prompt(); }
  try { await fn(rest.join(' ')); } catch (e) { console.log('ERROR:', e.message); }
  if (cmd === 'quit') { rl.close(); process.exit(0); }
  rl.prompt();
});
rl.on('close', async () => { await COMMANDS.quit(); process.exit(0); });

console.log('<app> driver — "help" for commands, "launch" to start');
rl.prompt();
```

**这是一个起始骨架。**当你尝试到达应用的有趣部分时，你将添加应用特定的命令：导航到特定视图、聚焦奇怪的输入类型、绕过身份验证门、无论什么。这些命令编码了来之不易的知识 — 保留它们。

## 步骤 3 — 自己使用它，通过 tmux

以与下一个智能体相同的方式运行驱动程序：

```bash
tmux new-session -d -s app -x 200 -y 50
tmux send-keys -t app 'cd /workspace/apps/desktop && xvfb-run -a node .claude/skills/run-desktop/driver.mjs' Enter
timeout 20 bash -c 'until tmux capture-pane -t app -p | grep -q "driver>"; do sleep 0.2; done'
tmux send-keys -t app 'launch' Enter
timeout 60 bash -c 'until tmux capture-pane -t app -p | grep -q "launched"; do sleep 0.2; done'
tmux send-keys -t app 'ss 01-landing' Enter
timeout 10 bash -c 'until tmux capture-pane -t app -p | grep -q "screenshot:"; do sleep 0.2; done'
tmux send-keys -t app 'windows' Enter    # 哪个页面有真正的 UI？
tmux capture-pane -t app -p
```

然后实际打开 `/tmp/shots/01-landing.png`。它是应用吗？它是空白吗？它是登录屏幕吗？这些每一个都告诉你下一步做什么。

继续 — 点击进入主要功能，填写表单，查看结果显示，截图它。驱动程序增长你需要的任何命令（`focus-input`、`goto-settings`、`login-as-test-user`…）。当一个真正的流程端到端工作时，你就完成了构建并准备好编写。

## 步骤 4 — 编写 SKILL.md

保持简短。驱动程序是实质；`SKILL.md` 是手册。有效的结构：

> ---
> name: run-desktop
> description: Build, run, and drive the <app> Electron desktop app. Use when asked to start the desktop app, take a screenshot of it, build it, or interact with its UI.
> ---
>
> <App> 是一个 Electron 桌面应用。对于智能体/自动化使用，通过 xvfb 下的 Playwright REPL 在 `.claude/skills/run-desktop/driver.mjs` 驱动它。启动很慢（~10s），有趣的 UI 存在于 BrowserView 中，而不是主窗口 — 驱动程序处理两者。
>
> 所有路径都相对于 `apps/desktop/`。
>
> ## 先决条件
>
> ```bash
> apt-get install -y xvfb libnss3 libgbm1 libasound2t64 libgtk-3-0 \
>   libxss1 libxkbcommon0 libatk-bridge2.0-0 libcups2 libdrm2
> ```
>
> ## 构建
>
> ```bash
> npm install
> npx electron-forge start   # 构建 .vite/build/ — 构建后 Ctrl-C
> # <你必须应用的任何修补：sed 功能门等>
> ```
>
> ## 运行（智能体路径）
>
> ```bash
> cd apps/desktop
> xvfb-run -a node .claude/skills/run-desktop/driver.mjs
> ```
>
> 包装在 tmux 中以进行交互式使用：
>
> ```bash
> tmux new-session -d -s app -x 200 -y 50
> tmux send-keys -t app 'cd apps/desktop && xvfb-run -a node .claude/skills/run-desktop/driver.mjs' Enter
> timeout 20 bash -c 'until tmux capture-pane -t app -p | grep -q "driver>"; do sleep 0.2; done'
> tmux send-keys -t app 'launch' Enter
> timeout 60 bash -c 'until tmux capture-pane -t app -p | grep -q "launched"; do sleep 0.2; done'
> tmux send-keys -t app 'ss landing' Enter
> tmux capture-pane -t app -p
> ```
>
> 截图落在 `/tmp/shots/` 中（覆盖：`SCREENSHOT_DIR`）。
>
> ### 命令
>
> | 命令 | 它做什么 |
> |---|---|
> | `launch` | 启动应用，等待窗口 |
> | `ss [name]` | 截图 → `/tmp/shots/<name>.png` |
> | `click <css-sel>` | 点击元素（通过 DOM，而不是坐标 — 见 Gotchas） |
> | `click-text <text>` | 点击包含文本的按钮/链接 |
> | `type <text>` / `press <key>` | 键盘输入 |
> | `wait <css-sel>` | 等待元素，10s 超时 |
> | `eval <js>` | 在页面中评估，打印 JSON |
> | `text [css-sel]` | 打印 innerText |
> | `windows` | 列出所有窗口 + webContents（找到真正的 UI） |
> | `quit` | 关闭应用，退出 |
>
> 加上你构建的任何应用特定命令：`<your-command>` — <它做什么>。
>
> ## 运行（人类路径）
>
> ```bash
> npm start   # 打开窗口；无头无用。Ctrl-C 退出。
> ```
>
> ## Gotchas
>
> - **<你遇到的特定奇怪事物>** — <为什么> → <修复/解决方法>
> - <等等 — 只有你实际遇到的事情，而不是通用建议>
>
> ## 故障排除
>
> - **启动超时（30s）：**构建输出缺失？ → 重新运行构建步骤。`nodeCliInspect` 熔丝禁用？ → Playwright 无法附加；不要在开发构建中禁用该熔丝。
> - **"Missing X server"：**忘记 `xvfb-run`。无头 Linux 需要它。
> - **陈旧的 Xvfb 锁：** `rm -f /tmp/.X*-lock; pkill Xvfb`
> - <你实际遇到的任何其他事情>

## 你将遇到的障碍（它们进入 Gotchas）

这些是来自真实 Electron 应用的真实模式。你将遇到一些子集：

- **`firstWindow()` 给你一个启动/加载屏幕，**而不是应用。等待更长时间，或通过 URL 找到正确的页面，或等待仅在应用实际就绪时出现的特定选择器。

- **真正的 UI 在 BrowserView 中，而不是 BrowserWindow 中。** Playwright 将其视为具有不同 URL 的单独"窗口"。`windows` 命令正是为了弄清楚这一点而存在。`getBrowserViews()` 在较新的 Electron 上也可能返回空 — 改用 `webContents.getAllWebContents()`。

- **`locator.click()` 点击错误的东西。** Playwright 计算相对于主窗口的点击坐标。如果你的内容在 BrowserView 覆盖中，这些坐标会击中它后面的窗口。驱动程序骨架因此使用 `page.evaluate(el => el.click())` — DOM 点击完全绕过坐标。

- **功能门阻止你需要测试的东西。**应用检查计划层级、环境标志或烘焙到 SSR HTML 中的功能标志。找到检查发生的位置（在构建输出中 grep 门名称）并为你的本地运行修补它 — 对构建输出的 `sed`、环境变量覆盖，或（对于 SSR 嵌入标志）通过 CDP `Fetch.enable` 拦截响应并飞行中重写它。确切文档化你修补了什么以及为什么。

- **contentEditable 输入**（ProseMirror、Tiptap、Slate）不是 `<textarea>`。`fill()` 不会工作。聚焦元素，然后使用 `keyboard.type()`。如果应用有这些，添加 `focus <sel>` 命令。

- **Electron 窃取 stdin。**骨架中的 `fs.openSync('/dev/stdin', 'r')` + `createReadStream` 技巧保护你的 REPL 输入。

- **原生模块无法加载**（密钥链、通知等）。通常不是致命的 — 核心应用运行，那些功能 no-op。注意它并继续。
