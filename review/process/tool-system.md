# OpenCode TUI 模式下 Tool 系统完整链路

> 分析时间: 2026-06-15
> 范围: TUI 启动后 tool 如何加载、注册、暴露给 LLM、LLM 调用 tool、结果回 TUI 渲染
>
> 路径以 `packages/opencode/src/tool/` 和 `packages/opencode/src/session/` 为根。TUI 端**不加载 tool** — server 加载 + 暴露给 LLM，TUI 端只**渲染** tool 调用结果。

---

## 一、阶段总览

```
[1] TUI 启动 (tui-startup.md) → AppLayer 装配，ToolRegistry.defaultLayer 在内
       │
       ▼
[2] 第一个 HTTP 请求进来 → InstanceContextMiddleware → InstanceStore.load
       │  → InstanceBootstrap.run
       │  → project.bootstrap 配置 + plugin.init() 完成
       │
       ▼
[3] 第一次有人需要 tool 列表 (runLoop 第一轮) → SessionTools.resolve
       │  → ToolRegistry.tools({ modelID, providerID, agent })
       │     → InstanceState.get(toolRegistryState)
       │        → ScopedCache miss → 调 init(ctx) (registry.ts:110-241)
       │           ├── 扫 ConfigPaths.directories() 找 tool/*.{js,ts}
       │           ├── dynamic import → fromPlugin() 转 Tool.Def
       │           ├── 合并 plugin.list() 里的 p.tool
       │           └── 装配 15 个内置 tool
       │           → 写进 cache
       │        → 返回 过滤后的 Def[]
       │     ← 缓存
       │  → 调 plugin.trigger("tool.definition", { toolID }, output)  (允许插件改 description/parameters)
       │  → 合并 MCP tools (mcp.tools())
       │  → 用 AI SDK 的 tool() 包装每个 tool (含 execute 闭包)
       │  → return tools: Record<string, AITool>
       │
       ▼
[4] handle.process({ system, messages, tools, model, ... })  (processor.ts:1336)
       │  ↓
       ▼
[5] llm.stream(streamInput)  (processor.ts:974)
       │  - LLMClient.stream (packages/llm/src/route/client.ts:367)
       │  - compile() 把 LLMRequest 编译为 provider-native body + HttpClientRequest
       │  - route.streamPrepared(prepared, ...)
       │  - transport.frames 走 HttpClient.execute (POST + SSE)
       │  - protocol.step 状态机把 SSE 帧 → LLMEvent
       │  - tools 字段被协议适配器编码为 provider 原生 (function calling / tool use)
       │
       ▼
[6] LLM 响应 → LLMEvent 流
       │  provider stream → handleEvent 切 case (processor.ts:371-844):
       │    tool-input-start  → ensureToolCall (建 tool part)
       │    tool-input-delta → 累加 raw input, publish Tool.Input.Delta
       │    tool-input-end   → 标 inputEnded, publish Tool.Input.Ended
       │    tool-call        → updateToolCall 固化 input, publish Tool.Called
       │    tool-result      → completeToolCall, publish Tool.Success | Tool.Failed
       │    step-finish      → 累计 tokens/cost, publish Step.Ended
       │
       ▼
[7] AI SDK 在 tool-call event 时自动调 tool.execute() (来自 tools.ts:74-114 的闭包)
       │  execute(args, options):
       │    plugin.trigger("tool.execute.before", {tool, sessionID, callID}, {args})
       │    ctx.ask(permission) 如果 tool 内部需要
       │    item.execute(decoded, ctx)  ← 真正的 tool 逻辑 (Tool.init 包装)
       │      → Schema.decode(args) 验参
       │      → 调 toolInfo.execute(decoded, ctx)
       │      → truncate.output (输出过大时截断)
       │    plugin.trigger("tool.execute.after", ..., output)
       │    return output
       │
       ▼
[8] tool result → handleEvent("tool-result") → completeToolCall + publish Tool.Success
       │  → V2 event → EventV2Bridge → GlobalBus → worker Rpc.emit → TUI SDKProvider SSE
       │  → SyncProvider 收到 Tool.Success → setStore("part", ...) → Session 路由重渲
       │  → TUI 显示 tool result 卡片
       │
       ▼
[9] runLoop 下一轮 → MessageV2.toModelMessagesEffect 把 tool_use + tool_result 装回 model messages
       │  → llm.stream 再次调用 (LLM 现在能看到自己刚调用的结果)
       │  → LLM 继续生成 (可能再调 tool 或给最终回复)
       │  → finish → runLoop break → return lastAssistant
```

---

## 二、Tool 定义与注册

### 2.1 Tool 接口（`packages/opencode/src/tool/tool.ts:55-77`）

```ts
export interface Def<Parameters, M> {
  id: string
  description: string
  parameters: Parameters          // Effect Schema decoder
  jsonSchema?: JSONSchema7       // 给 LLM 的 JSON Schema
  execute(args, ctx): Effect<ExecuteResult<M>>
  formatValidationError?(error): string
}

export interface Info<P, M> {
  id: string
  init: () => Effect<DefWithoutID<P, M>>    // 懒初始化
}
```

`Tool.define(id, init)` 返回 `Effect<Info>`（用 `Object.assign` 同时挂上 `id` 让 chain 友好）— `init` 里 yield `Skill.Service` / `Permission.Service` 等 service，组成完整 tool 定义。

### 2.2 Tool.init 包装（`tool.ts:99-149`）

```ts
function wrap(id, init, truncate, agents) {
  return () => Effect.gen(function* () {
    const toolInfo = typeof init === "function" ? { ...(yield* init()) } : { ...init }
    const decode = Schema.decodeUnknownEffect(toolInfo.parameters)
    const execute = toolInfo.execute
    toolInfo.execute = (args, ctx) => {
      return Effect.gen(function* () {
        const decoded = yield* decode(args).pipe(
          Effect.mapError((error) => new InvalidArgumentsError({ tool: id, detail: ... })),
        )
        const result = yield* execute(decoded, ctx)
        if (result.metadata.truncated !== undefined) return result
        const agent = yield* agents.get(ctx.agent)
        const truncated = yield* truncate.output(result.output, {}, agent)
        return { ...result, output: truncated.content, metadata: { ...result.metadata, truncated: truncated.truncated, ... } }
      }).pipe(Effect.orDie, Effect.withSpan("Tool.execute", { attributes: {...} }))
    }
    return toolInfo
  })
}
```

**职责**：
- **参数 schema 验证**（`decode(args)`）— 失败抛 `InvalidArgumentsError`（`tool.ts:24-34`），message 是个**给模型看的友好描述**
- **输出截断**（`truncate.output`）— 大输出截断 + 写盘 + metadata 标 `truncated: true`
- **tracing span**（`withSpan("Tool.execute", { attributes })`）— 配 OpenTelemetry

### 2.3 15 个内置 tool（`registry.ts:92-107, 198-214`）

```ts
yield* InvalidTool         // 无效
yield* TaskTool            // 派子 agent
yield* ReadTool            // 读文件
yield* QuestionTool        // 问用户
yield* TodoWriteTool       // 写 todo
yield* LspTool             // LSP (experimental)
yield* PlanExitTool        // plan 模式
yield* WebFetchTool        // HTTP fetch
yield* WebSearchTool       // 搜索
yield* ShellTool           // shell
yield* GlobTool            // glob 文件
yield* WriteTool           // 写文件
yield* EditTool            // edit
yield* GrepTool            // grep
yield* ApplyPatchTool      // apply_patch (gpt 专用)
yield* SkillTool           // 加载 skill
```

### 2.4 ToolRegistry 装配（`registry.ts:83-440`）

`ToolRegistry.layer` 在 `InstanceState.make<State>((ctx) => init(ctx))` 里 **per-project** 缓存。`init(ctx)` 步骤：

| 行号 | 步骤 |
| ---- | ---- |
| `:110-111` | `InstanceState.make<State>(Effect.fn("ToolRegistry.state")(ctx => ...))` |
| `:112` | 初始化 `custom: Tool.Def[] = []` |
| `:114-170` | `fromPlugin(id, def)` 工具：把 plugin tool 的 Zod / JSON Schema 归一，转成 `Tool.Def` |
| `:172-176` | `config.directories()` → 扫 `tool/*.{js,ts}` 文件（用 `Glob.scanSync`）|
| `:177-186` | `Effect.promise(() => import(pathToFileURL(match).href))` 动态加载每个 tool 文件 → `custom.push(fromPlugin(...))` |
| `:188-193` | 合并 `plugin.list()` 里 `p.tool` 的所有 tool |
| `:195` | `config.get()` 触发 config 加载 |
| `:196` | `questionEnabled = ["app", "cli", "desktop"].includes(flags.client) || flags.enableQuestionTool` |
| `:198-215` | `Effect.all({ ... })` 并行调 `Tool.init(tool)` 装配所有内置 tool |
| `:217-239` | 返回 `{ custom, builtin: [tool.invalid, ... 14 more], task, read }` |

### 2.5 ToolRegistry.tools 过滤（`registry.ts:267-307`）

```ts
const tools: Interface["tools"] = Effect.fn("ToolRegistry.tools")(function* (input) {
  const filtered = (yield* all()).filter((tool) => {
    if (tool.id === WebSearchTool.id) {
      return webSearchEnabled(input.providerID, { exa: flags.enableExa, parallel: flags.enableParallel })
    }
    const usePatch = input.modelID.includes("gpt-") && !input.modelID.includes("oss") && !input.modelID.includes("gpt-4")
    if (tool.id === ApplyPatchTool.id) return usePatch
    if (tool.id === EditTool.id || tool.id === WriteTool.id) return !usePatch
    return true
  })

  return yield* Effect.forEach(filtered, Effect.fnUntraced(function* (tool) {
    const output = { description: tool.description, parameters: tool.parameters, jsonSchema: tool.jsonSchema }
    yield* plugin.trigger("tool.definition", { toolID: tool.id }, output)   // 插件可改
    const jsonSchema = output.parameters === tool.parameters || output.jsonSchema !== tool.jsonSchema
      ? output.jsonSchema : undefined
    return {
      id: tool.id,
      description: [output.description, tool.id === TaskTool.id ? yield* describeTask(input.agent) : undefined]
        .filter(Boolean).join("\n"),
      parameters: output.parameters,
      jsonSchema,
    }
  }))
})
```

**过滤规则**：
- **WebSearch**：默认仅 opencode/exa/parallel provider 启用
- **Edit/Write vs ApplyPatch**：GPT 模型（不带 oss/gpt-4）走 `apply_patch`，否则走 edit + write

**插件钩子**：`plugin.trigger("tool.definition", { toolID }, output)` — 插件可改 `output.description/parameters/jsonSchema`，最终被 LLM 看到

---

## 三、SessionTools.resolve：把 Tool 暴露给 LLM

**文件**: `packages/opencode/src/session/tools.ts:24-205`

**调用点**: `SessionPrompt.runLoop`（`prompt.ts:1279-1293`）：

```ts
const tools = yield* SessionTools.resolve({
  agent, session, model,
  processor: handle,
  bypassAgentCheck, messages, promptOps,
}).pipe(Effect.provideService(Plugin.Service, plugin), ...)

const result = yield* handle.process({
  user, agent, permission, sessionID, parentSessionID,
  system, messages, tools, model,    // ← tools 传给 processor
  toolChoice: format.type === "json_schema" ? "required" : undefined,
})
```

`SessionTools.resolve` 干了 3 件事：

1. **构造 `Tool.Context` 闭包**（`tools.ts:41-72`）：
   - `metadata` 回调 → 调 `processor.updateToolCall` 改 part state 为 "running"
   - `ask` 回调 → 调 `Permission.ask` 发 permission.asked 事件

2. **遍历 `registry.tools({...})`**（`tools.ts:74-115`）：
   - 用 `ai` SDK 的 `tool({ description, inputSchema: jsonSchema, execute })` 包装
   - `execute` 闭包：
     - `plugin.trigger("tool.execute.before", ...)` 改 args
     - `item.execute(args, ctx)` 调真实 tool（来自 `Tool.init` 包装）
     - `plugin.trigger("tool.execute.after", ..., output)` 改 output
     - abort 时调 `processor.completeToolCall` 收尾
   - 把 part.attachments 重设 id/sessionID/messageID

3. **遍历 `mcp.tools()`**（`tools.ts:117-202`）：
   - 把 MCP tool 包装成同样形态
   - 额外 `ctx.ask({ permission: key, ... })`（MCP tool 一律 ask）
   - 转换 MCP content → SessionV1 attachment（image / resource → file part）

**结果**: `tools: Record<string, AITool>` 返回给 runLoop，传给 `handle.process`。

---

## 四、LLM 调用 & 协议编码

**`llm.stream(streamInput)`**（`processor.ts:974`）：

```ts
const stream = llm.stream(streamInput)

yield* stream.pipe(
  Stream.tap((event) => handleEvent(event)),   // 每个 LLMEvent 走 handleEvent
  Stream.takeUntil(() => ctx.needsCompaction),
  Stream.runDrain,
)
```

`llm.stream` 链路（详见 `review/process/tui-prompt-flow.md` 阶段 7）：

```
LLMClient.stream
  → streamRequestWith
    → compile (request → provider-native body + HttpClientRequest)
    → route.streamPrepared
      → transport.frames  HttpClient.execute (POST + SSE)
      → protocol.step 状态机  →  LLMEvent 流
```

**关键点**：
- `streamInput` 包含 `tools: Record<string, AITool>` 字段
- `compile()` 调 `route.body.from(request)` 构造 provider-native body（**协议适配器**自动把 AI SDK `AITool` 编码为 OpenAI `function calling` / Anthropic `tool use` / Gemini `functionDeclarations` 等）
- 各 provider 协议模块（`packages/llm/src/protocols/*.ts`）负责编码

---

## 五、LLMEvent 处理（`processor.ts:371-844`）

LLM 一次完整 tool 调用的事件流：

| LLMEvent | 行号 | 行为 | 副作用 |
| -------- | ---- | ---- | ------ |
| `step-start` | `:676` | 一轮 LLM 开始 | 记 snapshot，写 part |
| `tool-input-start` | `:427` | tool 调开始 | `ensureToolCall` 建 tool part（pending 状态）|
| `tool-input-delta` | `:434` | tool input 流式 | 累加 raw text，发 `Tool.Input.Delta` |
| `tool-input-end` | `:451` | tool input 结束 | 标 `inputEnded`，发 `Tool.Input.Ended` |
| `tool-call` | `:468` | 完整 tool call | `updateToolCall` 固化 input，发 `Tool.Called`；doom loop 检测 |
| **`AI SDK auto-execute`** | - | AI SDK 看到 `tool-call` event 后**自动**调 `tool.execute()` 闭包（这是 ai 库内置行为，**不**在 opencode 代码里）| 实际跑 tool |
| `tool-result` | `:549` | tool 跑完 | `completeToolCall` 写 part，发 `Tool.Success`/`Tool.Failed` |
| `step-finish` | `:693` | 一轮 LLM 结束 | 累计 tokens/cost，发 `Step.Ended` |

### 5.1 `updateToolCall` / `completeToolCall`（`processor.ts:138-256`）

```ts
const updateToolCall = Effect.fn("SessionProcessor.updateToolCall")(function* (callID, fn) {
  const match = Object.values(ctx.toolcalls).find((c) => c.call.id === callID)
  // ...更新 ctx.assistantMessage 里的 tool part 状态
  yield* session.updatePart({ ... })
  if (mirrorAssistant) yield* events.publish(SessionEvent.Tool.Input.*, { ... })
})

const completeToolCall = Effect.fn("SessionProcessor.completeToolCall")(function* (callID, output) {
  // ...更新 part 状态为 completed / error
  // ...image normalize (如果有 image attachment)
  yield* session.updatePart({ ... tool, state: { status: "completed", output, metadata, time: { start, end } } })
})
```

**事件 publish 是双写**（dual-write）— `mirrorAssistant` flag 决定是否同时发 V2 event（`Tool.Input.Delta/Ended/Called/Success/Failed`），目前一直开着，等迁移完成会去掉 v1 part 持久化。

### 5.2 doom loop 检测（`processor.ts:522-545`）

```ts
if (
  recentParts.length !== DOOM_LOOP_THRESHOLD ||
  !recentParts.every(
    (part) => part.type === "tool" &&
              part.tool === value.name &&
              part.state.status !== "pending" &&
              JSON.stringify(part.state.input) === JSON.stringify(input),
  )
) return

yield* permission.ask({ permission: "doom_loop", ... })
```

同 tool 同 input 连续出现 `DOOM_LOOP_THRESHOLD` 次 → 触发 `doom_loop` 权限询问。

---

## 六、AI SDK 自动执行 tool

**关键认知**：LLM 发出 `tool-call` 后，**AI SDK（`ai` package）内部**就会调 `tool({...}).execute(args, options)` — 这是 ai SDK 的标准行为，**不是 opencode 自己代码**。

opencode 在 `tools.ts:80-114` 通过闭包捕获：
- `run = EffectBridge.make()` — Effect 上下文桥
- `execute(args, options) { return run.promise(Effect.gen(...)) }` — 把 Effect-based 工具实现包成 Promise-returning function 给 AI SDK

```ts
tools[item.id] = tool({
  description: item.description,
  inputSchema: jsonSchema(schema),
  execute(args, options) {
    return run.promise(
      Effect.gen(function* () {
        const ctx = context(args, options)
        yield* plugin.trigger("tool.execute.before", { tool: item.id, sessionID: ctx.sessionID, callID: ctx.callID }, { args })
        const result = yield* item.execute(args, ctx)              // ← 实际 tool 跑
        // ... 调 plugin.trigger("tool.execute.after", ...)
        return output
      }),
    )
  },
})
```

**执行链**：
```
ai SDK auto-execute
  → SessionTools.execute 闭包 (run.promise)
    → Effect.gen
      → plugin.trigger("tool.execute.before")
      → ctx.ask(permission) (如需要)
      → item.execute(args, ctx)              ← Tool.init 包装的函数
        → Schema.decode(args) 验参
        → toolInfo.execute(decoded, ctx)    ← 内置 tool 实际逻辑 (如 BashTool.execute)
          → Effect.gen 里跑 shell 进程 / 读文件 / 写文件
          → yield* Truncate.output (截断)
        → 返回 { title, output, metadata, attachments }
      → plugin.trigger("tool.execute.after", ..., output)
      → 返回 output
```

---

## 七、tool-result 回 LLM

`handleEvent("tool-result")`（`processor.ts:549-647`）：

```ts
case "tool-result": {
  const toolCall = yield* readToolCall(value.id)
  if (!toolCall && value.result.type === "error") return
  if (value.result.type === "error") {
    // publish Tool.Failed
    yield* failToolCall(value.id, error)
    return
  }
  // image normalize (如有 image attachment)
  const output = { ...rawOutput, output, attachments }
  // publish Tool.Success with full content + attachments
  yield* completeToolCall(value.id, output)
  return
}
```

**回 LLM**：AI SDK 在 `tool.execute()` Promise resolve 后**自动**把 output 编码进下一轮 LLM 调用的 `messages` 数组（`tool` role + `content: { type: "tool_result", tool_use_id, content }`）— runLoop 下轮把 `tool_use + tool_result` 一起装回 `modelMsgs`，LLM 看到自己调 tool 的结果。

---

## 八、TUI 端：纯渲染

TUI **不加载 tool** — server 完成后通过 V2 事件流推过来：

| 事件 | TUI 端 store 更新 | 渲染 |
| ---- | ---------------- | ---- |
| `Tool.Input.Delta` | `message.part.delta` 流式追加 tool call raw input | `<textarea>` 里 tool input 文本流式显示 |
| `Tool.Input.Ended` | `message.part.updated` reconcile | tool input 折叠 + 准备渲染 result |
| `Tool.Called` | `message.part.updated` 标 running | tool card 状态变 "running" |
| `Tool.Success` | `message.part.updated` 标 completed + 写 output | tool result 展开显示 |
| `Tool.Failed` | `message.part.updated` 标 error + 写 error msg | tool card 红框显示 |

**渲染位置**（`packages/tui/src/routes/session/index.tsx`）：`<For each={messages()}>` → 每条 message 内部按 `parts[]` 渲染，tool part 用 `<ToolPart status={...} input={...} output={...} />` 显示 — 折叠/展开、状态色、stdout/stderr 分别着色。

---

## 九、值得注意的几个点

1. **Tool 加载是 lazy** — `InstanceState.make` 首次访问才扫文件 + 装配；不访问就不跑
2. **Per-project 缓存** — 同一 directory 多次 `tools()` 命中 cache；切 directory 才重新扫
3. **动态 import tool 文件** — `Effect.promise(() => import(pathToFileURL(match).href))` — Windows 也能跑；用户写的 tool 文件**是普通 Node 模块**
4. **AI SDK 自动 execute** — opencode 不需要写 "调 tool 后等待结果" 的代码，ai SDK `tool({ execute })` 内置了完整的 tool-use ↔ tool-result 桥；opencode 只在 `tools.ts:74-115` 用闭包把 Effect 包装成 Promise
5. **工具流式输入** — LLM 边生成 tool input 边通过 `tool-input-delta` 事件流回 TUI（用户能看到 tool 名字、参数一行行出现）
6. **MCP tool 跟内置 tool 走两条独立分支**（`tools.ts:74-115` 内置 / `:117-202` MCP）但对外表现一致（同样 tool-call → tool-result 流，同样 session event）
7. **Plugin 既能添加 tool 也能改 tool 描述/参数** — `tool.definition` 钩子在最终传给 LLM 前修改
8. **doom loop 防失控** — 连续 N 次同 tool 同 input 自动触发 permission ask
9. **Output truncate** — 大输出截断后写盘 + metadata 标 `truncated: true, outputPath: "..."`，避免 LLM context 被刷爆
10. **schema 验证失败友好** — `InvalidArgumentsError` message 是给模型看的"请重写 input 以满足 schema" 提示

---

## 十、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| Tool 接口 + `define` + `init` | `packages/opencode/src/tool/tool.ts:1-183` |
| ToolRegistry（per-project cache） | `packages/opencode/src/tool/registry.ts:83-440` |
| 15 个内置 tool | `packages/opencode/src/tool/{bash,read,write,edit,glob,grep,task,fetch,todo,websearch,skill,question,lsp,plan,apply_patch,invalid}.ts` |
| 工具路径扫描 | `packages/opencode/src/config/paths.ts:23-41` |
| SessionTools.resolve（暴露给 LLM） | `packages/opencode/src/session/tools.ts:24-205` |
| LLM stream + tool 编码 | `packages/llm/src/route/client.ts:272-373` |
| `llm.stream` 协议适配 | `packages/llm/src/route/transport/http.ts:83-102` |
| LLMEvent 处理 | `packages/opencode/src/session/processor.ts:371-844` |
| `updateToolCall` / `completeToolCall` | `packages/opencode/src/session/processor.ts:138-256` |
| `runLoop` 调 resolve | `packages/opencode/src/session/prompt.ts:1279-1347` |
| Doom loop 检测 | `packages/opencode/src/session/processor.ts:522-545` |
| Tool truncate | `packages/opencode/src/tool/truncate.ts` |
| Tool 端到端流程 | 详见 `review/process/tui-prompt-flow.md` 阶段 5-8 |
