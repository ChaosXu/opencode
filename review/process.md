# OpenCode 项目进程清单

本文档梳理 OpenCode 完整构建后真正落地为 OS 进程的入口、其启动方式、生命周期和父子关系。

不含 dev server（Astro/Vite/Storybook/jsx-email）、Cloudflare Workers、单纯的 shell 工具调用（bash 工具、git 快照等）。

---

## 1. 进程总览

完整构建后，独立可执行的服务/二进制共 **6 类**，按"独立 OS 进程单元"算：

| # | 名称 | 进程入口 | 命令/启动方 | 生命周期 |
| - | ---- | -------- | ----------- | -------- |
| 1 | OpenCode CLI | `packages/opencode/src/index.ts` | `opencode ...` | 6 种长驻模式 |
| 2 | lildax CLI | `packages/cli/src/commands/handlers/*.ts` | `lildax ...` | 2 种长驻模式 + 1 个真正 detached 守护 |
| 3 | Desktop Main | `packages/desktop/src/main/index.ts` | 打包后由 OS / `electron-vite dev` | 长驻 |
| 4 | Desktop Utility Sidecar | `packages/desktop/src/main/sidecar.ts` | Main 用 `utilityProcess.fork` | 长驻 HTTP（child of Main） |
| 5 | Desktop WSL Sidecar | `packages/desktop/src/main/wsl/sidecar.ts` | Main 用 `spawn("wsl", ...)` | 长驻（仅 Windows；child of Main） |
| 6 | Slack Bot | `packages/slack/src/index.ts` | `bun run src/index.ts` | 长驻 socket-mode |
| 7 | Stats Ingest Server | `packages/stats/server/src/server.ts` | `bun src/server.ts` | 长驻 HTTP（`PORT=3000`） |
| 8 | Stats Sync Daemon | `packages/stats/server/src/stat-sync.ts` | `bun src/stat-sync.ts` | 长驻周期任务 |

外加 2 个 npm bin 启动壳（`packages/opencode/bin/opencode`、`packages/cli/bin/lildax.cjs`），它们 spawn 出真实二进制后透传 stdio，自身进程很短命。

---

## 2. OpenCode CLI（`packages/opencode`）

### 2.1 包概述

- **类型**: Bun 编译的单一二进制
- **npm bin 入口**: `packages/opencode/bin/opencode`（`packages/opencode/package.json:13` 注册 `"opencode": "./bin/opencode"`），行为是 spawn 真实二进制并透传 stdio。
- **真实二进制入口**: `packages/opencode/src/index.ts`（子命令路由表位于 `packages/opencode/src/index.ts:81-103`）。
- **构建脚本**: `packages/opencode/script/build.ts`、`packages/opencode/script/build-node.ts`（Node 端 bundle，给 desktop sidecar 用）。

### 2.2 长驻模式（一个二进制，多种运行形态）

| 模式 | 入口文件 | 命令 | 生命周期 / 关键调用 |
| ---- | -------- | ---- | ------------------- |
| TUI（默认） | `packages/opencode/src/cli/cmd/tui.ts:71` | `opencode [project]` | 长驻 TUI；worker 线程跑 `Server.listen`（`packages/opencode/src/cli/tui/worker.ts:49`） |
| `tui` 显式 | 同上 | `opencode tui ...` | 同上 |
| `serve` | `packages/opencode/src/cli/cmd/serve.ts:6` | `opencode serve ...` | 长驻 HTTP；handler 末尾 `yield* Effect.never`（`packages/opencode/src/cli/cmd/serve.ts:22`）；底层 `Server.listen`（`packages/opencode/src/server/server.ts:72`） |
| `web` | `packages/opencode/src/cli/cmd/web.ts:31` | `opencode web ...` | 长驻 HTTP（`Effect.never`，`packages/opencode/src/cli/cmd/web.ts:82`） + 短暂 fork `open` 拉起浏览器（`packages/opencode/src/cli/cmd/web.ts:75, 79`） |
| `acp` | `packages/opencode/src/cli/cmd/acp.ts:9` | `opencode acp ...` | 长驻 NDJSON-over-stdio（`packages/opencode/src/cli/cmd/acp.ts:65-71`） |
| `attach` | `packages/opencode/src/cli/cmd/attach.ts:7` | `opencode attach <url>` | 长驻 TUI，连远端 HTTP |
| `run -i` | `packages/opencode/src/cli/cmd/run.ts:122` | `opencode run -i ...` | 长驻 TUI（默认一次性；`runInteractiveMode` / `runInteractiveLocalMode`，`packages/opencode/src/cli/cmd/run.ts:811, 839`） |

### 2.3 一次性命令（短命进程）

`generate`、`stats`、`models`、`providers`、`agent`、`upgrade`、`uninstall`、`export`、`import`、`pr`、`session`、`db`、`account`、`console`、`debug`、`github install/run`、`mcp add/list/auth/logout/debug`、`plug` 等，子命令处理文件位于 `packages/opencode/src/cli/cmd/<name>.ts`，执行后退出。

`mcp` 不自身运行 MCP server：用户配置的 MCP server 是 server 运行后由 `packages/opencode/src/mcp/index.ts:839-846` 派生的子进程。

---

## 3. lildax CLI（`packages/cli`）

### 3.1 包概述

- **类型**: 第二个独立的 Bun 编译二进制（v2 CLI）
- **npm bin 入口**: `packages/cli/bin/lildax.cjs`（`packages/cli/package.json:9` 注册 `"lildax": "./bin/lildax.cjs"`），是 re-exec 启动壳。
- **真实二进制入口**: `packages/cli/src/index.ts`（默认 handler 在 `packages/cli/src/index.ts:26`）。

### 3.2 长驻模式

| 模式 | 入口文件 | 命令 | 生命周期 / 关键调用 |
| ---- | -------- | ---- | ------------------- |
| TUI（默认） | `packages/cli/src/commands/handlers/default.ts:6` | `lildax` | 长驻 TUI；TUI runtime 在 `packages/cli/src/tui.ts:6` |
| `serve` | `packages/cli/src/commands/handlers/serve.ts:13` | `lildax serve ...` | 长驻 HTTP（`return yield* Effect.never`，`packages/cli/src/commands/handlers/serve.ts:22`） |
| `serve --register`（**detached**） | 由 `packages/cli/src/services/daemon.ts:122` 派生 | `lildax serve --register` | **唯一真正 detached 的后台守护进程** |

### 3.3 detached 守护（重点）

`lildax service start` / `restart` 触发 `Daemon.Service.start`，spawn 出一个独立的 `lildax serve --register` 进程（`packages/cli/src/services/daemon.ts:122-125`）：

```ts
spawn(process.execPath, [...(entrypoint ? [entrypoint] : []), "serve", "--register"], {
  detached: true,
  stdio: "ignore",
}).unref()
```

- 父进程退出后子进程继续运行（`detached: true` + `unref()`）
- 注册表写 `~/.local/state/opencode/server.json`（`packages/cli/src/services/daemon.ts:40`）
- 健康检查 + 复用逻辑在 `packages/cli/src/services/daemon.ts:66-72`
- 停止用 `lildax service stop`（`packages/cli/src/services/daemon.ts:91-108`：SIGTERM → 50ms 重试 100 次 → SIGKILL）

### 3.4 一次性命令

`migrate`（`packages/cli/src/commands/handlers/migrate.ts`）、`debug agents`（`packages/cli/src/commands/handlers/debug/agents.ts`）。

---

## 4. Desktop（`packages/desktop`）

### 4.1 包概述

- **类型**: Electron 应用（`electron-vite` 三 bundle：main / preload / renderer）
- **package.json#main**: `./out/main/index.js`（`packages/desktop/package.json:31`）
- **真实入口**: `packages/desktop/src/main/index.ts:367`（`Effect.runFork(main)`）
- **生命周期**: 唯一真正根进程，UI、auto-updater、sidecar 都由它编排。

### 4.2 进程清单

| 进程 | 入口文件 | 启动方 | 生命周期 / 备注 |
| ---- | -------- | ------ | --------------- |
| Main | `packages/desktop/src/main/index.ts` | 打包后由 OS 启动 / `electron-vite dev` | 长驻，监听 SIGINT/SIGTERM（`packages/desktop/src/main/index.ts:229-233`），`before-quit`/`will-quit` 杀 sidecar（`packages/desktop/src/main/index.ts:209-215`） |
| Utility Sidecar | `packages/desktop/src/main/sidecar.ts:51` | Main 用 `utilityProcess.fork(sidecar, [], { serviceName: "opencode server", stdio: "pipe" })`（`packages/desktop/src/main/server.ts:62`） | 长驻 HTTP；动态 `import("virtual:opencode-server")` 解析到 `packages/opencode/dist/node/node.js`；sidecar 内部 `Server.listen`（`packages/desktop/src/main/sidecar.ts:59`） |
| WSL Sidecar（仅 Windows） | `packages/desktop/src/main/wsl/sidecar.ts:16` | Main 用 `spawn("wsl", ...)`，stdin 喂 bash 脚本，结尾 `exec opencode serve --hostname 0.0.0.0 --port N`（`packages/desktop/src/main/wsl/sidecar.ts:26-37`） | 长驻；外部 `wsl.exe` 进程，WSL 内部又是一个 `opencode` 进程；停止在 `packages/desktop/src/main/wsl/servers.ts:391-401` 的 `wslServers.stopAll()` |

### 4.3 不算独立 OS 进程的内部单元

- **Preload**: Electron preload bundle（`packages/desktop/electron.vite.config.ts:71-81`），运行在 main 进程上下文，**不是**独立进程。
- **Renderer**: 跑在 `BrowserWindow` 里的 SolidJS bundle，Electron 内部子进程。
- **WSL 内联工具子进程**（`packages/desktop/src/main/wsl/runtime.ts:322-335` 等）：`cmd.exe start wsl -d <distro>` detached+unrefed 拉起 WSL 终端，但本身是一次性 launcher，不算服务。

---

## 5. Slack Bot（`packages/slack`）

- **入口**: `packages/slack/src/index.ts:1`
- **启动**: `await app.start()`（`packages/slack/src/index.ts:144`）
- **运行模式**: `@slack/bolt` socket-mode 长驻监听；每条 thread 内部用 `createOpencode({ port: 0 })` 派生一个进程内 HTTP server（不额外起 OS 进程）
- **dev 命令**: `bun run src/index.ts`（`packages/slack/package.json:9`）
- **生产**: 由部署方托管为长驻进程

---

## 6. Stats 服务（`packages/stats/server`）

### 6.1 Ingest HTTP

- **入口**: `packages/stats/server/src/server.ts:1`
- **启动**: `NodeRuntime.runMain(main, ...)`（`packages/stats/server/src/server.ts:28`）；底层 `Layer.launch(HttpRouter.serve(...))`（`packages/stats/server/src/server.ts:22`）
- **命令**: `bun src/server.ts`（`packages/stats/server/package.json:13` `"start"`）
- **Docker**: `CMD ["bun", "src/server.ts"]`（`packages/stats/server/Dockerfile:24`）
- **端口**: `PORT` env，默认 3000（`packages/stats/server/src/server.ts:15`）

### 6.2 Sync Daemon

- **入口**: `packages/stats/server/src/stat-sync.ts:20`
- **启动**: `NodeRuntime.runMain`
- **命令**: `bun src/stat-sync.ts`
- **调度**: `Effect.repeat(Schedule.fixed("1 hour"))`（`packages/stats/server/src/stat-sync.ts:16`）

---

## 7. 父子进程关系

```
OpenCode CLI binary ── (worker thread, 不算进程) ── TUI in-process HTTP server

lildax CLI binary ── spawn(detached, unref) ── lildax serve --register (长驻)

Desktop Main (Electron) ── utilityProcess.fork ── Desktop Utility Sidecar (opencode Server.listen)
                              └─ (Windows only) spawn("wsl") ── WSL Sidecar (wsl.exe → opencode serve)

Slack Bot ── in-process ── createOpencode server (port 0)

Stats Ingest Server
Stats Sync Daemon  (独立进程，独立调度)
```

整个仓库**唯一真正 detached 的 OS 进程**是 `lildax serve --register`（`packages/cli/src/services/daemon.ts:123`）。其余 long-lived 进程的生命周期都跟随其父进程。

---

## 8. 关键引用

- OpenCode CLI 子命令路由: `packages/opencode/src/index.ts:81-103`
- `Server.listen`（TUI worker / serve / web / acp / desktop sidecar 共享的 HTTP 服务）: `packages/opencode/src/server/server.ts:72-97`
- lildax detached spawn: `packages/cli/src/services/daemon.ts:122-125`
- lildax 服务管理: `packages/cli/src/services/daemon.ts:91-108`
- Desktop main 启动: `packages/desktop/src/main/index.ts:367`；`package.json:31` `"main": "./out/main/index.js"`
- Desktop sidecar fork: `packages/desktop/src/main/server.ts:62`；sidecar 脚本入口: `packages/desktop/src/main/sidecar.ts:51`
- WSL sidecar: `packages/desktop/src/main/wsl/sidecar.ts:39`
- Slack bot 启动: `packages/slack/src/index.ts:144`
- Stats server: `packages/stats/server/src/server.ts:28`
- Stats sync: `packages/stats/server/src/stat-sync.ts:20`
