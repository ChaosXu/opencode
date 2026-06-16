# OpenCode TUI Command 加载与执行完整流程

> 分析时间: 2026-06-15
> 范围: TUI 启动后 Command 的加载、初始化；用户输入 `/[command]` 后的执行、响应渲染到终端
>
> 路径以 `packages/` 为根。"LLM"指 OpenCode 服务端；"TUI"指 `packages/tui/`。
>
> 此前已分析过的内容（TUI 启动、prompt flow、worker/SSE/event 流）作为前置，本节聚焦"command 维度的加载、显示、执行"。

---

## 一、阶段总览

```
TUI 启动后（详见 review/process/tui-startup.md）
  │
  ▼
[阶段 A: 服务端 Command Service 装配]  packages/opencode/src/command/index.ts
  │  per-project InstanceState.make(init)
  │  ├── 内置: init (PROMPT_INITIALIZE), review (PROMPT_REVIEW)
  │  ├── 配置: config.command[name]
  │  ├── MCP:  mcp.prompts() (mcp source)
  │  └── Skill: skill.all() (skill source, 名字冲突时跳过)
  │  → 一次性算出当前 project 可见的完整 command 字典
  │
  ▼
[阶段 B: TUI SyncProvider 拉取]  packages/tui/src/context/sync.tsx:430-507
  │  onMount 触发一批并行 HTTP 请求:
  │  ├── 阻塞批: provider / agent / config / session list
  │  └── 非阻塞批: command.list / lsp / mcp / formatter / provider.auth / vcs / workspace.sync
  │  → command 落在非阻塞批里，setStore("command", reconcile(x.data ?? []))
  │
  ▼
[阶段 C: Autocomplete 渲染]  packages/tui/src/component/prompt/autocomplete.tsx:437-464
  │  createMemo 从 sync.data.command 构造可选项:
  │  ├── 过滤 source === "skill"（skill 不当 slash 显示，避免和 command 冲突）
  │  ├── source === "mcp" 加 :mcp 后缀
  │  └── 与本地 slashes() 合并、按 display 排序、padEnd 对齐
  │
  ▼
[阶段 D: 用户输入 /]  prompt 文本框检测 store.visible === "/"
  │  → Autocomplete 弹出，按 fuzzysort 模糊匹配 + frecency 加权
  │  → 用户选中命令（Enter / Tab）→ 插入 "/<name> " 到 prompt
  │
  ▼
[阶段 E: 用户回车]  submitInner @ prompt/index.tsx:1065-1085
  │  检测首字符 "/" 且首词匹配 sync.data.command
  │  → sdk.client.session.command({ sessionID, command, arguments, agent, model, variant, parts })
  │  void 异步触发（响应通过 SSE 流回来）
  │
  ▼
[阶段 F: HTTP 路由]  POST /session/:sessionID/command
  │  SessionHttpApi.command @ handlers/session.ts:329-337
  │  → promptSvc.command({ ...payload, sessionID })
  │
  ▼
[阶段 G: SessionPrompt.command 渲染模板]  prompt.ts:1417-1542
  │  ├── commands.get("init") → Info (含模板字符串)
  │  ├── 选 agent / 替换 $1...$N 占位符 / 替换 $ARGUMENTS
  │  ├── ConfigMarkdown.shell 内联 `` !`cmd` `` 替换
  │  ├── 选 model (cmd.model / input.model / current)
  │  ├── getModel() 校验
  │  ├── resolvePromptParts(template) → 文本/文件 parts
  │  ├── 判定 subtask 走 subtask part
  │  ├── plugin.trigger("command.execute.before", ...)
  │  └── prompt({ sessionID, messageID, model, agent, parts, variant })   ← 走普通 prompt 流
  │
  ▼
[阶段 H: 普通 prompt 流 → 走 runLoop]  prompt.ts:1134-1402
  │  → llm.stream → LLM 响应
  │  → SessionProcessor.handleEvent 写 part + 发 V2 事件
  │  → EventV2Bridge → GlobalBus → Worker Rpc.emit → TUI SDKProvider SSE
  │
  ▼
[阶段 I: TUI 渲染响应]  routes/session/index.tsx
  │  SyncProvider 更新 store
  │  <For each={messages()}> 重渲
  │  文本/工具调用/skill 加载/AGENTS.md 写文件 全部实时显示
  │
  ▼
[阶段 J: Command.Event.Executed 落地]  prompt.ts:1535
        publish(Command.Event.Executed, { name, sessionID, arguments, messageID })
        Project.initState 监听: 若是 init command 且匹配当前 project dir
        → Project.Service.setInitialized → 写 ProjectTable.time_initialized
```

---

## 二、阶段 A — 服务端 Command Service 装配

**文件**: `packages/opencode/src/command/index.ts:66-181`

### 2.1 接口

```ts
export const Info = Schema.Struct({
  name: Schema.String,
  description: Schema.optional(Schema.String),
  agent: Schema.optional(Schema.String),
  model: Schema.optional(Schema.String),
  source: Schema.optional(Schema.Literals(["command", "mcp", "skill"])),
  template: Schema.Unknown,                  // 实际是 Promise<string> | string
  subtask: Schema.optional(Schema.Boolean),
  hints: Schema.Array(Schema.String),
})

export interface Interface {
  readonly get: (name: string) => Effect.Effect<Info | undefined>
  readonly list: () => Effect.Effect<Info[]>
}
```

`Info.hints` 由 `hints(template)` 自动从模板字符串抽出 `$1`/`$2`/.../`$ARGUMENTS` 占位符，便于 SDK 知道命令接受几个位置参数（`command/index.ts:44-52`）。

### 2.2 四种 command 来源

`Command.layer` 装配时，遍历（`command/index.ts:73-158`）：

| 来源 | 行号 | 模板来源 | `source` 字段 |
| ---- | ---- | -------- | ------------ |
| **内置** `init` (PROMPT_INITIALIZE) | `:78-86` | `command/template/initialize.txt` + `${path}` → `ctx.worktree` | `"command"` |
| **内置** `review` (PROMPT_REVIEW) | `:87-96` | `command/template/review.txt` + `${path}` → `ctx.worktree`；`subtask: true` | `"command"` |
| **config** `cfg.command[name]` | `:98-111` | `command.template` 字段；agent/model/subtask/description 透传 | `"command"` |
| **MCP prompts** | `:113-140` | `mcp.getPrompt(...)` 拉取并 join 文本；用 `EffectBridge.promise` 包成异步模板 | `"mcp"` |
| **Skill** | `:142-153` | `item.content`（已包含 `AGENTS.md` 等的 markdown 正文）| `"skill"` |

> 关键：内置 `init` 和 `review` 用的 `${path}` 占位符在 `get template()` getter 里**懒替换**，每次 `commands.get("init")` 都会基于当前 `ctx.worktree` 重新生成 — 项目切换会自动跟着切。
>
> `source === "skill"` 的 command 通常用不到（因为同名的 skill 本身就能被模型用 `skill` 工具调），所以 TUI Autocomplete 主动把它们过滤掉（见阶段 C）。

### 2.3 缓存策略

```ts
const state = yield* InstanceState.make<State>((ctx) => init(ctx))
```

`InstanceState.make` 用 `ScopedCache` — **每个 project directory 一份**。同一 project 多次访问共享 state，project 切换 / dispose 会清理。

`Skill`/`MCP`/`Config` service 的 state 变化不会自动让 `Command.state` 失效（MCP prompts、skills 列表是动态的）— 这是**潜在的一致性窗口**，但实践中命令列表在 TUI 启动时拉一次就够了。

### 2.4 `list` 输出

```ts
const list = Effect.fn("Command.list")(function* () {
  const s = yield* InstanceState.get(state)
  return Object.values(s.commands)   // 字典 → 数组，保留插入顺序
})
```

返回的就是 `Info[]` 数组，TUI 端不排序（Autocomplete 内自己 sort by display name）。

---

## 三、阶段 B — TUI SyncProvider 拉取

**文件**: `packages/tui/src/context/sync.tsx`

`SyncProvider` 的 `onMount`（`sync.tsx:430-507`）做了两批并行 HTTP 请求：

### 3.1 阻塞批（影响 ready 状态）

```ts
return Promise.all([
  providersResponse,        // GET /config/providers
  providerListResponse,     // GET /config/providers (catalog)
  consoleStateResponse,     // GET /config/console-state
  agentsResponse,           // GET /agent
  configResponse,           // GET /config
  ...(sessionListResponse ? [sessionListResponse] : []),  // GET /session
])
```

这些是渲染首页所必需的。

### 3.2 非阻塞批（在阻塞批完成后触发，不阻塞首屏）

```ts
void Promise.all([
  ...(args.continue ? [] : [sessionListPromise.then((sessions) => setStore("session", reconcile(sessions)))]),
  consoleStatePromise.then(...),
  sdk.client.command.list({ workspace }).then((x) => setStore("command", reconcile(x.data ?? []))),  // ← 关键
  sdk.client.lsp.status({ workspace }).then(...),
  sdk.client.mcp.status({ workspace }).then(...),
  sdk.client.experimental.resource.list({ workspace }).then(...),
  sdk.client.formatter.status({ workspace }).then(...),
  sdk.client.session.status({ workspace }).then(...),
  sdk.client.provider.auth({ workspace }).then(...),
  sdk.client.vcs.get({ workspace }).then(...),
  project.workspace.sync(),
])
.then(() => {
  setStore("status", "complete")
})
```

`status` 从 `loading` → `partial`（阻塞批完）→ `complete`（非阻塞批完）。`command` 在 `partial` 阶段已经到，但用户可能已经能开始输入 — 这意味着**用户在 command 列表到达前按 `/`，Autocomplete 里命令会暂时不显示**。这是已知设计取舍。

> 注：非阻塞批其实很轻量（命令列表通常只有几条），可能几毫秒就到。实际体验上一般无感。

### 3.3 走 SDK → HTTP

`GET /command`（路径定义 `groups/instance.ts:51`，handler `handlers/instance.ts:76-78`）：

```ts
const getCommand = Effect.fn("InstanceHttpApi.command")(function* () {
  return yield* command.list()
})
```

`command` 是从 `InstanceState` 拿到的 `Command.Service`（`handlers/instance.ts:21` 附近 yield 出来）。返回 `Info[]` 直接进 OpenAPI schema 序列化为 JSON。

`workspace` query 参数是 `WorkspaceRoutingQuery`，决定 InstanceState 路由到哪个 project（详情见 `Location` / `Workspace` 概念，参见 `CONTEXT.md` "Location-scoped services"）。

---

## 四、阶段 C — Autocomplete 构造命令可选项

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:437-464`

```ts
const commands = createMemo((): AutocompleteOption[] => {
  const results: AutocompleteOption[] = [...slashes()]   // 本地命令面板里的 slash（如 prompt.skills）

  for (const serverCommand of sync.data.command) {
    if (serverCommand.source === "skill") continue       // 过滤掉 skill source
    const label = serverCommand.source === "mcp" ? ":mcp" : ""
    results.push({
      display: "/" + serverCommand.name + label,
      description: serverCommand.description,
      onSelect: () => {
        const newText = "/" + serverCommand.name + " "
        const cursor = props.input().logicalCursor
        props.input().deleteRange(0, 0, cursor.row, cursor.col)
        props.input().insertText(newText)
        props.input().cursorOffset = Bun.stringWidth(newText)
      },
    })
  }

  results.sort((a, b) => a.display.localeCompare(b.display))

  const max = firstBy(results, [(x) => x.display.length, "desc"])?.display.length
  if (!max) return results
  return results.map((item) => ({
    ...item,
    display: item.display.padEnd(max + 2),
  }))
})
```

`AutocompleteOption.onSelect` 不直接提交 — 它只把 `"/<name> "` 文本插到 prompt input 里，**用户继续输入 arguments 然后回车**才会走 submitInner。

### 4.1 选项 vs 选项的视觉

`Autocomplete` 在用户输入 `/` 触发时，把 `[...commandsValue]` 作为 `nonFileOptions`（`autocomplete.tsx:482`）。`options()` memo 做：

- 无 `searchValue`：直接返回 `[...nonFileOptions, ...fileOptions]`
- 有 `searchValue`：fuzzysort 模糊匹配 + frecency 加权
  - `keys`: `display`、`description`（slash 时）、`aliases`
  - 分数 *=2 偏向"display starts with visible+search"
  - 加 frecency 偏好常用项
- 取 top 10

### 4.2 MCP / Skill command 区别

- MCP source 的 display 会加 `:mcp` 后缀（`/commit:mcp`），UI 上一眼能区分
- Skill source 被直接过滤（用户想用 skill 走 `prompt.skills` 命令面板里的"Skills"或 `/skill` 的 tool call 路径，不在 Autocomplete 露）

---

## 五、阶段 D — 用户输入 `/`

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:466-514` + 调用方

```
用户敲下 "/"
  ↓
[1] opentui 触发 prompt 的 onInput
  ↓
[2] Autocomplete.onInput(value)  (refs/autocomplete 调用)
  ↓
[3] 检测到 trigger = "/" 且当前 token 是 "/"
     setStore("visible", "/")
  ↓
[4] Autocomplete 弹窗渲染 options()
  - commandsValue 来自 sync.data.command
  - 文件选项在 "/" trigger 下隐藏
  - 模糊匹配输入
  ↓
[5] 用户 ↑↓ 选 / Tab 选 / Enter 选 → onSelect 注入 "/<name> "
     或用户继续打字（fuzzysort 实时过滤）
```

如果用户在 Autocomplete 弹出时**直接打命令名**（如 `/init`），fuzzysort 实时过滤后第一项会高亮，回车走 onSelect。另一种场景：**Autocomplete 还没来得及弹出**（command 列表还没到，status=partial）用户手敲 `/init ` 再回车 — 同样能提交（见阶段 E），因为检测逻辑只看 `sync.data.command.some(...)`，不依赖 Autocomplete UI 是否已渲染。

---

## 六、阶段 E — submitInner 检测 slash 命令

**文件**: `packages/tui/src/component/prompt/index.tsx:1065-1085`

```ts
} else if (
  inputText.startsWith("/") &&
  sync.data.command.some((x) => x.name === inputText.split("\n")[0].split(" ")[0].slice(1))
) {
  move.startSubmit()
  // Parse command from first line, preserve multi-line content in arguments
  const firstLineEnd = inputText.indexOf("\n")
  const firstLine = firstLineEnd === -1 ? inputText : inputText.slice(0, firstLineEnd)
  const [command, ...firstLineArgs] = firstLine.split(" ")
  const restOfInput = firstLineEnd === -1 ? "" : inputText.slice(firstLineEnd + 1)
  const args = firstLineArgs.join(" ") + (restOfInput ? "\n" + restOfInput : "")

  void sdk.client.session.command({
    sessionID,
    command: command.slice(1),
    arguments: args,
    agent: agent.name,
    model: `${selectedModel.providerID}/${selectedModel.modelID}`,
    variant,
    parts: nonTextParts.filter((x) => x.type === "file"),
  })
}
```

**关键行为**：

1. **检测**：首字符 `/` + 首行首词（在 `sync.data.command` 里能查到）
2. **拆 command vs arguments**：
   - 第一个空格之前是 command name
   - 第一个空格之后是 arguments（多行也合并进去；空行除外）
3. **`command.slice(1)` 去掉前导 `/`**
4. **`void` 异步触发**：不 await — 响应通过 SSE 事件流回来（参见 `review/process/tui-prompt-flow.md`）
5. **不传 messageID**（服务端自动分配）
6. **支持 `args = "/focus on backend"` 这种形态** → `arguments: "focus on backend"` → 模板里 `$ARGUMENTS` 替换

如果首行首词**不**在 `sync.data.command` 里（用户手敲了一个不存在的 `/foo`），**不会进这个分支**，落到普通 prompt 分支（`else`），整段 `/foo bar` 当文本发给模型解释 — 友好降级。

---

## 七、阶段 F — HTTP 路由

**文件**: `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts:329-337`

```ts
const command = Effect.fn("SessionHttpApi.command")(function* (ctx) {
  yield* requireSession(ctx.params.sessionID)
  return yield* promptSvc
    .command({ ...ctx.payload, sessionID: ctx.params.sessionID })
    .pipe(Effect.mapError(() => new HttpApiError.BadRequest({})))
})
```

- 路径 `POST /session/:sessionID/command`（`groups/session.ts:97`）
- 校验 session 存在
- 调 `SessionPrompt.Service.command(...)` — **同步执行整个 prompt 流并返回最终 assistant message**（不是异步模式，所以不像 `promptAsync` 那样立即 204）
- 业务错误（命令未找到、agent 找不到等）→ 400 BadRequest

注意区别：

| 端点 | 模式 | 返回 |
| ---- | ---- | ---- |
| `POST /session/:id/prompt` | 同步流 | Stream<JSON>（SSE-like） |
| `POST /session/:id/prompt_async` | 异步 | 204 No Content，结果走 SSE |
| `POST /session/:id/command` | **同步** | 最终 assistant message |

command 端点用同步模式 — 跟 `prompt` 一样流出来。但因为 TUI 是 `void` 调，实际渲染依然靠 `GlobalBus` SSE 推过来的事件。

---

## 八、阶段 G — SessionPrompt.command 模板渲染

**文件**: `packages/opencode/src/session/prompt.ts:1417-1542`

8 个关键步骤：

| # | 步骤 | 行号 | 行为 |
| - | ---- | ---- | ---- |
| 1 | log + 查 command | `:1418-1430` | `commands.get("init")` 拿 `Info`；缺失 → 发 error event + 抛错 |
| 2 | 选 agent | `:1431, :1484` | `cmd.agent ?? input.agent ?? defaultInfo` |
| 3 | 解析 args → 替换占位符 | `:1433-1456` | `$1`/`$2`/.../ 顺序替换；`$ARGUMENTS` 全量替换；没占位符则 args 追加到模板末尾（换行隔开）|
| 4 | 内联 shell 命令 | `:1458-1469` | `ConfigMarkdown.shell(template)` 抽 `` !`cmd` ``，并行执行并替换为 stdout |
| 5 | 选 model | `:1472-1480` | `cmd.model > cmd.agent.model > input.model > currentModel(sessionID)` |
| 6 | 校验 model | `:1482` | `getModel(providerID, modelID, sessionID)` 失败 → 发 error event + 抛错 |
| 7 | 拼 parts + 钩子 | `:1493-1525` | `resolvePromptParts(template)` → 去重 file parts → 判定 subtask → 拼最终 parts；`plugin.trigger("command.execute.before", ...)` 允许改 parts |
| 8 | 走 prompt | `:1527-1540` | `prompt({...})` 走普通 prompt 流；完成后 `publish(Command.Event.Executed, ...)` |

> 注意：模板是**作为 user message 的 parts 传**给 LLM 的（不是塞 system prompt）。模型看到的是"用户发来一段很长的指示"，自然知道按指示执行。

### 8.1 PROMPT_INITIALIZE 实例

`packages/opencode/src/command/template/initialize.txt:1-66`：

```
Create or update `AGENTS.md` for this repository.

The goal is a compact instruction file that helps future OpenCode sessions avoid mistakes and ramp up quickly. Every line should answer: "Would an agent likely miss this without help?" If not, leave it out.

User-provided focus or constraints (honor these):
$ARGUMENTS
... (66 行) ...
```

`/init` 提交时，假设用户输入 `/init focus on backend`，则：

- `command = "init"`
- `arguments = "focus on backend"`
- `template` = PROMPT_INITIALIZE 内容，`$ARGUMENTS` 被替换为 `focus on backend`，`${path}` 在 getter 里已被替换为 worktree 路径
- 渲染后作为 user message 的 text part 发给 LLM

### 8.2 Command.Event.Executed 落地

`prompt.ts:1535-1540`：

```ts
yield* events.publish(Command.Event.Executed, {
  name: input.command,
  sessionID: input.sessionID,
  arguments: input.arguments,
  messageID: result.info.id,
})
```

注意 `result` 是 `prompt()` 的返回 — 即最终 assistant message。这个 `messageID` 字段是给下游用的（Project.initState 之类）。

---

## 九、阶段 H — 普通 prompt 流

`SessionPrompt.command` 调 `prompt()` 后就走与普通用户消息**完全相同**的链路（详见 `review/process/tui-prompt-flow.md` 阶段 5-8）：

```
prompt()
  ↓
createUserMessage + ensureAssistantMessage
  ↓
runLoop (prompt.ts:1134-1402)
  ├── step++
  ├── processor.create
  ├── handle.process({ system, messages, tools, model, ... })
  └── llm.stream(streamInput)
       ↓
     LLMClient.stream → compile → route.streamPrepared
     走 HTTP POST + SSE 流
     协议解码为 LLMEvent 流
  ↓
SessionProcessor.handleEvent (processor.ts:371-844)
  ├── text-delta → updatePartDelta + publish(Text.Delta)
  ├── tool-call → updateToolCall + publish(Tool.Called)
  ├── tool-result → completeToolCall + publish(Tool.Success)
  └── step-finish → 累计 tokens + publish(Step.Ended)
  ↓
EventV2Bridge.publish → 落 event store + emit GlobalBus
  ↓
Worker GlobalBus.on("event", e => Rpc.emit("global.event", e))
  ↓
TUI SDKProvider SSE 收到
```

这一段**与 `/init` 走 /init 走 `/foo` 走普通消息**完全无差别。`/init` 只是"用模板填好 user message 内容"的一次性预填动作。

---

## 十、阶段 I — TUI 渲染响应

**文件**: `packages/tui/src/context/sync.tsx`（事件 → store 转换） + `packages/tui/src/routes/session/index.tsx`（渲染）

### 10.1 SyncProvider 收到事件（`sync.tsx:154-426`）

`handleEvent`（位于 `:154-426` 一带）switch 事件类型：

| 事件 | 同步动作（`sync.tsx`）|
| ---- | ------------------ |
| `message.updated`（`:298-336`）| `setStore("message", sessionID, idx, reconcile(msg))` |
| `message.part.updated`（`:353-373`）| `setStore("part", messageID, idx, reconcile(part))` |
| `message.part.delta`（`:375-392`）| `produce(d => d[idx][field] = (existing ?? "") + delta)` — 追加 |
| `message.removed` / `message.part.removed` | splice |
| `session.status`（`:293`）| 标 busy/idle/retry |
| `session.diff` / `todo.updated` | 标 diff/todo |
| `lsp.updated` / `mcp.*` | 标 lsp/mcp |
| `permission.asked` / `permission.replied` | 触发 TUI 弹窗 |
| `question.asked` / `question.replied` / `question.rejected` | 触发 TUI 弹窗 |

**`Command.Event.Executed`** 没有专门的 case — 但它**也走**同一套 `message.updated` 链路：因为 `command()` 内部调 `prompt()`，`prompt()` 内部会写 user message + assistant message（loop 结束返回 `lastAssistant`），所以**正常的 `message.updated` / `message.part.delta` 事件会被触发**，TUI 不需要特别区分"是命令触发还是普通 prompt"。

### 10.2 Session 路由渲染

`routes/session/index.tsx:207`：

```ts
const messages = createMemo(() => sync.data.message[route.sessionID] ?? [])
```

`<For each={messages()}>` 渲染每条 message（`:1188-1480` 一带）。每条 message 内部按 `parts[]` 渲染：

- `text` part → 直接 `<text>{part.text}</text>`
- `tool` part → `<ToolPart ...>` 卡片（status: running → completed → 可能附带 `<skill_content>`）
- `reasoning` part → `<ReasoningPart>`
- 元数据 part（`step-start` / `step-finish` / `patch` / `snapshot`）→ 折叠/展开

**Solid 响应式 + opentui renderer**：

- `part.text` 是 store 字段，访问触发依赖收集
- `setStore("part", messageID, idx, ...)` → Solid 标记脏 → opentui 在下一帧重绘
- 文本追加在终端上是**字符级 stream**（因为是 `text-delta` 事件 → `updatePartDelta` 追加）

### 10.3 `/init` 特殊体验

`/init` 跑起来时，用户会看到：

1. **用户消息**：文本里显示 `Create or update AGENTS.md...`（66 行 PROMPT_INITIALIZE 内容）— 这是 `prompt()` 写下的 user message
2. **assistant 消息**：可能是空（模型立刻开始调工具，没有直接说人话）
3. **一连串 tool 卡片**：
   - `read` / `glob` / `grep` — 调查项目
   - `write` / `edit` — 写或改 AGENTS.md
   - 每次工具调用都有完整 `tool-input-delta` 流式参数 + `tool-result` 折叠
4. **最后 finish**：assistant 输出 "I've created AGENTS.md at..." 之类的总结
5. **status → idle**：`session.status` 变 idle
6. **路由不变**：仍然在 session 路由，只是 messages 多了一条 assistant message

---

## 十一、阶段 J — 副作用

`Command.Event.Executed` 事件被一个**唯一**的监听者捕获：

**文件**: `packages/opencode/src/project/project.ts:415-425`

```ts
const initState = yield* InstanceState.make(
  Effect.fn("Project.initState")(function* (ctx) {
    const unsubscribe = yield* events.listen((event) => {
      if (event.type !== Command.Event.Executed.type || event.location?.directory !== ctx.directory)
        return Effect.void
      const data = event.data as EventV2.Data<typeof Command.Event.Executed>
      return data.name === Command.Default.INIT ? setInitialized(ctx.project.id) : Effect.void
    })
    yield* Effect.addFinalizer(() => unsubscribe)
  }),
)
```

`Project.initState` 在每个 project 实例化时注册一次监听：

- 收到 `command.executed` 事件
- 且 `event.location.directory` 匹配当前 project dir
- 且 `event.data.name === "init"`

→ `Project.Service.setInitialized(ctx.project.id)`（`project.ts:406-413`）：

```ts
yield* db
  .update(ProjectTable)
  .set({ time_initialized: Date.now() })
  .where(eq(ProjectTable.id, id))
  .run()
  .pipe(Effect.orDie)
```

把 `ProjectTable.time_initialized` 设为 `Date.now()` — 一个**审计戳**，不直接影响功能。

---

## 十二、整体可改进点（注意到的）

1. **Command 列表不在 sync store 增量更新**：`setStore("command", reconcile(x.data ?? []))` 只在 `onMount` 那一轮跑一次。如果用户中途切了 agent 触发 `command.execute.before` 钩子改了 `cmd.agent`，`commands` 列表不会反映 — 不过 `cmd.agent` 是单次调用的参数，不影响列表本身。
2. **新增 skill 不会触发 command 列表重拉**：因为 `command.service` 是 `InstanceState.make<State>`（缓存的），TUI 也只在 bootstrap 拉一次。如果运行时新增 skill（通过 file watcher 之类），TUI 不会自动看到 — **当前没有 hot-reload 路径**。
3. **Autocomplete 的 `commands` memo 重新计算**依赖于 `sync.data.command` 变化；目前是 startup-only，所以 memo 一次性计算后稳定。
4. **`status === "partial"` 时用户输入 `/`**：command 列表可能还没到，导致 autocomplete 弹窗暂时空。TUI 不会报错，等 `status === "complete"` 后 `commands` memo 自动重算、自动出项。

---

## 十三、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| 服务端 Command Service | `packages/opencode/src/command/index.ts:66-181` |
| `init` 模板 | `packages/opencode/src/command/template/initialize.txt:1-66` |
| `review` 模板 | `packages/opencode/src/command/template/review.txt` |
| HTTP `/command` 列表 | `packages/opencode/src/server/routes/instance/httpapi/groups/instance.ts:139-148` + `handlers/instance.ts:76-78` |
| HTTP `/session/:id/command` | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts:329-337` |
| SessionPrompt.command 渲染 | `packages/opencode/src/session/prompt.ts:1417-1542` |
| Command.Event.Executed | `packages/opencode/src/command/index.ts:18-28` |
| Project.initState | `packages/opencode/src/project/project.ts:415-425` |
| TUI SyncProvider 拉取 | `packages/tui/src/context/sync.tsx:430-507` (`:491` 关键) |
| TUI Autocomplete 构造 | `packages/tui/src/component/prompt/autocomplete.tsx:437-464` |
| TUI submitInner 检测 | `packages/tui/src/component/prompt/index.tsx:1065-1085` |
| TUI Session 路由渲染 | `packages/tui/src/routes/session/index.tsx:207, 1188` |
| TUI SyncProvider 事件处理 | `packages/tui/src/context/sync.tsx:154-426` |
| 通用 prompt 流（阶段 H） | 详见 `review/process/tui-prompt-flow.md` |
| TUI 启动链路 | 详见 `review/process/tui-startup.md` |
