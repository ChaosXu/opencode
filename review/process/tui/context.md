# OpenCode TUI Provider 组件分析

> 分析时间: 2026-02-28

---

## 一、render 函数调用

在 `packages/opencode/src/cli/cmd/tui/app.tsx` 中，`render` 函数接收两个参数：

```typescript
render(
  () => <App />,  // 根组件（第一个参数：JSX 函数）
  {               // 配置对象（第二个参数）
    targetFps: 60,
    gatherStats: false,
    exitOnCtrlC: false,
    useKittyKeyboard: {},
    autoFocus: false,
    openConsoleOnError: false,
    consoleOptions: {
      keyBindings: [{ name: "y", ctrl: true, action: "copy-selection" }],
      onCopySelection: (text) => { /* 复制到剪贴板 */ },
    },
  }
)
```

### render 参数说明

| 参数       | 类型                | 说明            |
| ---------- | ------------------- | --------------- |
| 第一个参数 | `() => JSX.Element` | 返回 JSX 的函数 |
| 第二个参数 | `RenderOptions`     | 渲染配置对象    |

### RenderOptions 配置项

| 配置项               | 类型      | 默认值 | 说明                 |
| -------------------- | --------- | ------ | -------------------- |
| `targetFps`          | `number`  | 60     | 目标渲染帧率         |
| `gatherStats`        | `boolean` | false  | 是否收集性能统计     |
| `exitOnCtrlC`        | `boolean` | false  | Ctrl+C 是否退出应用  |
| `useKittyKeyboard`   | `object`  | {}     | Kitty 键盘协议配置   |
| `autoFocus`          | `boolean` | false  | 是否自动聚焦         |
| `openConsoleOnError` | `boolean` | false  | 错误时是否打开控制台 |
| `consoleOptions`     | `object`  | -      | 控制台选项           |

---

## 二、Provider 组件层级（共 17 层）

```tsx
render(
  () => {
    return (
      <ErrorBoundary>
        <ArgsProvider>
          {" "}
          {/* 1. 命令行参数 */}
          <ExitProvider>
            {" "}
            {/* 2. 退出处理 */}
            <KVProvider>
              {" "}
              {/* 3. 键值存储 */}
              <ToastProvider>
                {" "}
                {/* 4. 提示消息 */}
                <RouteProvider>
                  {" "}
                  {/* 5. 路由管理 */}
                  <TuiConfigProvider>
                    {" "}
                    {/* 6. TUI 配置 */}
                    <SDKProvider>
                      {" "}
                      {/* 7. API 客户端 */}
                      <SyncProvider>
                        {" "}
                        {/* 8. 状态同步 */}
                        <ThemeProvider>
                          {" "}
                          {/* 9. 主题 */}
                          <LocalProvider>
                            {" "}
                            {/* 10. 本地状态 */}
                            <KeybindProvider>
                              {" "}
                              {/* 11. 键盘绑定 */}
                              <PromptStashProvider>
                                {" "}
                                {/* 12. 提示暂存 */}
                                <DialogProvider>
                                  {" "}
                                  {/* 13. 对话框 */}
                                  <CommandProvider>
                                    {" "}
                                    {/* 14. 命令 */}
                                    <FrecencyProvider>
                                      {" "}
                                      {/* 15. 频率排序 */}
                                      <PromptHistoryProvider>
                                        {" "}
                                        {/* 16. 历史 */}
                                        <PromptRefProvider>
                                          {" "}
                                          {/* 17. 提示引用 */}
                                          <App />
                                        </PromptRefProvider>
                                      </PromptHistoryProvider>
                                    </FrecencyProvider>
                                  </CommandProvider>
                                </DialogProvider>
                              </PromptStashProvider>
                            </KeybindProvider>
                          </LocalProvider>
                        </ThemeProvider>
                      </SyncProvider>
                    </SDKProvider>
                  </TuiConfigProvider>
                </RouteProvider>
              </ToastProvider>
            </KVProvider>
          </ExitProvider>
        </ArgsProvider>
      </ErrorBoundary>
    )
  },
  {
    /* 配置 */
  },
)
```

---

## 三、各 Provider 详解

### 1. ArgsProvider

```typescript
import { ArgsProvider, useArgs, type Args } from "./context/args"

// 命令行参数上下文
<ArgsProvider {...input.args}>
```

**作用**：提供命令行参数

**参数类型**：

```typescript
type Args = {
  continue?: boolean // -c, --continue
  sessionID?: string // -s, --session
  agent?: string // --agent
  fork?: boolean // --fork
  model?: string // -m, --model
}
```

**使用方式**：

```typescript
const args = useArgs()
if (args.agent) local.agent.set(args.agent)
```

---

### 2. ExitProvider

```typescript
import { ExitProvider, useExit } from "./context/exit"

// 退出处理
<ExitProvider onExit={onExit}>
```

**作用**：处理应用退出逻辑，包括信号处理

**使用方式**：

```typescript
const exit = useExit()
// 调用 exit() 退出应用
```

---

### 3. KVProvider

```typescript
import { KVProvider, useKV } from "./context/kv"

// 键值存储（持久化）
<KVProvider>
```

**作用**：持久化键值存储，保存用户偏好设置

**存储内容**：

- `tips_hidden` - 是否隐藏提示
- `terminal_title_enabled` - 是否启用终端标题
- `animations_enabled` - 是否启用动画
- `diff_wrap_mode` - diff 换行模式

**使用方式**：

```typescript
const kv = useKV()

// 读取
const hidden = kv.get("tips_hidden", false)

// 写入
kv.set("tips_hidden", true)
```

---

### 4. ToastProvider

```typescript
import { ToastProvider, useToast } from "./ui/toast"

// 提示消息
<ToastProvider>
```

**作用**：显示临时通知消息

**变体**：info, success, warning, error

**使用方式**：

```typescript
const toast = useToast()

toast.show({
  message: "操作成功",
  variant: "success",
  duration: 3000,
})
```

---

### 5. RouteProvider

```typescript
import { RouteProvider, useRoute } from "@tui/context/route"

// 路由管理
<RouteProvider>
```

**作用**：管理应用路由（首页/会话）

**路由类型**：

```typescript
type Route = { type: "home" } | { type: "session"; sessionID: string }
```

**使用方式**：

```typescript
const route = useRoute()

// 读取当前路由
route.data.type // "home" | "session"

// 导航
route.navigate({ type: "session", sessionID: "xxx" })
```

---

### 6. TuiConfigProvider

```typescript
import { TuiConfigProvider } from "./context/tui-config"

// TUI 配置
<TuiConfigProvider config={input.config}>
```

**作用**：提供 TUI 配置文件

**配置内容**：

- 键盘快捷键定义
- 主题配置
- 其他 TUI 相关设置

---

### 7. SDKProvider

```typescript
import { SDKProvider, useSDK } from "@tui/context/sdk"

// API 客户端
<SDKProvider
  url={input.url}
  directory={input.directory}
  fetch={input.fetch}
  headers={input.headers}
  events={input.events}
>
```

**作用**：提供 OpenCode API SDK 客户端

**提供内容**：

- `sdk.client.session` - 会话 API
- `sdk.client.provider` - Provider API
- `sdk.client.config` - 配置 API
- `sdk.event` - 事件订阅

**使用方式**：

```typescript
const sdk = useSDK()

// 创建会话
const session = await sdk.client.session.create()

// 发送提示
await sdk.client.session.promptAsync({ ... })
```

---

### 8. SyncProvider

```typescript
import { SyncProvider, useSync } from "@tui/context/sync"

// 全局状态同步
<SyncProvider>
```

**作用**：全局状态管理，响应后端事件

**同步内容**：

- `session` - 会话列表
- `message` - 消息
- `part` - 消息片段
- `provider` - AI Provider
- `mcp` - MCP 服务器

**使用方式**：

```typescript
const sync = useSync()

// 读取
const sessions = sync.data.session

// 写入
sync.set("session", sessionID, newSession)

// 监听状态
sync.status // "loading" | "ready" | "complete"
```

---

### 9. ThemeProvider

```typescript
import { ThemeProvider, useTheme } from "@tui/context/theme"

// 主题（深色/浅色）
<ThemeProvider mode={mode}>
```

**作用**：管理深色/浅色主题

**模式**：dark | light

**使用方式**：

```typescript
const { theme, mode, setMode } = useTheme()

// 主题颜色
theme.background // 背景色
theme.text // 文本色
theme.success // 成功色
theme.error // 错误色
```

---

### 10. LocalProvider

```typescript
import { LocalProvider, useLocal } from "@tui/context/local"

// 本地运行时状态
<LocalProvider>
```

**作用**：管理本地运行时状态

**内容**：

- 当前选中的模型
- 当前选中的 agent
- 会话状态

**使用方式**：

```typescript
const local = useLocal()

// 模型
local.model.current()
local.model.set({ providerID, modelID })

// Agent
local.agent.current()
local.agent.set("build")
```

---

### 11. KeybindProvider

```typescript
import { KeybindProvider, useKeybind } from "@tui/context/keybind"

// 键盘快捷键
<KeybindProvider>
```

**作用**：注册和处理键盘快捷键

**使用方式**：

```typescript
const keybind = useKeybind()

keybind.register("key_name", (handler) => {
  // 处理快捷键
})
```

---

### 12. PromptStashProvider

```typescript
import { PromptStashProvider, usePromptStash } from "./component/prompt/stash"

// 提示词暂存
<PromptStashProvider>
```

**作用**：临时保存和恢复输入内容

**使用场景**：切换会话时保存当前输入

**使用方式**：

```typescript
const stash = usePromptStash()

// 暂存
stash.save()

// 恢复
stash.restore()
```

---

### 13. DialogProvider

```typescript
import { DialogProvider, useDialog } from "@tui/ui/dialog"

// 对话框管理
<DialogProvider>
```

**作用**：管理模态对话框

**功能**：

- 显示确认对话框
- 显示提示对话框
- 显示选择对话框
- 替换当前对话框

**使用方式**：

```typescript
const dialog = useDialog()

// 显示对话框
dialog.show(<Component />)

// 替换对话框
dialog.replace(<Component />)

// 清除对话框
dialog.clear()
```

---

### 14. CommandProvider

```typescript
import { CommandProvider, useCommandDialog } from "@tui/component/dialog-command"

// 命令注册与触发
<CommandProvider>
```

**作用**：注册和触发 slash 命令

**注册命令**：

- `/new` - 新建会话
- `/connect` - 连接 Provider
- `/help` - 帮助
- `/models` - 切换模型
- `/agents` - 切换 Agent
- 等等

**使用方式**：

```typescript
const command = useCommandDialog()

command.register(() => [
  {
    title: "New session",
    slash: { name: "new" },
    onSelect: () => {
      /* ... */
    },
  },
])

// 触发命令
command.trigger("session.new")
```

---

### 15. FrecencyProvider

```typescript
import { FrecencyProvider, useFrecency } from "@tui/component/prompt/frecency"

// 频率+新鲜度排序
<FrecencyProvider>
```

**作用**：基于使用频率和最近使用时间智能排序

**算法**：Frecency（Frequency + Recency）

**使用场景**：

- 文件自动补全
- 命令建议
- 历史记录排序

---

### 16. PromptHistoryProvider

```typescript
import { PromptHistoryProvider, usePromptHistory } from "./component/prompt/history"

// 提示历史
<PromptHistoryProvider>
```

**作用**：管理输入历史（上下方向键遍历）

**使用方式**：

```typescript
const history = usePromptHistory()

// 添加到历史
history.add(prompt)

// 上一个
history.previous()

// 下一个
history.next()
```

---

### 17. PromptRefProvider

```typescript
import { PromptRefProvider, usePromptRef } from "./context/prompt"

// 提示引用
<PromptRefProvider>
```

**作用**：提供对当前输入框组件的引用

**使用场景**：在任意位置访问输入框内容

---

## 四、数据流向

```
┌─────────────────────────────────────────────────────────────────┐
│                     外层 → 内层（数据提供）                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ArgsProvider     →  命令行参数                                  │
│       ↓                                                       │
│  ExitProvider     →  退出函数                                   │
│       ↓                                                       │
│  KVProvider       →  持久化存储                                 │
│       ↓                                                       │
│  ToastProvider    →  提示函数                                  │
│       ↓                                                       │
│  RouteProvider    →  路由函数                                   │
│       ↓                                                       │
│  TuiConfigProvider → 配置                                      │
│       ↓                                                       │
│  SDKProvider      →  API 客户端                                │
│       ↓                                                       │
│  SyncProvider     →  同步状态                                  │
│       ↓                                                       │
│  ThemeProvider    →  主题                                      │
│       ↓                                                       │
│  LocalProvider    →  本地状态                                   │
│       ↓                                                       │
│  KeybindProvider  →  快捷键                                    │
│       ↓                                                       │
│  PromptStashProvider → 暂存                                    │
│       ↓                                                       │
│  DialogProvider   →  对话框                                    │
│       ↓                                                       │
│  CommandProvider  →  命令                                       │
│       ↓                                                       │
│  FrecencyProvider →  排序                                      │
│       ↓                                                       │
│  PromptHistoryProvider → 历史                                  │
│       ↓                                                       │
│  PromptRefProvider  → 输入框引用                                │
│       ↓                                                       │
│  App (根组件)    →  渲染 UI                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、事件流向

```
┌─────────────────────────────────────────────────────────────────┐
│                     内层 → 外层（事件传播）                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  App (用户操作)                                                 │
│       ↓                                                        │
│  PromptRefProvider → 获取输入                                   │
│       ↓                                                        │
│  CommandProvider  → 触发命令                                    │
│       ↓                                                        │
│  DialogProvider   → 显示对话框                                  │
│       ↓                                                        │
│  SDKProvider      → 发送 API 请求                              │
│       ↓                                                        │
│  SyncProvider     → 响应后端事件                                │
│       ↓                                                        │
│  ThemeProvider    → 更新主题                                    │
│       ↓                                                        │
│  RouteProvider    → 路由切换                                    │
│       ↓                                                        │
│  ToastProvider    → 显示提示                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、关键代码文件

| Provider              | 文件路径                                    |
| --------------------- | ------------------------------------------- |
| ArgsProvider          | `cli/cmd/tui/context/args.tsx`              |
| ExitProvider          | `cli/cmd/tui/context/exit.tsx`              |
| KVProvider            | `cli/cmd/tui/context/kv.tsx`                |
| ToastProvider         | `cli/cmd/tui/ui/toast.tsx`                  |
| RouteProvider         | `cli/cmd/tui/context/route.tsx`             |
| TuiConfigProvider     | `cli/cmd/tui/context/tui-config.tsx`        |
| SDKProvider           | `cli/cmd/tui/context/sdk.tsx`               |
| SyncProvider          | `cli/cmd/tui/context/sync.tsx`              |
| ThemeProvider         | `cli/cmd/tui/context/theme.tsx`             |
| LocalProvider         | `cli/cmd/tui/context/local.tsx`             |
| KeybindProvider       | `cli/cmd/tui/context/keybind.tsx`           |
| PromptStashProvider   | `cli/cmd/tui/component/prompt/stash.tsx`    |
| DialogProvider        | `cli/cmd/tui/ui/dialog.tsx`                 |
| CommandProvider       | `cli/cmd/tui/component/dialog-command.tsx`  |
| FrecencyProvider      | `cli/cmd/tui/component/prompt/frecency.tsx` |
| PromptHistoryProvider | `cli/cmd/tui/component/prompt/history.tsx`  |
| PromptRefProvider     | `cli/cmd/tui/context/prompt.tsx`            |

---

## 七、总结

OpenCode TUI 使用 **Provider 模式** 实现依赖注入：

1. **17 层嵌套**：每个 Provider 负责特定功能
2. **数据外→内**：外层提供数据/方法给内层使用
3. **事件内→外**：用户操作产生的事件逐层向上传递
4. **职责分离**：每个 Provider 只关心自己的职责

这种架构使得：

- 代码模块化，易于维护
- 组件之间解耦，不需层层传递 props
- 便于测试和替换
