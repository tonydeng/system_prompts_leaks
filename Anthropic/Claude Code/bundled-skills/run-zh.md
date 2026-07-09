> **说明**：本文件为英文原文（`run.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

---
name: run
description: 启动并驱动此项目的应用程序以查看更改是否工作。
---

# run 技能

**运行意味着启动实际应用程序并与之交互**——
不是测试套件，不是内部函数的 `import` 和
`console.log`。应用程序作为用户（人类或程序）会遇到它：
CLI 在其命令处，服务器在其套接字处，GUI 在其窗口处。

## 首先：项目技能是否已覆盖此内容？

启动此应用程序的项目技能是仓库的验证路径——
其作者已经从 Linux 容器冷启动并提交了有效的内容：
确切的 `apt-get` 行、环境变量、补丁、驱动程序。
使用它而不是重新发现。

```bash
d=$PWD; while :; do
  grep -Hm1 '^description:' "$d"/.claude/skills/*/SKILL.md 2>/dev/null
  [ -e "$d/.git" ] || [ "$d" = / ] && break
  d=$(dirname "$d")
done
```

- **一个描述启动/驱动此应用程序** → 读取该 SKILL.md
  并逐字遵循。不要意译；不要跳过补丁。
- **大型仓库，几个合理的，没有明确匹配** → 询问用户
  运行哪个单元。
- **过时**（在与你的任务无关的机制上失败） → 告诉
  用户；提议通过 `/run-skill-generator` 刷新它。
- **没有关于运行的内容** → 回退到以下模式。

## 否则：匹配形状，使用模式

选择最接近你项目的行。每个示例都遍历
启动 + 第一次交互；忽略任何尾随的"编写技能"
部分——你使用的是配方，而不是编写一个。

| 项目类型 | 处理 | 示例 |
|---|---|---|
| CLI 工具 | 直接调用、退出代码、stdin/stdout | examples/cli.md |
| Web 服务器 / API | 后台启动 + `curl` 烟雾 | examples/server.md |
| TUI / 交互式终端 | tmux `send-keys` / `capture-pane` | examples/tui.md |
| Electron / 桌面 GUI | xvfb 下的 Playwright `_electron` REPL | examples/electron.md |
| 浏览器驱动 | 开发服务器 + `chromium-cli` 脚本 | examples/playwright.md |
| 库 / SDK | 包边界的导入和调用烟雾脚本 | examples/library.md |

如果没有任何适合，从最接近的匹配开始并调整。对于 Web
应用程序，examples/playwright.md —— 使用 `chromium-cli` 驱动它，不需要自定义
驱动程序。对于桌面应用程序，examples/electron.md —— 它具有
`_electron` REPL 驱动程序骨架和 tmux 包装。

## 驱动它，不要只是启动它

没有交互的启动证明入口点已解析。那不是
运行应用程序——这是带有额外步骤的类型检查。将其驱动到
用户会看到某些内容的点：

- CLI → 键入代表性命令，检查退出代码和输出。
- 服务器 → 使用 `curl` 访问差异触及的路由，读取正文。
- TUI → `send-keys` 导航，`capture-pane` 结果。
- GUI → 单击按钮，截取窗口屏幕。**查看屏幕截图。**
  空白帧是启动失败。

如果后备模式没有开箱即用——你必须
安装包、设置环境变量、修补配置或编写驱动程序——
在你的报告中推荐 `/run-skill-generator`，以便该工作被
捕获为项目技能。如果它只是有效，则不要。

---

# examples/cli.md — CLI 工具

CLI 是最简单的情况——通常没有后台进程要
管理，没有端口，没有生命周期。技能专注于**安装**、
**代表性调用**和**测试**。

## 重要的是什么

- **如何将二进制文件放在 `PATH` 上。** 全局安装？通过
  `npx`/`uv run` 运行？构建到 `./target/release/foo`？要明确。
- **两个或三个示例调用**，涵盖主要用例。
  包括预期输出，以便读者可以判断它是否有效。
- **退出代码**（如果它们有意义）（例如，linter 在发现时返回 1）。
- **Stdin 行为**（如果工具从 stdin 读取）。

## 示例片段

> ---
> name: run-mytool
> description: 构建、安装和运行 mytool。当被要求运行 mytool、测试它或验证它是否正确安装时使用。
> ---
>
> ## 设置
>
> ```bash
> pip install -e .
> ```
>
> 这将 `mytool` 放在 PATH 上。验证：
>
> ```bash
> mytool --version
> # → mytool 0.3.1
> ```
>
> ## 运行
>
> 处理单个文件：
>
> ```bash
> mytool process input.json
> # → 处理了 42 条记录，写入 output.json
> ```
>
> 从 stdin 读取，写入 stdout：
>
> ```bash
> cat input.json | mytool process -
> ```
>
> Lint 目录（有问题时退出非零）：
>
> ```bash
> mytool lint ./src
> echo $?  # 如果干净则为 0，如果发现问题则为 1
> ```
>
> ## 测试
>
> ```bash
> pytest
> ```

## 保持简短

CLI 的运行技能可以非常紧凑。不要用每个标志填充它——
`--help` 输出涵盖了这一点。只需显示足够的代理可以
(a) 构建它，(b) 确认它有效，(c) 运行测试。

---

# examples/server.md — Web 服务器 / API

服务器的区别关注点是**生命周期**：代理需要
在后台启动服务器，验证它已启动，与之交互，然后
干净地关闭它。阻塞 shell 的前台 `npm start` 对
代理无用。

## 要遵循的结构

一个好的服务器运行技能具有：

1. **先决条件和设置**——与任何项目相同。
2. **运行**——后台启动模式（如下），而不是阻塞命令。
3. **验证**——确认服务器实际已启动的 `curl` 或类似命令。
4. **停止**——如何干净地终止后台进程。

如果后台启动 + 就绪轮询 + 烟雾 curl 序列超过
几行，将其放在技能目录内的 `smoke.sh` 中，
并让 `SKILL.md` 说"运行烟雾脚本。"一个命令，退出代码
告诉你服务器是否健康。

## 后台启动模式

不要写：

> ```bash
> npm start
> ```

那会阻塞。相反，展示如何在后台启动，等待就绪，
并在以后找到 PID：

> ```bash
> npm start &> /tmp/server.log &
> SERVER_PID=$!
>
> # 等待服务器启动（根据需要调整超时/端口）
> for i in {1..30}; do
>   curl -sf http://localhost:3000/health > /dev/null && break
>   sleep 1
> done
> ```

然后验证步骤：

> ```bash
> curl http://localhost:3000/health
> # → {"status":"ok"}
> ```

以及停止：

> ```bash
> kill $SERVER_PID
> # 或者，如果你丢失了 PID：
> pkill -f "node.*server.js"
> ```

## 值得记录的细节

- **哪个端口。** 要明确，并说明如何覆盖它（`PORT=4000 npm start`）。
- **"就绪"是什么样子。** 特定的日志行或要命中的健康端点。
- **必需的环境变量。** 数据库 URL、API 密钥等——如果列表很长，则使用模板 `.env`。
- **热重载与生产模式。** 如果它们有显著差异，说明使用哪个以及何时使用。
- **依赖服务。** 如果服务器需要 Redis/Postgres 等，请
  指向启动它们的 docker-compose，或直接包含 `docker run`
  命令。

## 示例片段

以下是典型 Node API 的运行部分可能的样子：

> ## 运行
>
> 在后台启动开发服务器：
>
> ```bash
> npm run dev &> /tmp/api.log &
> ```
>
> 服务器在端口 3000 上监听。等待它就绪，然后验证：
>
> ```bash
> for i in {1..20}; do
>   curl -sf http://localhost:3000/health && break
>   sleep 0.5
> done
> curl http://localhost:3000/health
> # → {"status":"ok","version":"1.2.3"}
> ```
>
> 日志位于 `/tmp/api.log`。使用以下命令停止：
>
> ```bash
> pkill -f "tsx watch src/index.ts"
> ```
>
> ### 环境
>
> | 变量 | 必需 | 默认 | 注释 |
> |---|---|---|---|
> | `DATABASE_URL` | 是 | — | Postgres 连接字符串 |
> | `PORT` | 否 | `3000` | |
> | `LOG_LEVEL` | 否 | `info` | `debug` / `info` / `warn` / `error` |

---

# examples/tui.md — TUI / 交互式终端应用程序

交互式终端应用程序（文本编辑器、REPL、基于 curses 的 UI）不能
由代理的 bash 工具直接驱动——它们接管终端。
技能必须展示如何将它们包装在 `tmux` 中，以便代理可以发送
输入、捕获输出并截取屏幕截图。

## tmux 模式

这是标准方法：

1. 在分离的 tmux 会话中启动 TUI
2. 使用 `tmux send-keys` 发送击键
3. 使用 `tmux capture-pane` 读取屏幕内容
4. 使用 `tmux kill-session` 清理

技能的 `SKILL.md` 应将其呈现为驱动应用程序的主要方式。
一个包装启动+附加序列的小型 `driver.sh` 可以
存在于技能目录中，但对于大多数 TUI，技能正文中的原始 tmux 命令就足够了。

## 示例片段

> ## 运行（交互式，用于代理）
>
> 在 tmux 中启动 TUI：
>
> ```bash
> tmux new-session -d -s app -x 120 -y 40 './myapp'
> ```
>
> 轮询直到就绪标记出现（比固定睡眠更快 + 更可靠——
> 在应用程序启动的瞬间返回，如果没有则大声失败）：
>
> ```bash
> timeout 10 bash -c 'until tmux capture-pane -t app -p | grep -q "Ready"; do sleep 0.2; done'
> tmux capture-pane -t app -p
> ```
>
> 发送输入（此示例导航到设置屏幕并切换
> 一个选项）：
>
> ```bash
> tmux send-keys -t app 's'
> timeout 5 bash -c 'until tmux capture-pane -t app -p | grep -q "Settings"; do sleep 0.2; done'
> tmux send-keys -t app 'Down' 'Down' 'Space'  # 导航 + 切换
> timeout 5 bash -c 'until tmux capture-pane -t app -p | grep -qF "[x]"; do sleep 0.2; done'
> tmux capture-pane -t app -p
> ```
>
> 如果你发现自己编写了不止几个这样的轮询行，请将它们
> 拉到技能旁边的 `driver.sh` 中的 `wait_for()` 帮助器中。
>
> 退出：
>
> ```bash
> tmux send-keys -t app 'q'
> tmux kill-session -t app 2>/dev/null || true
> ```
>
> ### 键引用
>
> | 键 | 操作 |
> |---|---|
> | `j` / `k` 或 `Down` / `Up` | 导航列表 |
> | `Enter` | 选择 |
> | `s` | 设置 |
> | `q` | 退出 |

## 值得记录的细节

- **终端大小。** 某些 TUI 在小宽度下会中断或隐藏内容。
  在 `tmux new-session -x -y` 参数中指定已知良好的大小。
- **启动时间。** 轮询就绪标记（`until tmux capture-pane | grep -q X`）
  而不是固定的 `sleep N`——在应用程序启动的瞬间返回，并且在
  永不启动时有用地失败。说明什么字符串意味着就绪。
- **键绑定引用。** 主要键的表格。这是 TUI 的"API"
  ——代理需要它来驱动应用程序。
- **干净退出。** 显示退出击键*和* `tmux kill-session` 作为
  后备。
- **颜色/unicode 怪癖。** 如果 `capture-pane` 输出难以阅读，
  请注意有帮助的标志（`-e` 用于转义序列，`-J` 连接换行
  行）。

## 还要记录直接调用

对于交互式运行应用程序的人类，tmux 是过度杀伤。也包括
单行命令：

> ## 运行（直接，用于人类）
>
> ```bash
> ./myapp
> ```
>
> 按 `q` 退出。

---

# examples/electron.md — Electron / 桌面 GUI 应用程序

Electron 应用程序有一个窗口。无头容器中的未来代理
无法看到窗口。因此，你在这里交付的不是 markdown 文件
说"`npm start` 打开一个窗口"——它是一个**驱动程序脚本**，在
xvfb 下启动应用程序，公开命令 REPL（单击、输入、
屏幕截图），并允许代理通过发送文本行来戳 UI。

然后，技能的 `SKILL.md` 成为该驱动程序的简短手册。

## 你正在构建什么

```
apps/desktop/
  .claude/skills/run-desktop/
    SKILL.md               ← 简短。"运行驱动程序，这里是命令"
    driver.mjs             ← REPL：stdin 命令 → Playwright 操作
```

驱动程序就是产品。没有它，技能描述了一个代理
永远无法触摸的 GUI。

**毕业路径：**如果驱动程序增长项目真实 e2e 套件想要共享的启动帮助器，请将其移动到 `e2e-playwright/driver.mjs`
（或 `scripts/drive.mjs`）并更新技能的路径。技能保留
在 `.claude/skills/run-desktop/`；驱动程序找到更好的家。

## 步骤 1 — 让应用程序在 xvfb 下完全启动

这通常是最难的部分，并产生最多的陷阱。README
会说"仅限 macOS/Windows"。忽略它。安装 xvfb + Chromium
共享库，找到 Electron 二进制文件，并启动它：

```bash
apt-get install -y xvfb libnss3 libgbm1 libasound2t64 libgtk-3-0 \
  libxss1 libxkbcommon0 libatk-bridge2.0-0 libcups2 libdrm2

# 首先构建应用程序。通常"dev"脚本是 electron-forge，它
# 执行 Vite/webpack 构建，然后启动。你只需要构建：
npm install
npx electron-forge start &   # 构建 .vite/build/ 或 dist/
sleep 20 && kill %1          # 构建后杀死它——你将自己启动

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

迭代直到它启动。每个缺失的 `.so` → 再一个 `apt-get`
包 → 先决条件中再一行。每个启动超时 → 检查
`nodeCliInspect` 保险丝未禁用，检查构建输出存在。

**`--no-sandbox` 在容器中几乎总是需要的。** Electron 的
沙箱需要 CAP_SYS_ADMIN 或用户命名空间。默认情况下两者都没有。

## 步骤 2 — 构建 REPL 驱动程序

一旦你可以启动它，将那个一次性脚本转换为 REPL。开始
最小——你将根据需要添加命令。**REPL 是正确的形状**，
因为代理可以在 tmux 中运行它并在每次交互时迭代
而无需重新启动（慢速）应用程序。

```javascript
// .claude/skills/run-<unit>/driver.mjs
// <app> 的 REPL 驱动程序。在无头 Linux 上的 xvfb 下运行。
// 为代理设计：包装在 tmux 中，send-keys 命令，capture-pane 输出。
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
    await new Promise(r => setTimeout(r, 8_000));
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

**这是一个起始骨架。** 当你尝试到达应用程序的有趣部分时，
你将添加特定于应用程序的命令。

## 步骤 3 — 通过 tmux 自己使用它

像下一个代理一样运行驱动程序：

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

然后实际打开 `/tmp/shots/01-landing.png`。它是应用程序吗？它是
空白吗？它是登录屏幕吗？每一个都告诉你下一步做什么。

## 步骤 4 — 编写 SKILL.md

保持简短。驱动程序是主要内容；`SKILL.md` 是手册。

## 你将遇到的障碍（它们进入陷阱）

- **`firstWindow()` 给你一个闪屏/加载屏幕，**而不是应用程序。
- **真正的 UI 在 BrowserView 中，而不是 BrowserWindow 中。**
- **`locator.click()` 单击了错误的东西。** 使用 `page.evaluate(el => el.click())`。
- **功能门阻止你需要测试的东西。**
- **contentEditable 输入**（ProseMirror、Tiptap、Slate）不是 `<textarea>`。
- **Electron 窃取 stdin。** `fs.openSync('/dev/stdin', 'r')` 技巧保护你的 REPL 输入。
- **本机模块无法加载**（钥匙串、通知等）。

---

# examples/playwright.md — 浏览器驱动的 Web 应用程序

你有一个向浏览器提供 HTML 的开发服务器。无头容器中的代理
无法打开浏览器窗口——因此"运行应用程序"意味着
启动开发服务器，针对它驱动无头 Chromium，并
生成证明页面已渲染的屏幕截图。

不要编写浏览器驱动程序。使用 `chromium-cli`。

## 开发服务器

找到开发命令（`package.json` `scripts.dev`、`Makefile`、
README），在后台启动它，并等待它实际服务：

```bash
npm run dev &   # 或 yarn dev、pnpm dev、make serve、./dev.sh
echo $! > /tmp/dev.pid
timeout 30 bash -c 'until curl -sf http://localhost:3000 >/dev/null; do sleep 1; done'
```

## 驱动

`chromium-cli` 是无头 Chromium REPL。将脚本通过管道传输到 stdin：

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

## 要在技能中放入什么

仅特定于项目的位。`chromium-cli` 处理机制。

- **开发命令 + 端口 + 停止。**
- **身份验证。**
- **一个代表性交互。**
- **特定于应用程序的陷阱。**

## 经常出现的陷阱

- **React 受控输入。** 使用 `fill` / `type`，而不是 `eval el.value = '…'`。
- **Websockets / 长轮询。** `wait-idle` 永不解决。`wait-for` 你实际需要的元素。
- **慢速首次绘制。** Vite/Next 按需编译路由；第一个 `nav` 可能需要 10 秒以上。
- **`screenshot-element <sel>`** 裁剪到一个元素。
- **在声明成功之前检查 `console --errors`。**

---

# examples/library.md — 库 / SDK

库在流程意义上没有"运行"步骤。对于库，运行技能是关于：

1. **从源代码构建**库
2. **运行测试套件**
3. **练习库的最小工作示例**

## 烟雾测试示例

> ```bash
> python -c '
> from mylib import Client
> c = Client()
> print(c.ping())
> '
> # → pong
> ```

## 要考虑记录的事情

- **开发模式与已安装模式。**
- **可选依赖项。**
- **生成的代码。**