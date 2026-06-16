# OpenCode TUI 模式启动与初始化全流程

> 分析时间: 2026-06-15
> 范围: 用户在终端输入 `opencode [project]` → 屏幕上出现 OpenCode 界面
>
> 涉及的相对路径都以 `packages/opencode/` / `packages/tui/` 为根。

---

## 一、整体时序

```
进程 ┌─ index.ts: yargs 中间件 (Heap.start, env 注入)
    ├─ yargs 路由 → TuiThreadCommand ($0 默认命令)
    │
    ├─ tui.ts: 解析 project, chdir
    ├─ tui.ts: new Worker(worker.ts) ────────────────┐
    ├─ tui.ts: Rpc.client(worker)                    │ worker:
    ├─ tui.ts: 加载 TuiConfig.get()                  │  Heap.start
    ├─ tui.ts: 决定 transport (external or internal) │  GlobalBus → Rpc.emit
    ├─ tui.ts: validateSession (若给 --session)      │  Rpc.listen(rpc) ← fetch/server/shutdown...
    ├─ tui.ts: Effect.runPromise(run({...}))         │
    │       │                                        │
    │       ├─ app.tsx: createCliRenderer (opentui)  │
    │       ├─ app.tsx: keymap + SIGHUP              │
    │       ├─ app.tsx: 预热调色板 + 等 theme        │
    │       ├─ app.tsx: render(<...Provider 树>)     │
    │       │     ├─ SDKProvider → startSSE         ─┘ RPC: global.event
    │       │     ├─ SyncProvider → 订阅 SDK event
    │       │     └─ ...其余 provider mount
    │       ├─ app.tsx: <App> → pluginHost.start() 异步
    │       └─ app.tsx: Deferred.await(shutdown) 阻塞
    │
    └─ 用户退出 (Ctrl-C / 路由切换 / SIGHUP)
       ├─ destroyRenderer → Deferred.resolve
       ├─ finalizer: pluginHost.dispose, TuiAudio.dispose
       ├─ tui.ts finally: stop() = client.shutdown (5s) + worker.terminate
       └─ tui.ts: process.exit(0)
```

---

## 二、阶段 0 — 进程入口与 yargs 装配

| 步骤 | 代码位置 | 说明 |
| ---- | -------- | ---- |
| 启动壳 | `packages/opencode/bin/opencode` | npm bin 透传 stdio 给 Bun 编译二进制（很短命）|
| 真实入口 | `packages/opencode/src/index.ts:33` | `args = hideBin(process.argv)` |
| yargs 装配 | `packages/opencode/src/index.ts:45-78` | 注册 22 个子命令 + 默认 `$0 [project]`；middleware 注入 `OPENCODE_PRINT_LOGS` / `OPENCODE_LOG_LEVEL` / `OPENCODE_PURE`、`Heap.start()`、env `AGENT=1` / `OPENCODE=1` / `OPENCODE_PID=<pid>` |
| yargs 解析 | `packages/opencode/src/index.ts:118-127` | `opencode` 无子命令 → 匹配默认 `$0` → `TuiThreadCommand` handler |
| 错误兜底 | `packages/opencode/src/index.ts:128-141` | `catch → FormatError → UI.error → process.exitCode = 1`；`finally { process.exit() }` 强制退出（避免 docker MCP 等子进程挂住）|

> `TuiThreadCommand` 的注册：`packages/opencode/src/index.ts:83`，`command: "$0 [project]"`。`$0` 是 yargs 默认命令标记，等价于 `opencode` 直接进 TUI。`opencode tui` 实际上不识别（`tui` 会被当成项目路径 `./tui`）。

---

## 三、阶段 1 — `TuiThreadCommand` handler

入口: `packages/opencode/src/cli/cmd/tui.ts:71-223`

按行号顺序的执行步骤：

| # | 步骤 | 行号 | 关键点 |
| - | ---- | ---- | ------ |
| 1 | Win32 Ctrl-C 守卫 | `:108` | `win32InstallCtrlCGuard()`（`@opencode-ai/tui/terminal-win32`）|
| 2 | 校验 `--fork` | `:111-115` | 必须配 `--continue` 或 `--session`；失败 `process.exitCode = 1` |
| 3 | 解析项目路径 | `:119` | `resolveThreadDirectory(args.project)`（`:65-69`）：绝对路径用之，相对路径从 `PWD` 解析，否则用 `cwd` |
| 4 | 选 worker 文件 | `:120` + `target()` (`:51-56`) | 优先 `OPENCODE_WORKER_PATH` env；否则 `./cli/tui/worker.js`（构建产物）；回退 `../tui/worker.ts`（dev 直接跑 TS）|
| 5 | **chdir** | `:122-126` | `process.chdir(next)`，失败报错退出 |
| 6 | **启动 Bun Worker 线程** | `:129` | `new Worker(file)` — 跑 `cli/tui/worker.ts` |
| 7 | 建 RPC 客户端 | `:130` | `Rpc.client<typeof rpc>(worker)`（`util/rpc.ts:19-64`）|
| 8 | 注册 SIGUSR2 → reload | `:134` | `process.on("SIGUSR2", reload)` → `client.call("reload", undefined)` |
| 9 | 定义 `stop` 关闭函数 | `:137-143` | `withTimeout(client.call("shutdown", undefined), 5000).catch(() => {})` + `worker.terminate()` |
| 10 | 读 prompt 输入 | `:145` + `input()` (`:58-63`) | `--prompt` 与 stdin 拼接（`isTTY` 为 false 时 `Bun.stdin.text()`）|
| 11 | **加载 TUI 配置** | `:146` | `TuiConfig.get()` — `@/config/tui` 模块的 `Service.get`（合并 plugins/keymap/theme）|
| 12 | **解析网络选项** | `:148` | `resolveNetworkOptionsNoConfig(args)`（`src/cli/network.ts:46-64`）— 检查 `process.argv` 是否有 `--port`/`--hostname`/`--mdns`/`--mdns-domain` 显式给定，否则落回 `Config.server.*` |
| 13 | **判定 transport 模式** | `:149-167` | `external` = 任一网络选项被显式给出或非默认；<br>external → 调 `client.call("server", network)` 让 worker 起 `Server.listen` 并返回 URL；<br>internal → URL 写死 `http://opencode.internal`，fetch/EventSource 走 `createWorkerFetch(client)` + `createEventSource(client)`（RPC 桥到 worker 内的 in-process server）|
| 14 | **校验 session** | `:170-180` | 若 `--session <id>`：`validateSession(...)`（`cli/tui/validate-session.ts`）→ SDK `client.session.get(...)` 确认存在；`SessionID` brand 解析失败抛错 |
| 15 | **异步 upgrade check** | `:182-184` | 1s 后 `client.call("checkUpgrade", { directory: cwd })`（unref，不阻塞）|
| 16 | **跑 TUI** | `:190-212` | `Effect.runPromise(run({...}))`；`run` 来自 `cli/tui/layer.ts:5`，用 `Global.defaultLayer` 包裹 |
| 17 | 收尾 | `:213-221` | `finally { stop() }` → 5s 内 shutdown RPC + terminate worker；最后 `process.exit(0)` |

---

## 四、阶段 2 — Worker 线程初始化

文件: `packages/opencode/src/cli/tui/worker.ts`（71 行）。

> **关键**：Worker 是 Bun `Worker` 线程，不是独立 OS 进程。

执行步骤：

| # | 步骤 | 行号 | 说明 |
| - | ---- | ---- | ---- |
| 1 | 启堆监控 | `:14` + `Heap.start()`（`cli/heap.ts:12-43`）| RSS > 2GB 写 V8 heap snapshot 到 `~/.local/share/opencode/log/heap-*.heapsnapshot`；定时 1 分钟，`.unref()` |
| 2 | **订阅 global bus** | `:17-19` | `GlobalBus.on("event", e => Rpc.emit("global.event", e))` — worker 内发生的 global event 全部经 RPC event 转给主线程 |
| 3 | 声明 RPC 方法 | `:23-69` | `fetch` / `snapshot` / `server` / `checkUpgrade` / `reload` / `shutdown` |
| 4 | `Rpc.listen(rpc)` | `:71` | `onmessage` 收到 `rpc.request` → 调方法 → `postMessage(rpc.result)` |

RPC 方法细节：

- **`fetch({url, method, headers, body})`**：直接交给 `Server.Default().app.fetch(request)`（**进程内** HTTP 路由，不走 socket），返回 `{status, headers, body}`。这是 internal transport 模式下 TUI 调 server 的"假 HTTP"
- **`snapshot()`**：`writeHeapSnapshot("server.heapsnapshot")`，给 `Tui.run` 的 `onSnapshot` hook 用
- **`server({port, hostname, mdns?, cors?})`**：停旧 server，调 `Server.listen(input)`（`src/server/server.ts:72`），返回 `{url}`；走 `NodeHttpServer` 真绑端口；端口策略：给定非 0 用之，`0` 则先试 4096 失败回退随机
- **`checkUpgrade({directory})`**：`InstanceRuntime.load({directory})` + `upgrade()`，吞错
- **`reload()`**：`Config.Service.invalidate()` + `disposeAllInstancesAndEmitGlobalDisposed({swallowErrors:true})`
- **`shutdown()`**：`InstanceRuntime.disposeAllInstances()` + `server.stop(true)`

> **关键点**：worker 不会主动起 HTTP server — 只有 main 调 `client.call("server", ...)` 时才 `Server.listen`。

---

## 五、阶段 3 — TUI 运行时 setup

文件: `packages/tui/src/app.tsx:178-349`

`run` 是 `Effect.fn("Tui.run")`，由 `packages/opencode/src/cli/tui/layer.ts:5-7` 包了 `Global.defaultLayer`：

```ts
// packages/opencode/src/cli/tui/layer.ts
export function run(input: TuiInput) {
  return runTui(input).pipe(Effect.provide(Global.defaultLayer))
}
```

执行步骤：

| # | 步骤 | 行号 | 说明 |
| - | ---- | ---- | ---- |
| 1 | 取 `Global.Service` | `:179` | 解析 `Global.Path`（home/state/data）|
| 2 | `Effect.scoped(...)` | `:181` | 整个 TUI 在一个 scope 内，结束时跑所有 finalizer |
| 3 | **`createCliRenderer`** | `:183-203` | `@opentui/core` 创建终端渲染器；关键配置：`externalOutputMode: "passthrough"` / `targetFps: 60` / `exitOnCtrlC: false` / `useKittyKeyboard: {}` / `autoFocus: false` / `useMouse: !Flag.OPENCODE_DISABLE_MOUSE && input.config.mouse` / `consoleOptions.keyBindings: [{name:"y",ctrl:true,action:"copy-selection"}]`；acquire/release 包了 `destroyRenderer(renderer)` |
| 4 | Win32 raw 模式 | `:204` | `win32DisableProcessedInput()` |
| 5 | 默认 keymap | `:205` | `createDefaultOpenTuiKeymap(renderer)` |
| 6 | 装 opencode keymap | `:206-209` | `registerOpencodeKeymap(keymap, renderer, input.config)`；release 时 unregister |
| 7 | 加 plugin 释放 finalizer | `:210-218` | `input.pluginHost.dispose()` 异步调用 |
| 8 | 加 audio 释放 finalizer | `:219` | `TuiAudio.dispose()` |
| 9 | SIGHUP 守卫 | `:220-225` | `process.on("SIGHUP", () => destroyRenderer(renderer))` |
| 10 | Shutdown Deferred | `:220, :226` | `renderer.once("destroy", () => Deferred.doneUnsafe(shutdown, Effect.void))` |
| 11 | 建 plugin runtime | `:227` | `createPluginRuntime()` |
| 12 | **预热调色板** | `:231` | `void renderer.getPalette({ size: 16 }).catch(() => undefined)` — 避免 system theme 首帧闪烁 |
| 13 | **等 theme 模式** | `:232` | `await renderer.waitForThemeMode(1000) ?? "dark"`（最多 1s）|
| 14 | **`render(() => <...>, renderer)`** | `:235-337` | Solid 树挂到 opentui renderer — 见阶段 4 |
| 15 | **`Deferred.await(shutdown)`** | `:339` | 阻塞到 renderer 被 destroy（用户退出 / Ctrl-C / SIGHUP / 错误）|
| 16 | 收尾输出 | `:343-348` | Win32 flush input buffer + 把 `exit.reason` / `exit.epilogue` 写到 stderr/stdout |

---

## 六、阶段 4 — Provider 树

`render` 调用 `app.tsx:235-337`，自外向内依次挂载：

```
ExitProvider                        exit hook → destroyRenderer
  EpilogueProvider                  终态消息 setter
    ErrorBoundary                   错误兜底 → <ErrorComponent mode=...>
      TuiPathsProvider              {cwd, home, state, worktree}
        TuiTerminalEnvironmentProvider  {platform, multiplexer, displayServer}
          TuiStartupProvider        initialRoute? + skipInitialLoading (OPENCODE_FAST_BOOT)
            ClipboardProvider
              OpencodeKeymapProvider
                ArgsProvider {input.args}        // continue/sessionID/agent/model/prompt/fork
                  KVProvider                    // 持久化键值
                    ToastProvider
                      RouteProvider             // 初始: args.continue → {type:"session", sessionID:"dummy"}
                        TuiConfigProvider {input.config}
                          PluginRuntimeProvider
                            SDKProvider {url, directory, fetch, headers, events}
                              ProjectProvider
                                SyncProvider
                                  DataProvider
                                    ThemeProvider mode={mode}
                                      LocalProvider
                                        PromptStashProvider
                                          DialogProvider (ui/dialog)
                                          +DialogProvider (component/dialog-provider)
                                            FrecencyProvider
                                              PromptHistoryProvider
                                                PromptRefProvider
                                                  EditorContextProvider
                                                    <App onSnapshot pluginHost />
```

### 6.1 SDKProvider（`packages/tui/src/context/sdk.tsx:11-150`）

- 建 `createOpencodeClient({ baseUrl, signal, directory, fetch, headers })`
- `fetch` 是 `createWorkerFetch(client)`（internal）或 `undefined`（external，让 SDK 走 `globalThis.fetch`）
- **SSE**：`startSSE()` → `sdk.global.event({ signal, sseMaxRetryAttempts: 0 })`
  - 数据来源：worker 的 `GlobalBus.on("event", ...)` 转发成 `Rpc.emit("global.event", event)`，主线程 `createEventSource(client)` 把 RPC event 装成 EventSource
  - 重连：1s → 30s 退避
- 事件 batch 16ms 单次 Solid 重渲染（`handleEvent` + `flush`）

### 6.2 SyncProvider（`packages/tui/src/context/sync.tsx:53-640`）

- Solid `createStore` 装下：providers / agents / commands / permission / question / config / sessions / messages / parts / lsp / mcp / todo / ...
- 订阅 SDK 的 `event` emitter，把 server push 的 global event 同步到 store
- `sync.ready = true` 标志 bootstrap 完成
- `args.continue` 时也等 sync ready

### 6.3 TuiStartupProvider（`packages/tui/src/context/runtime.tsx:42-44`）

- `initialRoute` 从 `OPENCODE_ROUTE` env 取
- `skipInitialLoading` 从 `OPENCODE_FAST_BOOT` 决定是否跳过启动 loading 蒙层

### 6.4 TuiConfigProvider（`packages/tui/src/config/index.tsx`）

- 把 `input.config`（来自 `TuiConfig.get()`）注入：keymap / theme / plugins

### 6.5 RouteProvider

- 初始路由由 `args.continue` 决定：
  - `args.continue` → `{ type: "session", sessionID: "dummy" }`
  - 否则 → `undefined`（默认走 Home）

---

## 七、阶段 5 — `<App />` 组件

文件: `packages/tui/src/app.tsx:351+`

| # | 步骤 | 行号（参考） | 说明 |
| - | ---- | ---- | ---- |
| 1 | 拉所有 hooks | `:352-373` | `useTuiStartup` / `useTuiConfig` / `useRoute` / `useTerminalDimensions` / `useRenderer` / `useDialog` / `useLocal` / `useKV` / `useOpencodeKeymap` / `useEvent` / `useSDK` / `useToast` / `useTheme` / `useSync` / `useProject` / `useExit` / `usePromptRef` / `usePluginRuntime` |
| 2 | 注意力 | `:371` | `createTuiAttention({ renderer, config: tuiConfig, kv })` — 铃声/光标闪烁 |
| 3 | 插件 API 表面 | `:374-392` | `createTuiApi(createTuiApiAdapters({ ... }))` — 暴露给插件的 TUI API |
| 4 | **启插件** | `:394-406` | `props.pluginHost.start({ api, config, runtime, dispose }).catch().finally(() => setReady(true))`；`setReady(true)` 触发 `StartupLoading` 消失 |
| 5 | 拦截 Ctrl-Y 复制 | `:409-416` | `keymap.intercept("key", ..., {priority: 1})` 调 `Selection.handleSelectionKey` |
| 6 | 终端选区复制 | `:423-432` | `renderer.console.onCopySelection = async (text) => clipboard.write?.(text).then(toast.show)` |
| 7 | 读 KV | `:433-436` | `terminal_title_enabled` / `paste_summary_enabled` |
| 8 | 终端标题 | `:439+` | `createEffect(() => renderer.setTerminalTitle(...))` 跟随 route + session |
| 9 | **路由分派** | `:1051` | `return render({ params: route.data.data })`：<br>• `type: "home"` → `<Home />`（`packages/tui/src/routes/home.tsx`）<br>• `type: "session"` → `<Session sessionID={...} />`（`packages/tui/src/routes/session/index.tsx:176`）<br>• 默认进 home |

---

## 八、阶段 6 — Home 首屏

文件: `packages/tui/src/routes/home.tsx`

| # | 步骤 | 行号 | 说明 |
| - | ---- | ---- | ---- |
| 1 | Provider 包装 | `:71` | `<HomeSessionDestinationProvider>` |
| 2 | Logo | `:76-79` | `<Logo />` 包在 `<pluginRuntime.Slot name="home_logo" mode="replace">` — 插件可替换 |
| 3 | 提示 placeholder | `:17-20` | normal: `["Fix a TODO in the codebase", "What is the tech stack of this project?", "Fix broken tests"]`；shell: `["ls -la", "git status", "pwd"]` |
| 4 | Prompt 输入框 | — | `<Prompt />` 组件 |
| 5 | 清终端选区 | `:40-42` | `onMount(() => editor.clearSelection())` |
| 6 | **自动 submit** `--prompt` | `:59-68` | `createEffect` 等 `sync.ready && local.model.ready` 后调 `r.submit()` |
| 7 | 一次性填入 | `:44-56` | `bind` 一次性把 `route.prompt` / `args.prompt` 填到 prompt 组件；用 `once` 闭包变量标志 |

---

## 九、阶段 7 — 首帧绘制

- OpenTUI renderer 在 Solid 树 mount 后立即画第一帧
- `TimeToFirstDraw`（`@opentui/solid`）组件是首帧完成信号（`app.tsx:1075` 处使用）
- 用户看到：terminal title 改为 "OpenCode" / logo / prompt 输入框 / 底部 status bar / toast 容器 / **loading 蒙层**（若插件 `ready` 还没 true 且超过 500ms）
- `StartupLoading` 组件（`packages/tui/src/component/startup-loading.tsx:5-63`）：插件 ready 之前显示 "Loading plugins..."；ready 后显示 "Finishing startup..."，3s 内或立刻隐藏

---

## 十、阶段 8 — 退出流程

| 触发 | 链路 |
| ---- | ---- |
| **Ctrl-C** | tui.ts:108 的 Win32 guard + tui.ts 默认未注册 → 走 opentui renderer 的 Ctrl-C 处理（`exitOnCtrlC: false`，由 keymap 注册响应）|
| **SIGHUP** | tui.ts:108 守卫 + `:221-225` `process.on("SIGHUP", onSighup)` → `destroyRenderer(renderer)` |
| **Renderer 销毁** | `destroyRenderer` 触发 `Deferred.doneUnsafe(shutdown, Effect.void)`（`:226`）|
| **`<App>` exit** | `ExitProvider` 的 `exit={(reason) => { exit.reason = reason; destroyRenderer(renderer) }}`（`:238-243`）|
| **scope 关闭** | `Effect.scoped` 收尾 finalizer 链：`pluginHost.dispose` → `TuiAudio.dispose` → `unregisterOpencodeKeymap` → `destroyRenderer` |
| **await shutdown 返回** | `:340` 返回 `{epilogue, reason}`；`:343-348` 把 reason 写 stderr、epilogue 写 stdout |
| **run 结束** | `Effect.runPromise` 走 `finally: stop()`（`tui.ts:213-215`）|
| **stop()** | `tui.ts:137-143` 调 `client.call("shutdown", undefined)`（5s 超时）→ `worker.terminate()` |
| **进程退出** | `tui.ts:221` `process.exit(0)` |

---

## 十一、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| 入口 + yargs | `packages/opencode/src/index.ts:45-127` |
| TUI 命令 | `packages/opencode/src/cli/cmd/tui.ts:71-223` |
| Worker 线程 | `packages/opencode/src/cli/tui/worker.ts:1-71` |
| RPC 协议 | `packages/opencode/src/util/rpc.ts:1-66` |
| 共享 HTTP server（worker 调） | `packages/opencode/src/server/server.ts:72-225` |
| 网络选项解析 | `packages/opencode/src/cli/network.ts:6-64` |
| TUI 渲染入口（Effect wrap） | `packages/opencode/src/cli/tui/layer.ts:5-7` |
| TUI 渲染主循环 | `packages/tui/src/app.tsx:178-349` |
| App 组件 | `packages/tui/src/app.tsx:351+` |
| SDK context | `packages/tui/src/context/sdk.tsx:11-150` |
| Sync store | `packages/tui/src/context/sync.tsx:53-640` |
| Runtime context | `packages/tui/src/context/runtime.tsx:34-44` |
| 启动 loading 蒙层 | `packages/tui/src/component/startup-loading.tsx:5-63` |
| Home 首屏 | `packages/tui/src/routes/home.tsx:22-95` |
| Session 路由 | `packages/tui/src/routes/session/index.tsx:176+` |
| Heap 监控 | `packages/opencode/src/cli/heap.ts:12-43` |
| 进程模式（默认） | `packages/opencode/src/cli/cmd/tui.ts:72`（`command: "$0 [project]"`）|
| 网络选项默认值 | `packages/opencode/src/cli/network.ts:6-33` |

---

## 十二、总结

TUI 模式启动是一条**主进程 + Worker 线程 + RPC 桥 + 进程内 Server + opentui 渲染**的链路：

1. **进程模型**：CLI 主进程 + Bun Worker 线程（`new Worker`），通过自定义 JSON RPC（`util/rpc.ts`）通信
2. **双 transport**：external（真 HTTP）通过 `Server.listen` 起端口；internal（默认）走 RPC 把请求转发到 worker 的 `Server.Default().app.fetch` 进程内处理
3. **事件流**：worker 的 `GlobalBus` → `Rpc.emit` → 主线程 `createEventSource` → `SDKProvider.startSSE` → `SyncProvider` 同步到 Solid store
4. **渲染栈**：`@opentui/core` createCliRenderer + `@opentui/solid` render + 自定义 provider 树 + `<App>` 路由分派
5. **生命周期**：`Effect.scoped` 包住整个 TUI；shutdown 由 `renderer.once("destroy")` 触发；finalizer 链清理 plugin/audio/keymap/renderer
6. **首帧关键点**：调色板预热 + theme mode 等待（1s 超时） + Provider 树 mount + plugin host 异步 start（决定 loading 蒙层是否显示）
