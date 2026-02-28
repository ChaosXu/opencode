# OpenCode TUI 完整流程分析

> 分析时间: 2026-02-28

---

# 第一部分：命令执行流程

## 一、opencode 命令执行流程

```
用户输入 "opencode"
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ bin/opencode (Node.js 启动器)                                │
│ - 检查 OPENCODE_BIN_PATH 环境变量                            │
│ - 查找缓存的二进制文件 .opencode                             │
│ - 检测 CPU 支持的 AVX2 指令集                                │
│ - 根据平台和架构查找正确的二进制                              │
└─────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ packages/opencode/src/index.ts (yargs CLI 入口)             │
│ - 初始化日志系统                                             │
│ - 执行数据库迁移（如首次运行）                               │
│ - 解析命令行参数                                             │
│ - 注册所有子命令                                              │
└─────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────┐
│ 默认命令: TuiThreadCommand (command: "$0 [project]")       │
│ packages/opencode/src/cli/cmd/tui/thread.ts                 │
│ - 启动终端 TUI 界面                                          │
│ - 连接后端服务器                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、TuiThreadCommand 触发机制

### 1. 命令注册

在 `packages/opencode/src/index.ts` 中：

```typescript
cli = cli
  .command(AcpCommand)
  .command(McpCommand)
  .command(TuiThreadCommand) // <-- 注册 TUI 命令
  .command(AttachCommand)
  .command(RunCommand)
// ... 其他命令
```

### 2. 默认命令

TuiThreadCommand 的定义（`tui/thread.ts:49`）：

```typescript
command: "$0 [project]"
```

- `$0` 在 yargs 中表示**默认命令**
- 当用户运行 `opencode` 不带任何参数时，会自动匹配这个命令

### 3. 执行流程

```
终端输入: opencode
              │
              ▼
         yargs 解析
              │
              ▼
    没有指定子命令 → 匹配 "$0"
              │
              ▼
   TuiThreadCommand.handler 被调用
              │
              ▼
   启动终端 TUI 界面
```

**结论**: `opencode` 等同于 `opencode tui`

---

# 第二部分：提示词处理流程

## 三、提示词完整处理流程

### 1. 前端 - 用户输入提示词

**文件**: `packages/app/src/components/prompt-input/submit.ts`

```typescript
// 检测 "/" 开头的命令
if (text.startsWith("/")) {
  const [cmdName, ...args] = text.split(" ")
  const commandName = cmdName.slice(1)

  // 调用 SDK 发送命令到后端
  client.session.command({
    sessionID: session.id,
    command: commandName,
    arguments: args.join(" "),
    agent,
    model: `${model.providerID}/${model.modelID}`,
    variant,
    parts: images.map(...),
  })
}
```

### 2. createPromptSubmit 调用时机

- 在 `prompt-input.tsx` 组件初始化时调用（第 960 行）
- 返回 `handleSubmit` 函数

**触发时机**:

1. **按键触发** - 当用户在输入框中按 `Enter` 键且不按 Shift 时:

   ```typescript
   if (event.key === "Enter" && !event.shiftKey) {
     handleSubmit(event)
   }
   ```

2. **表单提交** - 通过 `<DockShellForm onSubmit={handleSubmit}>` 绑定

### 3. SDK 发送请求

- POST `/:sessionID/prompt_async`
- 使用 SSE 流式响应

### 4. 后端 API 路由

**文件**: `packages/opencode/src/server/routes/session.ts:771`

```typescript
.post("/:sessionID/prompt_async", ...)
async (c) => {
  const sessionID = c.req.valid("param").sessionID
  const body = c.req.valid("json")
  SessionPrompt.prompt({ ...body, sessionID })
}
```

### 5. SessionPrompt.prompt()

**文件**: `packages/opencode/src/session/prompt.ts:158`

```typescript
export const prompt = fn(PromptInput, async (input) => {
  const session = await Session.get(input.sessionID)
  await SessionRevert.cleanup(session)

  const message = await createUserMessage(input)
  await Session.touch(input.sessionID)

  if (input.noReply === true) {
    return message
  }

  return loop({ sessionID: input.sessionID })
})
```

### 6. SessionPrompt.loop() - AI 交互循环

**文件**: `packages/opencode/src/session/prompt.ts:274`

```typescript
export const loop = fn(LoopInput, async (input) => {
  const { sessionID, resume_existing } = input

  const abort = resume_existing ? resume(sessionID) : start(sessionID)

  while (true) {
    SessionStatus.set(sessionID, { type: "busy" })

    // 获取历史消息
    let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))

    // 获取模型配置
    const model = await Provider.getModel(...)

    // 处理子任务
    const task = tasks.pop()
    if (task?.type === "subtask") {
      // 执行子任务
    }

    // 检查是否结束
    if (lastAssistant?.finish && ...) {
      break
    }

    // 调用 LLM.stream() 与 AI Provider 交互
  }
})
```

### 7. LLM.stream() - 与 AI Provider 交互

**文件**: `packages/opencode/src/session/llm.ts:46`

```typescript
export async function stream(input: StreamInput) {
  // 获取 Provider、Language、Config
  const [language, cfg, provider, auth] = await Promise.all([...])

  // 构建 system prompt
  const system = [...]

  // 解析 tools (工具定义)
  const tools = await resolveTools(input)

  // 调用 streamText() 与 AI Provider 通信
  return streamText({
    tools,
    // ...
  })
}
```

### 8. 工具执行

工具定义在 `/tools` 目录:

- ReadTool: 读取文件
- EditTool: 编辑文件
- BashTool: 执行 shell 命令
- TaskTool: 子任务执行
- WebSearchTool: 网络搜索

### 9. 事件发布

**文件**: `packages/opencode/src/session/message-v2.ts:460`

```typescript
Bus.publish("message.part.updated", part)
```

### 10. 前端接收事件

**文件**: `packages/app/src/context/global-sdk.tsx:146`

```typescript
for await (const event of events.stream) {
  const directory = event.directory ?? "global"
  const payload = event.payload
  // 发送到事件队列
  queue.push({ directory, payload })
  schedule()
}
```

### 11. 状态更新

**文件**: `packages/app/src/context/global-sync/event-reducer.ts:202`

```typescript
case "message.part.updated": {
  const part = event.properties.part
  // 更新 store 中的消息 parts
  input.setStore("part", part.messageID, [...])
  break
}
```

### 12. UI 渲染更新

- SolidJS 响应式更新
- 显示 AI 回复到 TUI

---

# 第三部分：TUI 渲染原理

## 四、核心技术栈

TUI 使用 **@opentui** 库来实现终端中的类 React 风格渲染：

| 包                | 作用                                                                 |
| ----------------- | -------------------------------------------------------------------- |
| `@opentui/solid`  | SolidJS 集成，提供 render、useKeyboard、useRenderer 等 hooks         |
| `@opentui/core`   | 核心类型和组件，如 BoxRenderable、TextareaRenderable、TextAttributes |
| `opentui-spinner` | 终端微调器组件                                                       |

---

## 五、@opentui 渲染原理

### 1. 架构概述

```
┌─────────────────────────────────────────────────────────────────────┐
│                        JSX/TSX 代码                                 │
│   <box> <text>Hello</text> </box>                                   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   @opentui/solid                                    │
│   - 将 JSX 转换为虚拟 DOM                                           │
│   - 管理组件状态和生命周期                                           │
│   - 提供响应式更新机制                                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   @opentui/core                                     │
│   - Renderable 接口定义                                             │
│   - 布局计算 (box, flex)                                           │
│   - 文本样式 (fg, bg, bold, italic)                                │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   CliRenderer (终端渲染器)                           │
│   - 将虚拟 DOM 渲染为 ANSI 转义序列                                 │
│   - 使用 VT100/VT220 转义码控制光标和样式                          │
│   - 差量更新，只渲染变化的部分                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        终端输出                                     │
│   \x1b[31mHello\x1b[0m                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. JSX 到终端的转换

#### 2.1 基础组件

```tsx
// 代码
<box flexDirection="row" gap={1}>
  <text fg={theme.success}>✓</text>
  <text>Hello World</text>
</box>

// 渲染为终端输出
✓ Hello World
```

#### 2.2 样式属性

| 属性        | 作用                |
| ----------- | ------------------- |
| `fg`        | 前景色 (foreground) |
| `bg`        | 背景色 (background) |
| `bold`      | 粗体                |
| `italic`    | 斜体                |
| `dim`       | 暗淡                |
| `underline` | 下划线              |

#### 2.3 布局组件

```tsx
// box 组件 - 类似 Flexbox
<box flexDirection="row" gap={1} justifyContent="space-between">
  <text>Left</text>
  <text>Right</text>
</box>

// text 组件 - 文本
<text fg="red" bold>Important</text>

// span 组件 - 行内文本
<text>
  Hello <span fg="green">World</span>
</text>

// 条件渲染
<Show when={condition}>
  <text>Shown when true</text>
</Show>

// 列表渲染
<For each={items}>
  {(item) => <text>{item}</text>}
</For>
```

### 3. SolidJS 响应式原理

#### 3.1 响应式数据

```typescript
import { createSignal, createMemo, createEffect } from "solid-js"

// 信号 - 响应式状态
const [count, setCount] = createSignal(0)

// 计算属性 - 基于信号的派生值
const doubled = createMemo(() => count() * 2)

// 副作用 - 响应式变更
createEffect(() => {
  console.log("count changed:", count())
})
```

#### 3.2 自动依赖追踪

当信号变化时，SolidJS 会：

1. 追踪哪些计算属性使用了该信号
2. 只更新依赖变化的最小单位
3. 不需要 Virtual DOM 的全量 Diff

#### 3.3 示例：响应式 UI

```tsx
function Counter() {
  const [count, setCount] = createSignal(0)

  return (
    <box>
      <text>Count: {count()}</text>
      <button onClick={() => setCount((c) => c + 1)}>Increment</button>
    </box>
  )
}
```

---

## 六、OpenCode TUI 渲染流程

### 1. 入口：render() 函数

```typescript
// app.tsx
import { render } from "@opentui/solid"

render(
  () => <App />,
  {
    targetFps: 60,
    exitOnCtrlC: false,
  }
)
```

### 2. 主应用组件

```tsx
// app.tsx
function App() {
  const route = useRoute()

  return (
    <Switch>
      <Match when={route.data.type === "home"}>
        <Home />
      </Match>
      <Match when={route.data.type === "session"}>
        <Session />
      </Match>
    </Switch>
  )
}
```

### 3. 路由系统

```typescript
// context/route.tsx
export type Route = { type: "home" } | { type: "session"; sessionID: string }

// 默认路由
const [store, setStore] = createStore<Route>({ type: "home" })

// 导航
function navigate(route: Route) {
  setStore(route)
}
```

### 4. Provider 层级

```tsx
// 嵌套的 Context Providers
<SDKProvider>
  {" "}
  // API 客户端
  <SyncProvider>
    {" "}
    // 状态同步
    <ThemeProvider>
      {" "}
      // 主题
      <LocalProvider>
        {" "}
        // 本地状态
        <KeybindProvider>
          {" "}
          // 键盘绑定
          <DialogProvider>
            {" "}
            // 对话框
            <RouteProvider>
              {" "}
              // 路由
              <App /> // 主应用
            </RouteProvider>
          </DialogProvider>
        </KeybindProvider>
      </LocalProvider>
    </ThemeProvider>
  </SyncProvider>
</SDKProvider>
```

### 5. 渲染配置

```typescript
// app.tsx
render(
  () => {
    /* App */
  },
  {
    targetFps: 60, // 60 FPS 渲染
    gatherStats: false,
    exitOnCtrlC: false,
    useKittyKeyboard: {},
    autoFocus: false,
    openConsoleOnError: false,
  },
)
```

---

## 七、渲染组件示例

### 1. 首页 (Home)

```tsx
// routes/home.tsx
export function Home() {
  const sync = useSync()
  const theme = useTheme()

  return (
    <box flexDirection="column" height="100%">
      {/* Logo */}
      <Logo />

      {/* MCP 状态 */}
      <Show when={mcpCount() > 0}>
        <box>
          <text fg={theme.success}>●</text>
          <text>{mcpCount()} MCP servers</text>
        </box>
      </Show>

      {/* 输入框 */}
      <Prompt />

      {/* 会话列表 */}
      <SessionList />
    </box>
  )
}
```

### 2. 输入框组件 (Prompt)

```tsx
// component/prompt/index.tsx
export function Prompt() {
  const [text, setText] = createSignal("")

  return (
    <box>
      <Textarea value={text()} onChange={setText} placeholder="Ask anything..." />
    </box>
  )
}
```

### 3. 会话页面 (Session)

```tsx
// routes/session/index.tsx
export function Session() {
  return (
    <box flexDirection="column" height="100%">
      <Header />
      <MessageList flexGrow={1} />
      <Prompt />
      <Footer />
    </box>
  )
}
```

---

## 八、事件处理

### 1. 键盘事件

```typescript
import { useKeyboard } from "@opentui/solid"

function Component() {
  useKeyboard((event) => {
    // 处理键盘事件
    if (event.name === "enter") {
      // 回车键
    }
    if (event.ctrl && event.name === "c") {
      // Ctrl+C
    }
  })
}
```

### 2. 鼠标事件

```typescript
<text
  onClick={() => handleClick()}
  onMouseEnter={() => setHover(true)}
>
  Clickable
</text>
```

---

## 九、状态管理

### 1. 全局状态 (SyncProvider)

```typescript
const sync = useSync()

// 读取
const sessions = sync.data.session

// 写入
sync.set("session", sessionID, newSession)
```

### 2. 本地状态 (createSignal)

```typescript
const [value, setValue] = createSignal("default")
```

### 3. 持久化状态 (KVProvider)

```typescript
const kv = useKV()

// 读取
const hidden = kv.get("tips_hidden", false)

// 写入
kv.set("tips_hidden", true)
```

---

## 十、差量渲染

opentui 的 CliRenderer 支持差量更新：

1. **计算差异**：比较新旧虚拟 DOM
2. **最小更新**：只输出变化的终端代码
3. **光标控制**：使用 VT100 转义码移动光标

---

## 十一、关键代码文件

| 步骤       | 文件位置                                                    | 关键函数                                         |
| ---------- | ----------------------------------------------------------- | ------------------------------------------------ |
| 前端提交   | `packages/app/src/components/prompt-input/submit.ts`        | `handleSubmit()`, `client.session.promptAsync()` |
| API 路由   | `packages/opencode/src/server/routes/session.ts:771`        | `/prompt_async` 路由                             |
| 提示词处理 | `packages/opencode/src/session/prompt.ts:158`               | `SessionPrompt.prompt()`                         |
| AI 循环    | `packages/opencode/src/session/prompt.ts:274`               | `SessionPrompt.loop()`                           |
| LLM 调用   | `packages/opencode/src/session/llm.ts:46`                   | `LLM.stream()`                                   |
| 事件发布   | `packages/opencode/src/session/message-v2.ts:460`           | `Bus.publish("message.part.updated")`            |
| 前端接收   | `packages/app/src/context/global-sdk.tsx:146`               | SSE 流事件处理                                   |
| 状态更新   | `packages/app/src/context/global-sync/event-reducer.ts:202` | `message.part.updated` 处理                      |
| TUI 渲染   | `cli/cmd/tui/app.tsx`                                       | `render()` 函数                                  |
| 路由系统   | `cli/cmd/tui/context/route.tsx`                             | `RouteProvider`                                  |
| 首页组件   | `cli/cmd/tui/routes/home.tsx`                               | `<Home />`                                       |
| 会话组件   | `cli/cmd/tui/routes/session/index.tsx`                      | `<Session />`                                    |
| 输入组件   | `cli/cmd/tui/component/prompt/index.tsx`                    | `<Prompt />`                                     |

---

## 十二、可用的命令

| 命令                         | 说明                      |
| ---------------------------- | ------------------------- |
| `opencode` 或 `opencode tui` | 启动终端 TUI 界面         |
| `opencode web`               | 启动服务器并打开 Web 首页 |
| `opencode serve`             | 启动无头服务器            |
| `opencode run [dir]`         | 运行交互式会话            |

---

## 十三、总结

OpenCode TUI 架构分为三个主要部分：

1. **命令执行层**：从终端输入到 TUI 启动
2. **提示词处理层**：从用户输入到 AI 响应
3. **渲染层**：使用 @opentui/solid 实现类 React 的终端 UI

**渲染核心**：

- **JSX 语法**：使用 `<box>`、`<text>`、`<span>` 等组件
- **SolidJS 响应式**：细粒度响应式更新，无需 VDOM Diff
- **VT100/ANSI 转义**：底层使用终端转义序列渲染
- **Provider 模式**：多层嵌套的 Context 管理
- **60 FPS 渲染**：高性能的终端渲染引擎

这使得开发者可以使用与现代 Web 开发相同的组件化方式来构建终端 UI。
