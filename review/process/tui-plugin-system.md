# OpenCode TUI Plugin 系统完整实现分析

> 分析时间: 2026-06-15
> 范围: TUI 模式下 plugin 的实现 — 服务端 plugin loader、TUI plugin host、内置插件、slot/route/command 扩展点
>
> 路径以 `packages/` 为根。TUI 插件体系有 3 层：服务端 plugin loader（管理插件生命周期）+ TUI plugin host（提供 slots/routes/commands 能力）+ TUI 内置插件（11 个 feature 插件）。

---

## 一、整体架构

```
┌─────────────────────────────────────────────────────────────┐
│ 服务端 plugin 装载 (packages/opencode/src/plugin/)         │
│   ├── index.ts        装配 / 触发 hooks / 事件总线         │
│   ├── loader.ts       npm 安装 + 动态 import                │
│   ├── install.ts      `opencode plug` 命令实现             │
│   ├── shared.ts       解析 spec / 读 manifest               │
│   ├── meta.ts         Theme/Sound 等元数据                 │
│   └── azure.ts/cloudflare.ts/.../xai.ts 内置 auth provider  │
└─────────────────────────────────────────────────────────────┘
                              ↓ 提供 Plugin.Service
                              ↓ hooks: tool.execute.before/after,
                              ↓        chat.messages.transform,
                              ↓        experimental.text.complete,
                              ↓        config, event, tool.definition, dispose...
┌─────────────────────────────────────────────────────────────┐
│ TUI 专属 plugin 装载 (packages/opencode/src/plugin/tui/)    │
│   ├── runtime.ts      loadTuiPlugins / activate / deactivate │
│   └── internal.ts     internalTuiPlugins → 11 个内置       │
└─────────────────────────────────────────────────────────────┘
                              ↓ 暴露 TuiPluginHost 接口
┌─────────────────────────────────────────────────────────────┐
│ TUI 端 host (packages/tui/src/plugin/)                      │
│   ├── runtime.tsx     createPluginRuntime (slots/routes)    │
│   ├── api.ts          createTuiApi / createPluginRoutes      │
│   ├── slots.tsx       createSlots (Solid slot 注册表)       │
│   ├── adapters.tsx    createTuiApiAdapters (暴露 UI 上下文)  │
│   └── command-shim.ts 旧 v1 plugin API 兼容                  │
└─────────────────────────────────────────────────────────────┘
                              ↓ Solid 组件渲染
┌─────────────────────────────────────────────────────────────┐
│ TUI 内置插件 (packages/tui/src/feature-plugins/)            │
│   ├── builtins.ts     createBuiltinPlugins() = 12 个        │
│   ├── home/           HomeFooter, HomeTips                  │
│   ├── sidebar/        Context, Files, Footer, Lsp, Mcp, Todo│
│   └── system/         Notifications, Plugins, WhichKey,      │
│                       DiffViewer                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、服务端 Plugin Service（`packages/opencode/src/plugin/index.ts`）

### 2.1 接口

```ts
export interface Interface {
  readonly trigger: <Name, Input, Output>(name, input, output) => Effect<Output>
  readonly list: () => Effect<Hooks[]>
  readonly init: () => Effect<void>
}
```

**核心是 `trigger(name, input, output)`** — 走 "out parameter" 模式：每个 hook 拿 `input` 和 `output`，可以 mutate `output`，返回更新后的 `output`。例如：

```ts
// SessionProcessor.handleEvent 里：
yield* events.publish(SessionEvent.Tool.Called, { sessionID, callID, tool, input, ... })
// hook chain：
for (const hook of hooks) {
  void hook["event"]?.({ event: ... })
}
// 实验性文本完成 hook：
ctx.currentText.text = (yield* plugin.trigger(
  "experimental.text.complete",
  { sessionID, messageID, partID },
  { text: ctx.currentText.text },
)).text
```

### 2.2 装配（`index.ts:123-306`）

`Plugin.layer` 在 `InstanceState.make<State>((ctx) => init(ctx))` 里 **per-project** 缓存。`init(ctx)`：

| 行号 | 步骤 |
| ---- | ---- |
| `:132-137` | 准备 `hooks: Hooks[]` 数组 + `EffectBridge` 用于发错误事件 |
| `:139-147` | 构造 `client = createOpencodeClient({...})` 让插件**反向**调 opencode server |
| `:148-164` | 构造 `input: PluginInput` — `client`、`project`、`worktree`、`directory`、`experimental_workspace.register(type, adapter)`、`serverUrl` getter |
| `:166-175` | 加载**内置插件**（`internalPlugins(flags)`）— 9 个 auth provider 插件（Codex、Copilot、Gitlab、Cloudflare Workers、Cloudflare AI Gateway、Azure、DigitalOcean、Snowflake Cortex、Xai）|
| `:177-180` | 读 `cfg.plugin_origins`（用户配置的 plugin 列表）|
| `:182-214` | 调 `PluginLoader.loadExternal({ items: plugins, kind: "server", report })` — 自动 `npm install` + 动态 import，失败发 `Session.Event.Error` 事件 |
| `:215-238` | 顺序调 `applyPlugin(load, input, hooks)` 注册每个插件的 hooks（`Sequential` 保证 hook 注册顺序确定）|
| `:240-249` | 对每个 hook 调 `config?.(cfg)`（通知插件配置） |
| `:251-259` | 注册 event 总线订阅：把 server 事件分发到所有 `hook.event` 回调 |
| `:261-274` | 注册 finalizer：scope 释放时调 `hook.dispose?.()` |
| `:276` | 返回 `{ hooks }` 写进 ScopedCache |

### 2.3 触发链（`index.ts:280-302`）

```ts
const trigger = Effect.fn("Plugin.trigger")(function* (name, input, output) {
  if (!name) return output
  const s = yield* InstanceState.get(state)              // 拿到当前 project 的 hooks
  for (const hook of s.hooks) {                          // 顺序遍历所有插件
    const fn = hook[name] as any
    if (!fn) continue
    yield* Effect.promise(async () => fn(input, output))   // 串行 await
  }
  return output                                          // 返回最终 output
})
```

**关键设计**：
- **串行**执行（不是并行）— 保证 hook chain 顺序确定
- **out parameter 模式** — `output` 在每次 hook 后被覆盖，最后返回
- **缺失 hook 静默跳过**（`if (!fn) continue`）— 插件不必实现所有 hooks

### 2.4 Hooks 列表

服务端 hooks 来自 `@opencode-ai/plugin`（`packages/core/src/plugin/agent.ts`、`command.ts`、`provider.ts`），包括：

| Hook | 用途 | 调用方 |
| ---- | ---- | ------ |
| `tool.execute.before` | 工具执行前修改 args | `SessionTools.resolve` 包装层 |
| `tool.execute.after` | 工具执行后修改 output | 同上 |
| `tool.definition` | 修改工具描述/schema | `ToolRegistry.tools` |
| `chat.messages.transform` | 批量改 message 历史 | `SessionPrompt.runLoop` |
| `experimental.text.complete` | 文本完成时后处理 | `SessionProcessor.handleEvent text-end` |
| `shell.env` | 注入 shell 环境变量 | `SessionPrompt.shell` |
| `command.execute.before` | 命令执行前改 parts | `SessionPrompt.command` |
| `config` | 接收配置变化 | `Plugin.init` |
| `event` | 全局事件分发 | `Plugin.init` 订阅 GlobalBus |
| `dispose` | 清理 | `Plugin.init` finalizer |
| `auth` (内置) | 注册 provider auth | auth plugin 专用 |
| `provider` (内置) | 注册 model provider | provider plugin 专用 |

---

## 三、TUI Plugin Loader（`packages/opencode/src/plugin/tui/runtime.ts`，1130 行）

### 3.1 入口

`runtime.ts:1123-1128`：

```ts
export function createLegacyTuiPluginHost(): TuiPluginHost {
  return {
    start: init,
    dispose,
  }
}
```

调用方：
- `cli/cmd/tui.ts:189-199` — TUI 启动时
- `cli/cmd/attach.ts:81-86` — `opencode attach` 时

**`TuiPluginHost` 接口**（`packages/tui/src/plugin/runtime.tsx:61-69`）：

```ts
export type TuiPluginHost = {
  start(input: {
    api: TuiPluginApi
    config: TuiConfig.Resolved
    runtime: PluginRuntime
    dispose?: () => void
  }): Promise<void>
  dispose(): Promise<void>
}
```

### 3.2 `init(input)`（`runtime.ts:1050-1121`）

执行步骤：

| 行号 | 步骤 |
| ---- | ---- |
| `:1058` | `const cwd = process.cwd()` |
| `:1059` | `const slots = input.runtime.setupSlots(api)`（让 host 把 slot 注册表装上） |
| `:1060-1070` | 构造 `next: RuntimeState = { directory, api, view, dispose, slots, plugins: [], plugins_by_id: Map, pending: Map, dispose_timeout_ms: 5000 }` |
| `:1071` | `runtime = next`（模块级单例，slot 调试用） |
| `:1072-1080` | `view.update({ commands: { activate, deactivate, add, install }, status })` — 把 plugin 管理命令塞进 PluginRuntime |
| `:1081-1090` | 读 `plugin_origins`（用户配置的 plugin 来源），`OPENCODE_PURE` 时跳过 |
| `:1092-1103` | 加载**内置 TUI 插件**（`internalTuiPlugins(flags)`） — 11 个（`internal.ts:6-10` 调用 `createBuiltinPlugins({ experimentalEventSystem })` → `feature-plugins/builtins.ts:21-35`）|
| `:1105-1106` | 加载**外部 TUI 插件**（npm 包、git、本地路径），`resolveExternalPlugins` 自动 install + import |
| `:1108` | 应用初始 enable/disable 状态（从 KV 持久化） |
| `:1109-1116` | **顺序**激活每个启用的插件（`activatePluginEntry`）— 保证注册顺序确定 |
| `:1117` | `view.update({ status: listPluginStatus(next) })` |

### 3.3 内部 TUI 插件（`feature-plugins/builtins.ts:21-35`）

```ts
export function createBuiltinPlugins(options: { experimentalEventSystem: boolean }): BuiltinTuiPlugin[] {
  return [
    HomeFooter,
    HomeTips,
    SidebarContext,
    SidebarMcp,
    SidebarLsp,
    SidebarTodo,
    SidebarFiles,
    SidebarFooter,
    Notifications,
    PluginManager,
    WhichKey,
    DiffViewer,
  ]
}
```

每个都是 `BuiltinTuiPlugin = Omit<TuiPluginModule, "id"> & { id: string; tui: TuiPlugin; enabled?: boolean }`（`builtins.ts:15-19`）— 一个带 `tui` 导出函数的对象，描述插件 ID 和启用状态。

### 3.4 激活一个插件（`runtime.ts:515-555` `activatePluginEntry`）

每个插件的 `tui(input)` 函数接收 `TuiPluginApi` 上下文，**通常**是：

```ts
async (api) => {
  api.register((api) => {
    // 1. 注册 slot
    api.ui.register({ home_logo: { component: () => <MyLogo /> } })
    // 2. 注册 route
    api.route.register([{ name: "my-screen", render: (params) => <MyScreen /> }])
    // 3. 注册 keybind
    api.keymap.register(...)
    // 4. 读 SDK
    await api.sdk.client...
  })
  return {
    dispose: () => { /* 清理 */ }
  }
}
```

TUI 插件的 `tui` 钩子**返回** `Promise<TuiDispose>`（或 `Promise<TuiPlugin | TuiPluginApi | { slots?; routes?; ... }>`）。

---

## 四、TUI Plugin Host（`packages/tui/src/plugin/`）

### 4.1 PluginRuntime（`runtime.tsx:12-35`）

```ts
export function createPluginRuntime() {
  const [commands, setCommands] = createSignal<PluginRuntimeCommands>(emptyCommands)
  const [status, setStatus] = createSignal<ReadonlyArray<TuiPluginStatus>>([])
  const slots = createSlots()

  return {
    Slot: slots.Slot,                          // Solid 组件 - 在 UI 注入内容
    routes: createPluginRoutes(),              // 路由表
    commands,                                 // 信号 - plugin 管理命令
    status,                                   // 信号 - plugin 状态列表
    update(input: { commands?, status? }) { ... },
    clear() { ... },
    setupSlots(api: TuiPluginApi): HostSlots {
      return slots.setup(api)                  // 把 slot 注册表挂到 opentui renderer
    },
  }
}
```

通过 `PluginRuntimeProvider`（`runtime.tsx:73-75`）注入到 Solid 树。

### 4.2 Slot 系统（`slots.tsx:25-65`）

`createSlots()` 暴露：

```ts
{
  Slot,                                          // Solid 组件 - <Slot name="..." mode="...">
  setup(api: HostPluginApi): HostSlots {
    const registry = createSolidSlotRegistry<RuntimeSlotMap, TuiSlotContext>(...)
    const slot = createSlot<RuntimeSlotMap, TuiSlotContext>(registry)
    return {
      register(plugin: HostSlotPlugin) { ... },  // 插件调 api.ui.register({...}) 时调这里
      dispose() { ... },
    }
  },
  clear() { ... },
}
```

- `Slot` 是个 Solid 组件，渲染插件注入的内容
- `HostSlots.register` 是插件注册 slot 的入口

### 4.3 Routes（`api.ts:11-38`）

```ts
export function createPluginRoutes() {
  const routes: RouteMap = new Map()
  const [revision, setRevision] = createSignal(0)

  return {
    register(list: TuiRouteDefinition[]) { ... },
    get(name: string) {
      revision()
      return routes.get(name)?.at(-1)?.render    // 后注册者赢
    },
  }
}
```

后注册的同名 route **覆盖**先注册的（`.at(-1)` 取最后一项）。

### 4.4 TuiPluginApi（`api.ts:42-52`）

```ts
export function createTuiApi(input: Omit<TuiPluginApi, "lifecycle">): TuiPluginApi {
  return {
    ...input,
    lifecycle: {
      signal: new AbortController().signal,
      onDispose() { return () => {} },
    },
  }
}
```

加上 `lifecycle`（`AbortSignal` + 清理回调）。

### 4.5 API Adapters（`adapters.tsx:172-354`）

`createTuiApiAdapters(input)` 把 TUI 内部的 Solid signals / stores 包装成插件可用的 API：

| API 域 | 内容 |
| ------ | ---- |
| `app` | 版本、配置 |
| `attention` | 音频/光标闪烁 |
| `command` (v1 兼容 shim) | 旧版 `api.command` 接口 |
| `keys` | keymap 格式化工具 |
| `keymap` | 完整 keymap 暴露 |
| `mode` | 当前 mode 栈 |
| `route` | navigate / register / current |
| `ui` | Dialog/DialogAlert/Confirm/Prompt/Select + toast |
| `state` | sync 状态读取 |
| `theme` | 主题切换 |
| `event` | 订阅 server 事件 |
| `sdk` | SDK 客户端 + SSE 订阅 |
| `renderer` | opentui renderer（用于 slot 注册）|
| `lifecycle` | AbortSignal + dispose |

### 4.6 与 App 组件集成（`app.tsx`）

```ts
// app.tsx:140-213:  Plugin 集成 prop + finalizer
input: TuiInput = {
  ...
  pluginHost: TuiPluginHost
  ...
}
// 在 effect.scoped 里：
yield* Effect.addFinalizer(() =>
  Effect.promise(async () => {
    try { await input.pluginHost.dispose() } catch (error) { ... }
  })
)

// app.tsx:287-322:  Solid Provider 树
<PluginRuntimeProvider value={pluginRuntime}>
  <RouteProvider ...>
    <TuiConfigProvider config={input.config}>
      <PluginRuntimeProvider value={pluginRuntime}>
        <SDKProvider ...>
          ...
          <App pluginHost={input.pluginHost} ... />

// app.tsx:351-419:  App 组件实际用法
function App(props: { pluginHost: TuiPluginHost }) {
  const pluginRuntime = usePluginRuntime()
  const api = createTuiApi(createTuiApiAdapters({ ... Slot: pluginRuntime.Slot, ... }))
  const [ready, setReady] = createSignal(false)
  props.pluginHost
    .start({ api, config: tuiConfig, runtime: pluginRuntime, dispose: () => attention.dispose() })
    .catch(console.error)
    .finally(() => setReady(true))
  ...
}
```

启动时序：
1. TUI 启动 → `cli/cmd/tui.ts:189` 创建 `createLegacyTuiPluginHost()`
2. 传进 `run({ pluginHost, ... })`（`cli/tui/layer.ts:5-7` 包 `Global.defaultLayer`）
3. 实际渲染时（`app.tsx:394`）调 `pluginHost.start({ api, config, runtime, dispose })`
4. `pluginHost.start` = `init`（`runtime.ts:1050`）— 加载并激活所有插件
5. 插件通过 `api.ui.register({...})` 注册 slot，slot 立即可被 `<Slot name="...">` 渲染

---

## 五、Slot 注入点（所有命名 slot）

TUI 各处用 `<pluginRuntime.Slot name="..." mode="...">` 挖了"插件可注入内容"的洞：

| Slot 名 | 位置 | 模式 | 文件 |
| ------ | ---- | ---- | ---- |
| `app` | 整个 App 外层 | (默认) | `app.tsx:1094` |
| `app_bottom` | 整个 App 最底 | (默认) | `app.tsx:1092` |
| `home_logo` | Home 页面 logo 位置 | `replace` | `routes/home.tsx:76-79` |
| `home_prompt` | Home 页面 prompt 框 | `replace` | `routes/home.tsx:82-84` |
| `home_prompt_right` | Home prompt 框右侧 | (默认) | `routes/home.tsx:83` |
| `home_bottom` | Home 页面底部 | (默认) | `routes/home.tsx:86` |
| `home_footer` | Home 页面底部 footer | `single_winner` | `routes/home.tsx:91` |
| `session_prompt` | Session 页面 prompt 框 | (默认) | `routes/session/index.tsx:1300-1319` |
| `session_prompt_right` | Session prompt 右侧 | (默认) | `routes/session/index.tsx:1317` |
| `sidebar_content` | Session 侧边栏内容 | (默认) | `routes/session/sidebar.tsx:85` |
| `sidebar_footer` | Session 侧边栏底部 | `single_winner` | `routes/session/sidebar.tsx:90-98` |

**Slot mode 语义**（来自 `@opentui/solid`）：
- 默认 / `"merge"`：多个插件内容**叠加**
- `"replace"`：只显示最后注册的
- `"single_winner"`：只显示第一个响应（其他丢弃）

---

## 六、TUI 插件实际形态（举例）

`HomeFooter`（`packages/tui/src/feature-plugins/home/footer.tsx`）是内置插件之一，其形态：

```ts
import type { TuiPlugin, TuiPluginModule } from "@opencode-ai/plugin/tui"

const HomeFooter: TuiPlugin = async (api) => {
  // 1. 注册 slot 内容
  api.ui.register({
    home_footer: {
      component: () => <FooterInfo />,
    },
  })

  // 2. 订阅事件 / 数据
  const { sdk } = api
  // ...

  // 3. 返回 dispose
  return {
    dispose: () => {
      // 清理
    },
  }
}

const HomeFooterModule: TuiPluginModule = {
  id: "home-footer",
  tui: HomeFooter,
}

export default HomeFooterModule
```

外部 npm 插件跟这同构：

```js
// my-plugin/index.js
export const tui = async (api) => {
  api.ui.register({ home_logo: { component: () => "🌟" } })
  return { dispose: () => {} }
}
```

---

## 七、插件注册/启用/禁用/安装的运行时接口

`PluginRuntime` 暴露 4 个命令（`runtime.tsx:37-57`）：

| 命令 | 用途 |
| ---- | ---- |
| `commands.activate(id)` | 启用指定 ID 插件（`runtime.ts:557`） |
| `commands.deactivate(id)` | 禁用（`runtime.ts:564`） |
| `commands.add(spec)` | 加新 plugin 来源（写 `opencode.json` 持久化）|
| `commands.install(spec, options)` | `npm install` 一个 plugin 包（`runtime.ts:890`）|

TUI 端用户操作入口是 `system/plugins` 内置插件（`packages/tui/src/feature-plugins/system/plugins.tsx`）— 渲染 plugin 状态列表 + 提供 activate/deactivate/install UI。

---

## 八、几个值得注意的点

1. **服务端 plugin 和 TUI plugin 是两套独立体系**：
   - 服务端 hooks 影响 LLM 行为（tool/command/exec）
   - TUI plugin 影响 UI 渲染（slot/route/keybind）
   - 同一 npm 包可以**同时**导出服务端 `server` 函数和 TUI `tui` 函数，opencode 在不同进程上下文加载对应部分

2. **Plugin API 演进**：
   - v1（兼容）：`api.command` shim 走 `createCommandShim`（`adapters.tsx:177`）
   - v2（当前）：`api.ui.register`、`api.route.register` 等模块化
   - `createLegacyTuiPluginHost` 命名暗示未来会有 v2 host

3. **Plugin slot 是声明式的**：插件**只**注册 `{ name: { component, mode?, ... } }`；Solid 树里的 `<Slot name="home_logo">` 自动找匹配 + 渲染
4. **Plugin route 注册可"覆盖"**：`routes.get(name)?.at(-1)?.render` — 后注册同名 route 覆盖前者，**典型用法**是 `single_winner` slot + route 覆盖做"插件可换主题"
5. **Plugin 加载是 lazy**：TUI 启动时 `createLegacyTuiPluginHost()` 只是构造 host（无副作用），真正加载/激活在 `app.tsx:394` `pluginHost.start()` — 也就是第一帧 Solid 渲染时
6. **Plugin 状态持久化**：`KV_KEY = "plugin_enabled"`（`runtime.ts:122`）— 启用/禁用状态写到 `kv` 持久化
7. **Plugin 错误处理**：通过 `EventV2Bridge` 发 `Session.Event.Error` 事件，TUI 端用 `permission.asked` 类似的弹窗显示（虽然 plugin 错误没有专门的 UI）
8. **Sandbox 没**：TUI plugin 在主进程跑（不像 server plugin 在 `createServer` 里），所以 `tui` 插件可以**完全**访问 TUI 的所有内部 API（renderer、kv、keymap、dialog...）。这意味着**外部 plugin 的信任要求跟 server plugin 一样高**

---

## 九、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| 服务端 Plugin Service | `packages/opencode/src/plugin/index.ts:1-316` |
| 服务端 PluginLoader | `packages/opencode/src/plugin/loader.ts` |
| 服务端 install 命令 | `packages/opencode/src/plugin/install.ts` |
| 服务端内置 auth plugins | `packages/opencode/src/plugin/{azure,cloudflare,...}.ts` |
| TUI Plugin host 实现 | `packages/opencode/src/plugin/tui/runtime.ts:1-1130` |
| TUI Plugin 内置列表 | `packages/opencode/src/plugin/tui/internal.ts` |
| TUI PluginRuntime | `packages/tui/src/plugin/runtime.tsx:1-81` |
| TUI API 构造 | `packages/tui/src/plugin/api.ts:1-52` |
| TUI API 适配 | `packages/tui/src/plugin/adapters.tsx:172-354` |
| TUI Slot 系统 | `packages/tui/src/plugin/slots.tsx:1-65` |
| TUI 旧版 command shim | `packages/tui/src/plugin/command-shim.ts` |
| 内置 TUI 插件列表 | `packages/tui/src/feature-plugins/builtins.ts:21-35` |
| 内置插件实现 | `packages/tui/src/feature-plugins/{home,sidebar,system}/*.tsx` |
| 集成进 App 组件 | `packages/tui/src/app.tsx:140, 213, 287, 308, 351, 374-406` |
| TUI 启动时创建 host | `packages/opencode/src/cli/cmd/tui.ts:189-199` |
| `opencode attach` 也用 | `packages/opencode/src/cli/cmd/attach.ts:81-86` |
| Slot 注入点 | `packages/tui/src/{app,routes/home,routes/session/index,routes/session/sidebar}.tsx` |
