# OpenCode TUI 执行流程分析

> 分析时间: 2026-02-28

> ⚠️ 详细渲染原理请参考 [render.md](./render.md)

---

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

1. **按键触发** - 当用户在输入框中按 `Enter` 键且不按 Shift 时（第 1123-1125 行）:

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
    onError(error) { ... },
    async experimental_repairToolCall(failed) { ... },
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
- 等等

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

## 四、流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. 前端 - 用户输入提示词 (prompt-input/submit.ts)                  │
│    - handleSubmit() 触发                                           │
│    - 构建 parts (文本、文件、附件等)                                │
│    - 创建乐观消息 (optimistic message)                              │
│    - 调用 client.session.promptAsync()                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. SDK 发送请求                                                     │
│    - POST /:sessionID/prompt_async                                  │
│    - 使用 SSE 流式响应                                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. 后端 API 路由 (server/routes/session.ts:771)                    │
│    - 验证 PromptInput Schema                                        │
│    - 调用 SessionPrompt.prompt()                                    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. SessionPrompt.prompt() (session/prompt.ts:158)                 │
│    - createUserMessage() - 创建用户消息                             │
│    - 调用 loop() 进入 AI 交互循环                                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. SessionPrompt.loop() (session/prompt.ts:274)                    │
│    - 获取模型和 agent 配置                                          │
│    - while 循环处理多轮对话                                         │
│    - 处理 subtask (子任务)                                          │
│    - 调用 LLM.stream() 与 AI Provider 交互                        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. LLM.stream() (session/llm.ts:46)                                │
│    - 获取 Provider、Language、Config                                │
│    - 构建 system prompt                                             │
│    - 解析 tools (工具定义)                                          │
│    - 调用 streamText() 与 AI Provider 通信                         │
│    - 处理工具调用 (Read、Edit、bash 等)                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 7. 工具执行 (Tool 定义在 /tools 目录)                               │
│    - ReadTool: 读取文件                                             │
│    - EditTool: 编辑文件                                             │
│    - BashTool: 执行 shell 命令                                      │
│    - TaskTool: 子任务执行                                           │
│    - ...                                                            │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 8. 事件发布 (session/message-v2.ts:460)                            │
│    - Bus.publish("message.part.updated", part)                     │
│    - 存储到数据库                                                   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 9. 前端接收事件 (context/global-sdk.tsx:146)                       │
│    - SSE 长连接接收事件                                             │
│    - EventReducer 处理事件                                          │
│    - 更新 global-sync store                                         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 10. UI 渲染更新                                                     │
│    - SolidJS 响应式更新                                             │
│    - 显示 AI 回复到 TUI                                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键代码文件

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

---

## 六、可用的命令

| 命令                         | 说明                      |
| ---------------------------- | ------------------------- |
| `opencode` 或 `opencode tui` | 启动终端 TUI 界面         |
| `opencode web`               | 启动服务器并打开 Web 首页 |
| `opencode serve`             | 启动无头服务器            |
| `opencode run [dir]`         | 运行交互式会话            |
