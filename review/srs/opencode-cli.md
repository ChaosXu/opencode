# OpenCode CLI 功能规格

> 入口：`packages/opencode/src/index.ts`
> 子命令路由表：`packages/opencode/src/index.ts:81-103`
> 全部命令实现位于：`packages/opencode/src/cli/cmd/`

本文档梳理 `opencode` CLI 完整构建后对外暴露的全部子命令、全局选项和设计要点。

---

## 1. 概述

- **形态**: Bun 编译的单一二进制（npm bin 启动壳 `packages/opencode/bin/opencode` 透传 stdio）
- **CLI 框架**: yargs（严格模式 `strict()`，命令注册顺序决定 help 顺序）
- **命令条数**: 22 个子命令（含 1 个默认命令 `$0`）
- **错误处理**: 集中 `catch → FormatError → UI.error → process.exitCode = 1`
- **强退出**: `finally { process.exit() }` 避免 docker MCP 等子进程挂住（`src/index.ts:136-141`）

---

## 2. 全局选项（`src/index.ts:53-78`）

| 选项 | 说明 | 环境变量映射 |
| ---- | ---- | ------------ |
| `-h, --help` | 展示帮助 | — |
| `-v, --version` | 展示版本 | `InstallationVersion` |
| `--print-logs` | 把日志打印到 stderr | `OPENCODE_PRINT_LOGS=1` |
| `--log-level` | 日志级别 | `OPENCODE_LOG_LEVEL` ∈ {`DEBUG`,`INFO`,`WARN`,`ERROR`} |
| `--pure` | 不加载外部插件 | `OPENCODE_PURE=1` |
| `completion` | 生成 shell 补全脚本 | — |
| `--` | yargs `populate--` 透传剩余参数 | — |

**middleware 注入**：`AGENT=1`、`OPENCODE=1`、`OPENCODE_PID=<pid>`、调用 `Heap.start()`（V8 heap 监控）

---

## 3. 子命令

### 3.1 AI 交互与界面

#### 3.1.1 `run [message..]`（`src/cli/cmd/run.ts:122`）

三种模式：

1. **非交互（默认）** — 跑进程内 server，发一条 prompt，事件流式输出到 stdout，会话 idle 后退出
2. **本地交互 `--interactive` / `-i`** — 启 split-footer direct mode（`runInteractiveLocalMode`，`run.ts:839`），不依赖外部 HTTP
3. **远端交互 `--interactive --attach <url>`** — 接到已运行的 opencode server（`run.ts:811`）

主要选项：

| 选项 | 含义 |
| ---- | ---- |
| `--command` | 跑 slash-command，message 作 args |
| `--format json` | 原始事件流（不开 TUI） |
| `--continue, -c` | 续接最近 session |
| `--session, -s <id>` | 续接指定 session |
| `--fork` | 续接前先 fork |
| `--attach <url>` | 接到远端 server |
| `-i, --interactive` | 进 TUI |
| `-m, --model provider/model` | 指定模型 |
| `--dir <dir>` | 项目目录 |
| `--file <path>` | 附加 file part |

#### 3.1.2 `tui [project]`（默认命令 `$0`）（`src/cli/cmd/tui.ts:71`）

- 启动 TUI；worker 线程里跑 `Server.listen`（`src/cli/tui/worker.ts:49`）
- `[project]` 是项目目录（绝对或相对）
- 网络选项来自 `src/cli/network.ts`（`withNetworkOptions`，`withNetworkOptionsNoConfig`）
- 支持 `--model`、`--port`、`--hostname`、`--mdns`

#### 3.1.3 `attach <url>`（`src/cli/cmd/attach.ts:7`）

TUI 客户端连到已运行的 `serve`/`web`/`acp` server。`--dir`、`--continue`/`--session ID`、`--fork`、鉴权头（`Authorization: Basic` 用 `opencode:<password>`）由 attach 命令处理。

#### 3.1.4 `acp`（`src/cli/cmd/acp.ts:9`）

Agent Client Protocol server：NDJSON over stdio，进程内起一个私有端口 HTTP。`--cwd`、`--port`、`--hostname`。读到 stdin EOF 退出（`acp.ts:65-71`）。

#### 3.1.5 `web`（`src/cli/cmd/web.ts:31`）

起 HTTP server（同 `serve` 的 `Server.listen`）+ `open(url)` 拉起浏览器。handler 末尾 `yield* Effect.never`（`web.ts:82`）。

### 3.2 服务

#### 3.2.1 `serve`（`src/cli/cmd/serve.ts:6`）

无头 HTTP server（`Effect.never`，`serve.ts:22`），底层 `Server.listen`（`src/server/server.ts:72`）。被 TUI worker / `web` / `acp` / desktop sidecar 复用。

- `--port`
- `--hostname`
- `--mdns`
- `--cors <origin>`

### 3.3 认证与账户

#### 3.3.1 `account` / `console`（`src/cli/cmd/account.ts`）

OpenCode Console 登录（`https://console.opencode.ai`，`account.ts:18`），走设备码流（`Account.Service.login`）。支持：

- 登录（开浏览器，输 user code）
- 列表 / 切换账户
- 列表 / 切换 organization
- 登出

#### 3.3.2 `providers`（`src/cli/cmd/providers.ts`）

- 列出所有 provider（含插件注册的 `hooks.auth`）
- 登录某 provider（交互式选择 method 或 `--method` 指定）
- 登出
- 走 `Auth.Service` 把凭据持久化到 keychain / 配置文件

### 3.4 会话管理

#### 3.4.1 `session`（`src/cli/cmd/session.ts:44`）

无 handler，强制 `demandCommand`：

| 子命令 | 文件 | 说明 |
| ------ | ---- | ---- |
| `session list` | `session.ts:70` | 列 session；`--max-count, -n` 限定数量；`--format` 控制输出；非 Windows 走 `less -R -S`，Windows fallback 到 `more`（`pagerCmd`） |
| `session delete <sessionID>` | `session.ts:51` | 删除指定 session（`Session.Service.remove`，捕获 `NotFoundError`） |

### 3.5 分享导入导出

#### 3.5.1 `export`（`src/cli/cmd/export.ts:6`）

把 session 脱敏导出为本地 JSON：redact file/url/path/symbol/text 等（`export.ts:11-67`），保留结构可供分享。输出含 `info`、`messages[]`、`parts[]` 三段嵌套。

#### 3.5.2 `import <url>`（`src/cli/cmd/import.ts:6`）

从 `https://opncd.ai/share/<id>` 拉分享数据（`ShareNext` API），把扁平 `[session, message, message, part, part, ...]` 重建为 `{ info, messages: [{ info, parts: [] }] }` 后写入本地库。`parseShareUrl` 提取 share ID，`shouldAttachShareAuthHeaders` 判断是否需鉴权头。

### 3.6 MCP 管理（`src/cli/cmd/mcp.ts`）

总入口 `mcp`（`mcp.ts:849`），多子命令：

| 子命令 | 能力 |
| ------ | ---- |
| `mcp list` | 树状展示 `local`/`remote` MCP server 状态（`getAuthStatusIcon` 显示 ✓/⚠/✗）、`hasStoredTokens` 检查 OAuth token |
| `mcp add` | 交互式写入 `opencode.json`：走 `jsonc-parser.modify + applyEdits` 保留注释 |
| `mcp auth` | OAuth 流，调 `McpOAuthProvider` 触发授权；处理 `UnauthorizedError` |
| `mcp logout` | 清除 token |
| `mcp debug` | 启 `StreamableHTTPClientTransport` 直连一个 server 调试，遵守 `LATEST_PROTOCOL_VERSION` |

`mcp` 子命令本身**不启动** MCP server — 实际 server 进程由运行中的 `serve`/`tui` 在用户配置存在时按需派生（`src/mcp/index.ts:839-846`）。

### 3.7 信息查看

#### 3.7.1 `models`（`src/cli/cmd/models.ts`）

列出当前 provider 集合的全部可用模型，含 id、name、context window、tools 支持等。

#### 3.7.2 `agent`（`src/cli/cmd/agent.ts`）

列出 / 查看内置（`build` / `plan` / `general`）与项目自定义 agent。

#### 3.7.3 `stats`（`src/cli/cmd/stats.ts:49`）

聚合本地 session 库生成统计报告：

| 字段 | 说明 |
| ---- | ---- |
| `totalSessions` / `totalMessages` / `totalCost` | 累计 |
| `totalTokens.{input,output,reasoning,cache.{read,write}}` | token 分布 |
| `toolUsage` | 工具调用计数 |
| `modelUsage` | 按模型分组的 message/tokens/cost |
| `dateRange` / `days` / `costPerDay` / `tokensPerSession` / `medianTokensPerSession` | 派生指标 |

选项：`--days`（最近 N 天）、`--tools`（top N）、`--models`（`true`/数字/all）、`--project`（按项目过滤，空串表示当前项目）。

### 3.8 集成与工具

#### 3.8.1 `github`（`src/cli/cmd/github.ts:7, 17`）

| 子命令 | 能力 |
| ------ | ---- |
| `github install` | 安装 GitHub 集成（事件 webhook/Action 入口） |
| `github run` | 跑 GitHub 集成任务 |

#### 3.8.2 `pr <number>`（`src/cli/cmd/pr.ts:8`）

`gh pr checkout` + `gh pr view --json headRepository,headRepositoryOwner,isCrossRepository,headRefName,body`。识别 cross-repo fork → 加 fork remote → fetch → checkout `pr/<N>` → 启动 opencode 处理 PR。要求 `gh` CLI 已安装认证。

#### 3.8.3 `plug`（注册名为 `plugin`，`src/cli/cmd/plug.ts:6`）

插件管理：

- `installPlugin(spec, ...)` 支持 npm / git / 本地路径
- `resolvePluginTarget` 决定安装到 `~/.config/opencode/plugin/`（global）还是项目内
- 可选 `--force` 覆盖
- `readPluginManifest` 校验 manifest

### 3.9 调试 / 维护

#### 3.9.1 `generate`（`src/cli/cmd/generate.ts:6`）

一次性生成 JSON spec（OpenAPI/Schema 类），供 SDK/前端消费。

#### 3.9.2 `debug`（`src/cli/cmd/debug/*`）

14 个子目录，调试套件（运行时、session、provider、event、permission 等）。

#### 3.9.3 `db`（`src/cli/cmd/db.ts:7`）

直连本地 SQLite（`packages/core/src/session/sql.ts`），用于维护/排查。

#### 3.9.4 `upgrade`（`src/cli/cmd/upgrade.ts:7`）

自升级二进制，逻辑在 `src/cli/upgrade.ts`。

#### 3.9.5 `uninstall`（`src/cli/cmd/uninstall.ts:25`）

- `Installation.method()` 识别安装方式（curl/brew/scoop/...）
- 可选 `--keep-config`/`--keep-data`/`--dry-run`/`--force`
- 删除二进制、shell PATH 配置、`~/.config/opencode/`、`~/.local/share/opencode/` 等

---

## 4. 命令总览

| # | 命令 | 入口 | 主要职责 |
| - | ---- | ---- | -------- |
| 1 | `run` (默认之一) | `cmd/run.ts:122` | 跑 AI 对话（3 种模式） |
| 2 | `tui` (默认 `$0`) | `cmd/tui.ts:71` | 启 TUI |
| 3 | `attach` | `cmd/attach.ts:7` | TUI 连远端 |
| 4 | `acp` | `cmd/acp.ts:9` | ACP 协议 |
| 5 | `web` | `cmd/web.ts:31` | HTTP + 浏览器 |
| 6 | `serve` | `cmd/serve.ts:6` | 无头 HTTP |
| 7 | `account` / `console` | `cmd/account.ts` | Console 登录 |
| 8 | `providers` | `cmd/providers.ts` | Provider 登录/列表 |
| 9 | `session list\|delete` | `cmd/session.ts` | 会话管理 |
| 10 | `export` | `cmd/export.ts` | 脱敏导出 |
| 11 | `import` | `cmd/import.ts` | 导入分享 |
| 12 | `mcp` | `cmd/mcp.ts` | MCP 管理 |
| 13 | `models` | `cmd/models.ts` | 模型列表 |
| 14 | `agent` | `cmd/agent.ts` | Agent 列表 |
| 15 | `stats` | `cmd/stats.ts` | 使用统计 |
| 16 | `github` | `cmd/github.ts` | GitHub 集成 |
| 17 | `pr` | `cmd/pr.ts` | PR checkout + 处理 |
| 18 | `plug` (`plugin`) | `cmd/plug.ts` | 插件管理 |
| 19 | `generate` | `cmd/generate.ts` | 生成 spec |
| 20 | `debug` | `cmd/debug/*` | 调试套件 |
| 21 | `db` | `cmd/db.ts` | DB 操作 |
| 22 | `upgrade` | `cmd/upgrade.ts` | 自升级 |
| 23 | `uninstall` | `cmd/uninstall.ts` | 卸载 |

> 上述 23 行 = 22 个子命令 + 1 个 `run` 重复列示（`run` 与默认 `$0` 是两个独立注册）。

---

## 5. 设计要点

### 5.1 命令实现的两条线

- **普通 yargs cmd**（`cmd(...)`，`src/cli/cmd/cmd.ts`）— 老风格
- **Effect cmd**（`effectCmd(...)`，`src/cli/effect-cmd.ts`）— 新风格，handler 走 `Effect.fn("Domain.method")`，自带 trace、`fail()` 抛 `CliError`

### 5.2 Instance / Directory 解析

`run` 命令通过 `instance: (args) => !args.attach` 和 `directory: (args) => (args.dir && !args.attach ? ...)` 控制是否要起本地 project instance；`--attach` 模式只做 HTTP 客户端。

### 5.3 共享运行时

- HTTP server：`Server.listen`（`src/server/server.ts:72`）是 TUI worker、`serve`、`web`、`acp`、desktop sidecar 的共同底层
- UI 输出：`UI` 工具（`src/cli/ui.ts`）+ `clack/prompts`（交互式）
- 网络选项：`withNetworkOptions`（`src/cli/network.ts`）

### 5.4 子进程策略

- 短命一次性子命令（`generate`/`stats`/`models`/`upgrade`/`uninstall`/`db`/`debug` 等）：进程退出前 `process.exit()` 兜底，避免 docker MCP 等挂住
- 长生命子进程（`serve`/`web`/`acp`/`tui`）：`Effect.never` 维持运行

---

## 6. 关键引用

- CLI 入口与路由: `packages/opencode/src/index.ts:81-103`
- yargs 配置: `packages/opencode/src/index.ts:45-78`
- 错误处理: `packages/opencode/src/cli/error.ts`
- 强退出: `packages/opencode/src/index.ts:136-141`
- Effect 命令封装: `packages/opencode/src/cli/effect-cmd.ts`
- 共享 HTTP server: `packages/opencode/src/server/server.ts:72`
- TUI worker: `packages/opencode/src/cli/tui/worker.ts:49`
- 网络选项: `packages/opencode/src/cli/network.ts`
- 升级逻辑: `packages/opencode/src/cli/upgrade.ts`
- 卸载: `packages/opencode/src/cli/cmd/uninstall.ts:25`
