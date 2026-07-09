> **说明**：本文件为英文原文（`server.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 示例：Web 服务器 / API

服务器的区别关注点是**生命周期**：智能体需要在后台启动服务器，验证它已启动，与之交互，然后干净地关闭它。阻塞 shell 的前台 `npm start` 对智能体无用。

## 要遵循的结构

一个好的服务器 run skill 有：

1. **先决条件和设置** — 与任何项目相同。
2. **运行** — 后台启动模式（下面），而不是阻塞命令。
3. **验证** — `curl` 或类似的，确认服务器实际已启动。
4. **停止** — 如何干净地终止后台进程。

如果后台启动 + 就绪轮询 + 冒烟 curl 序列超过几行，将其放在 skill 目录内的 `smoke.sh` 中，并让 `SKILL.md` 说"运行冒烟脚本"。一个命令，退出代码告诉你服务器是否健康。

## 后台启动模式

不要写：

> ```bash
> npm start
> ```

那会阻塞。相反，展示如何在后台启动、等待就绪，然后稍后找到 PID：

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

## 值得文档化的细节

- **哪个端口。**明确说明并说明如何覆盖它（`PORT=4000 npm start`）。
- **"就绪"看起来像什么。**特定的日志行或要命中的健康端点。
- **必需的环境变量。**数据库 URL、API 密钥等 — 如果列表很长，提供模板 `.env`。
- **热重载与生产模式。**如果它们有意义地不同，说明何时使用哪个。
- **依赖服务。**如果服务器需要 Redis/Postgres/等，要么指向 docker-compose 将它们启动，要么直接包含 `docker run` 命令。

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
> 日志位于 `/tmp/api.log`。用以下命令停止：
>
> ```bash
> pkill -f "tsx watch src/index.ts"
> ```
>
> ### 环境
>
> | 变量 | 必需 | 默认 | 说明 |
> |---|---|---|---|
> | `DATABASE_URL` | 是 | — | Postgres 连接字符串 |
> | `PORT` | 否 | `3000` | |
> | `LOG_LEVEL` | 否 | `info` | `debug` / `info` / `warn` / `error` |
