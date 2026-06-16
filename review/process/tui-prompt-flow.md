# OpenCode TUI 输入到响应完整处理流程

> 分析时间: 2026-06-15
> 范围: 用户在 TUI 提示框输入一段话 → 后端处理 → 流式响应回显到终端
>
> 路径以 `packages/` 为根。"RPC"指 `util/rpc.ts:1-66` 的 JSON over Worker `postMessage`；"SSE"指 `text/event-stream` 格式 HTTP 响应；"SDK"指 `@opencode-ai/sdk/v2`。

---

## 一、阶段总览

```
[1] TUI 端输入 ────────────────► submit() @ prompt/index.tsx:925
       │
[2] SDK Client ────────────────► sdk.client.session.prompt(...)  (POST /session/:id/prompt_async)
       │                            ↳ 走 SDK fetch:
       │                                - internal transport → RPC → worker → Server.Default().app.fetch（进程内）
       │                                - external transport → 真 HTTP
       │
[3] HTTP 路由 ──────────────────► SessionHttpApi.promptAsync @ handlers/session.ts:309
       │     yield* promptSvc.prompt(...).pipe(Effect.forkIn(scope))  // 立即返回 204
       │
[4] SessionPrompt.prompt ──────► 写 user msg + 触发起 SessionRunState.ensureRunning → runLoop
       │
[5] SessionPrompt.runLoop ─────► while(true) {
       │                              create Assistant msg
       │                              processor.create(...)            // 拿到 Handle
       │                              handle.process({...})           // 调 LLM + 处理事件
       │                          }
       │     每轮 yield* session.updateMessage / updatePart  → 写 SQLite
       │     每轮 yield* events.publish(SessionEvent.*, ...) → EventV2Bridge
       │
[6] SessionProcessor.process ──► const stream = llm.stream(streamInput)            @ processor.ts:974
       │                          stream.pipe(Stream.tap(handleEvent), runDrain)  @ processor.ts:976-980
       │
[7] LLM stream  ───────────────► LLMClient.stream → compile → route.streamPrepared
       │     packages/llm/src/route/client.ts:367 → :272
       │     transport.frames() 走 HttpClient.execute  (POST + SSE)
       │     SSE frame → protocol.step state machine → LLMEvent
       │     文本/工具调用/usage 都被规范化为 LLMEvent（text-delta, tool-input-delta, tool-call, tool-result, step-finish, finish）
       │
[8] handleEvent @ processor.ts:371-844 ─► switch LLMEvent:
       │     text-delta       → session.updatePartDelta(...) + publish(SessionEvent.Text.Delta)
       │     text-start       → 新建 text part  + publish(SessionEvent.Text.Started)
       │     tool-input-start → ensureToolCall 新建 tool part
       │     tool-input-delta → updateToolCall 累加 raw input + publish(SessionEvent.Tool.Input.Delta)
       │     tool-call        → updateToolCall 固化 input + publish(SessionEvent.Tool.Called)
       │     tool-result      → completeToolCall + publish(SessionEvent.Tool.Success|Failed)
       │     step-start/finish→ publish(SessionEvent.Step.*) + 累计 tokens/cost + 检查 overflow → needsCompaction
       │     reasoning-*      → 同样进 reasoningMap + publish(SessionEvent.Reasoning.*)
       │     provider-error   → throw（retry 策略包了一层）
       │
[9] EventV2Bridge.publish ─────► events.publish(definition, data, {location: ctx.directory})
       │     同时落到持久化层（Drizzle event table）+ emit 到 GlobalBus (Node EventEmitter)
       │
[10] Worker 转发 ─────────────► worker.ts:17-19
       │     GlobalBus.on("event", e => Rpc.emit("global.event", e))
       │     主线程 createEventSource(client) 收到 RPC event
       │
[11] TUI SSE 接收 ─────────────► SDKProvider.startSSE @ packages/tui/src/context/sdk.tsx:82
       │     internal: events.on("event", handler) 直接绑 RPC 事件
       │     external: sdk.global.event({ signal, sseMaxRetryAttempts:0 }) 走 /global/event SSE
       │     /global/event handler @ handlers/global.ts:33-66 走 GlobalBus + Sse.encode
       │     事件批 16ms（@ sdk.tsx:75-79）→ batch() 单次 Solid 渲染
       │
[12] Sync store 更新 ──────────► SyncProvider @ packages/tui/src/context/sync.tsx
       │     switch event.type:
       │       message.part.delta   → setStore("part", messageID, produce(...) => text += delta)
       │       message.part.updated → setStore("part", messageID, reconcile(part))
       │       message.updated      → setStore("message", sessionID, reconcile(msg))
       │       session.status       → setStore("session_status", ...)
       │       ...更多（lsp, mcp, todo, permission, question 等）
       │
[13] Session 路由渲染 ──────────► routes/session/index.tsx
       │     const messages = createMemo(() => sync.data.message[route.sessionID] ?? [])   @ :207
       │     <For each={messages()}> ... 渲染每条 msg                                 @ :1188
       │     text part 用 <text>{part.text}</text> 之类的节点，依赖 store 重新计算
       │     Solid 响应式：store 变化 → 组件重渲染 → opentui 调度下一帧
       │
[14] Loop 继续 / 结束 ────────► runLoop 检查 lastAssistant.finish
       │     "stop"（无 tool calls）→ break → 退出 runLoop
       │     "tool-calls" / 有未执行的 tool → continue → 下一轮再调 llm.stream
       │     期间 tools 同步执行（同步分支由 SessionTools.resolve 注入到 request.tools）
       │     Session 状态机：busy → idle（SessionStatus.set 发布）
```

---

## 二、阶段 1 — TUI 端输入

**文件**: `packages/tui/src/component/prompt/index.tsx:925-1141`

| 行号 | 步骤 |
| ---- | ---- |
| `:925-939` | `submit()` — 防止重复提交（`submitting` 闭包锁），防双 Enter / 双 dispatch |
| `:941-954` | `submitInner()` — IME 同步、disabled 检查、workspace 状态、输入非空校验 |
| `:955-966` | 取 `local.agent.current()` / `local.model.current()` / `local.model.variant.current()`；缺失弹 model warning |
| `:968-981` | 检查 workspaceID 连通性（不连通弹 `DialogWorkspaceUnavailable`）|
| `:990-1018` | 若还没有 `sessionID` → `sdk.client.session.create({ directory, agent, model })` |
| `:1020-1031` | 把粘贴的 extmark 内容 `expandTrackedPastedText` 展开成 text parts；过滤掉已展开的 `text` parts |
| `:1053-1115` | **分支**：<br>• `store.mode === "shell"` → `sdk.client.session.shell({ sessionID, agent, model, command })`<br>• 输入以 `/` 开头且匹配到 command → `sdk.client.session.command({ sessionID, command, arguments, agent, model, variant, parts })`<br>• 其它 → `sdk.client.session.prompt({ sessionID, agent, model, variant, parts: [...editorParts, {type:"text", text:inputText}, ...nonTextParts] }, { throwOnError: true })` |
| `:1116-1140` | 收尾：`history.append(...)` / `input.extmarks.clear()` / `setStore("prompt", { input: "", parts: [] })` / `props.onSubmit?.()` / 若新建 session 则 `setTimeout(route.navigate({type:"session", sessionID}), 50)` / `input.clear()` |

触发方式：

- `prompt.submit` keybind（`:344, :564`） — 回车键
- 程序化 `submit()` ref 调用（`:93, :604, :1388`）— `--prompt` 自动提交、命令面板等

---

## 三、阶段 2 — SDK → server

- SDK 客户端构造（`packages/tui/src/context/sdk.tsx:23-31`）：`createOpencodeClient({ baseUrl, signal, directory, fetch, headers })`
- internal transport 下 `fetch = createWorkerFetch(client)`（`tui.ts:23-39`）：把每个 fetch 调用 JSON-RPC 到 worker 的 `Server.Default().app.fetch`（**进程内**）
- external transport 下走真实 HTTP（节点 `globalThis.fetch`）

请求 URL：`POST /session/{sessionID}/prompt_async`（路由注册于 `groups/session.ts:96`）

---

## 四、阶段 3 — HTTP 路由

**文件**: `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts:309-327`

```ts
const promptAsync = Effect.fn("SessionHttpApi.promptAsync")(function* (ctx) {
  yield* requireSession(ctx.params.sessionID)
  yield* promptSvc.prompt({ ...ctx.payload, sessionID: ctx.params.sessionID }).pipe(
    Effect.catchCause((cause) => /* log + publish session.error event */),
    Effect.forkIn(scope, { startImmediately: true }),
  )
  return HttpApiSchema.NoContent.make()
})
```

- 用 `forkIn(scope)` 立即返回 `204 No Content`，prompt 实际执行在独立 fiber 里跑
- 错误不抛给 HTTP（避免阻塞 UI），改成发 `Session.Event.Error` 事件
- 同一个 handler 的 `prompt`（同文件）走同步模式会 stream 完整 Assistant message；TUI 走的是 `promptAsync`

---

## 五、阶段 4 — SessionPrompt.prompt

**文件**: `packages/opencode/src/session/prompt.ts`（layer 装配 `:97-127`）

- `createUserMessage` 写 `user` message + 把 `parts` 拆解为 `text`/`file`/`agent` parts
- `ensureAssistantMessage` 复用既有 assistant msg 或新建
- `state.ensureRunning(sessionID, lastAssistant, runLoop(sessionID))` 启动 run-loop（`runState` 是 `SessionRunState` service）
- `publish(SessionEvent.Prompted, ...)` 通知 client

---

## 六、阶段 5 — runLoop（agent loop）

**文件**: `packages/opencode/src/session/prompt.ts:1134-1402`

每轮：

| 步骤 | 行号 | 行为 |
| ---- | ---- | ---- |
| 标 busy | `:1142` | `status.set(sessionID, { type: "busy" })` |
| 取消息 | `:1145-1149` | `MessageV2.filterCompactedEffect(sessionID)` + `MessageV2.latest(msgs)` |
| 退出条件 | `:1164-1183` | `lastAssistant.finish` 非 `"tool-calls"` 且无未执行 tool → break |
| 拿模型 | `:1194` | `getModel(lastUser.model.providerID, lastUser.model.modelID, sessionID)` |
| 子任务 | `:1197-1199` | 递归调 `handleSubtask` |
| 压缩 | `:1202-1212` | `compaction.process` 或 `compaction.create` |
| 创建 assistant msg | `:1239-1254` | 写 SQLite（`sessions.updateMessage(msg)`）|
| `processor.create` | `:1266-1272` | 返回 `Handle { message, updateToolCall, completeToolCall, process }` |
| 解析 tools | `:1279-1293` | `SessionTools.resolve({ agent, session, model, processor, ... })` |
| 系统提示 | `:1327-1333` | `[...env, ...instructions, ...(skills ? [skills] : [])]` |
| 调 LLM | `:1336-1347` | `handle.process({ user, agent, system, messages: [...modelMsgs, ...(isLastStep ? [MAX_STEPS] : [])], tools, model, ... })` 返回 `"stop" | "continue" | "compact"` |
| 终止 / 继续 | `:1349-1396` | `if (outcome === "break") break; else continue` |

---

## 七、阶段 6 — SessionProcessor.process

**文件**: `packages/opencode/src/session/processor.ts:960-1034`

```ts
const process = Effect.fn("SessionProcessor.process")(function* (streamInput) {
  return yield* Effect.gen(function* () {
    yield* Effect.gen(function* () {
      ctx.currentText = undefined
      ctx.currentTextID = undefined
      ctx.reasoningMap = {}
      yield* status.set(ctx.sessionID, { type: "busy" })
      const stream = llm.stream(streamInput)                       // LLM.StreamInput

      yield* stream.pipe(
        Stream.tap((event) => handleEvent(event)),                  // 每个 LLMEvent 走 handleEvent
        Stream.takeUntil(() => ctx.needsCompaction),               // 压缩触发点截断
        Stream.runDrain,                                            // 拉满
      )
    }).pipe(
      Effect.onInterrupt(() => /* 标 aborted */),
      Effect.catchCauseIf((cause) => !Cause.hasInterruptsOnly(cause), /* 转 Error */),
      Effect.retry(SessionRetry.policy({ ... })),                  // 可重试错误 → 退避重试
      Effect.catch(halt),                                          // halt 标 assistant error
      Effect.ensuring(cleanup()),                                   // 收尾
    )
    ...
  })
})
```

`handleEvent`（`processor.ts:371-844`）switch LLMEvent 类型，**每个分支都做两件事**：

1. `yield* session.updatePart/updatePartDelta/updateMessage(...)` → 写 SQLite（同时是事件的"事实记录"）
2. `yield* events.publish(SessionEvent.X, { sessionID, ... })` → 走 EventV2Bridge

关键事件类型与对应持久化字段：

| LLMEvent | 持久化 | V2 event |
| -------- | ------ | -------- |
| `text-start` (`:763`) | `session.updatePart({ type:"text", text:"" })` | `Text.Started` |
| `text-delta` (`:784`) | `session.updatePartDelta({ field:"text", delta })` | `Text.Delta` |
| `text-end` (`:806`) | `session.updatePart({ time:{end} })` + 触发 `experimental.text.complete` 钩子 | `Text.Ended` |
| `tool-input-start` (`:427`) | `ensureToolCall` 新建 tool part | (无) |
| `tool-input-delta` (`:434`) | `ctx.toolcalls[id].raw += text` | `Tool.Input.Delta` |
| `tool-input-end` (`:451`) | `ctx.toolcalls[id].inputEnded = true` | `Tool.Input.Ended` |
| `tool-call` (`:468`) | `updateToolCall` 固化 input + doom-loop 检测 | `Tool.Called` |
| `tool-result` (`:549`) | `completeToolCall(value.id, output)` | `Tool.Success` / `Tool.Failed` |
| `tool-error` (`:649`) | `failToolCall(value.id, error)` | `Tool.Failed` |
| `step-start` (`:676`) | `session.updatePart({ type:"step-start", snapshot })` | (dual-write) |
| `step-finish` (`:693`) | 累计 cost/tokens + `updatePart({type:"step-finish",tokens,cost})` + `updateMessage` + `summary.summarize(...)` 派生 | `Step.Ended` |
| `reasoning-start/delta/end` (`:373-426`) | `ctx.reasoningMap[id]` 维护 | `Reasoning.Started/Delta/Ended` |
| `provider-error` (`:673`) | throw（被 retry 策略捕获）| (无) |
| `finish` (`:841`) | return | (无) |

---

## 八、阶段 7 — LLM stream

执行链（`packages/llm/src/route/client.ts`）：

1. `llm.stream(streamInput)` = `LLMClient.stream(request)`（`client.ts:394-399`）
2. → `streamRequestWith`（`client.ts:367-373`）→ `compile(request)`（`client.ts:337-352`）
3. `compile` 解析 `request.model.route` → 调 `route.body.from(request)` 拼出 provider-native body → `route.prepareTransport(body, request)` 构造 `HttpClientRequest`
4. `route.streamPrepared(prepared, request, runtime)`（`client.ts:272-288`）：
   - `transport.frames(prepared, request, runtime)` 走 `HttpClient.execute` 拿到 SSE 字节流
   - `Stream.mapEffect(decodeEvent(route))` — 用 `Framing.sse` 解帧、`protocol.stream.step` 状态机解析成 `LLMEvent`
   - `Stream.mapAccumEffect(initial, step, onHalt)` — 单次 provider turn 的事件流
5. `transport.frames` 在 `transport/http.ts:83-102`：`runtime.http.execute(prepared.request)` 拿响应，再用 `framing.frame(response.stream)` 切 SSE

---

## 九、阶段 8 — 事件回 TUI

`EventV2Bridge.publish`（`event-v2-bridge.ts:22-28`）→ `events.publish(definition, data, { location: ctx.directory })`
→ 同时持久化到 V2 event store + 触发 `GlobalBus.emit("event", ...)`（`bus/global.ts` 是 `EventEmitter`）

### Worker 订阅

`cli/tui/worker.ts:17-19`：

```ts
GlobalBus.on("event", (event) => Rpc.emit("global.event", event))
```

每次 bus 事件 → RPC event 到主线程。

### TUI 接收

`packages/tui/src/context/sdk.tsx:82-100`：

- internal transport：`createEventSource(client).subscribe(handler)` → 内部是 `client.on("global.event", handler)` → 收到 worker 的 RPC event
- external transport：`sdk.global.event({ signal, sseMaxRetryAttempts: 0 })` → 真 HTTP GET `/global/event`
  - `globalHandlers`（`handlers/global.ts:33-66`）订阅 `GlobalBus.on("event", handler)` → `Sse.encode` 输出到 HTTP 流
  - 客户端 SDK 解析 SSE → emit `EventSource.onmessage` 事件

事件批处理（`sdk.tsx:75-79`）：

```ts
if (elapsed < 16) {
  timer = setTimeout(flush, 16)   // 16ms 内合并 → Solid batch 单次重渲染
  return
}
flush()
```

---

## 十、阶段 9 — Sync store 更新

`packages/tui/src/context/sync.tsx`（`handleEvent` 在 `:154-426`）：

| V2 event | 同步到 store |
| -------- | ------------ |
| `message.part.delta`（`:375`）| `setStore("part", messageID, produce(draft => { draft[idx][field] = (existing ?? "") + delta }))` — **追加**模式 |
| `message.part.updated`（`:353`）| `reconcile(part)` — **整体替换** |
| `message.updated`（`:298`）| `reconcile(msg)` |
| `message.removed`（`:338`）| 列表里 splice |
| `session.status`（`:293`）| `session_status[sessionID] = status` |
| `permission.asked` / `replied`（`:167/182`）| `permission[sessionID] = [...]` |
| `question.*`（`:204/220`）| `question[sessionID] = [...]` |
| `todo.updated`（`:242`）| `todo[sessionID] = [...]` |
| `session.diff`（`:246`）| `session_diff[sessionID] = [...]` |
| `lsp.updated` / `mcp.*` / `vcs.*` 等 | 对应 store 字段 |

---

## 十一、阶段 10 — Session 路由渲染

`packages/tui/src/routes/session/index.tsx:207`：

```ts
const messages = createMemo(() => sync.data.message[route.sessionID] ?? [])
```

`<For each={messages()}>` 渲染每条 message（`session/index.tsx:1188-1480` 一带）。每条 message 内部按 `parts[]` 渲染：

- `text` part → `<text>{part.text}</text>`（直接读 store 字段）
- `tool` part → `<ToolPart ...>`（status: running / completed / error / pending）
- `reasoning` part → `<ReasoningPart ...>`
- `step-start` / `step-finish` / `patch` / `snapshot` 等元数据 part → 折叠/展开显示

`part.text` 是 Solid 响应式访问，每次 `setStore("part", ...)` 变更都会触发组件 re-evaluation → opentui renderer 在下一帧重绘 → 字符追加到终端。

---

## 十二、阶段 11 — Loop 继续 / 结束

`runLoop` 顶部检查（`prompt.ts:1164-1183`）：

```ts
if (
  lastAssistant?.finish && !["tool-calls"].includes(lastAssistant.finish) &&
  !hasToolCalls && lastUser.id < lastAssistant.id
) {
  break   // 普通 stop，loop 退出
}
```

否则 `continue` → 下一轮：

1. 重新 `MessageV2.toModelMessagesEffect(msgs, model)` 把工具结果合到 model messages
2. 再调 `llm.stream(...)` 一次（第二轮 provider turn）
3. 工具调用在 `SessionTools.resolve` 时已经通过 `ai` SDK 的 `execute` 包装同步执行，所以 tool call/result 已经在 store 里，下一轮 turn 工具结果会作为新 messages 的一部分

最终所有 tool 调完 + assistant finish = `"stop"` → break → `compaction.prune` 异步 fork → `return lastAssistant(sessionID)`。

---

## 十三、阶段 12 — 收尾

- 退出 runLoop 后 SessionStatus 发 `idle`
- 触发 `compaction.prune`（`:1399`）后台清理过期消息
- `lastAssistant(sessionID)` 返回最终的 assistant message（含 parts）
- 该信息不再通过 HTTP 返回（因为是 async 模式）— 全靠 SSE event 流把状态推给 TUI

---

## 十四、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| TUI prompt submit | `packages/tui/src/component/prompt/index.tsx:925-1141` |
| TUI SDK event subscribe | `packages/tui/src/context/sdk.tsx:82-100` |
| TUI 渲染 messages | `packages/tui/src/routes/session/index.tsx:207, 1188` |
| TUI Sync store | `packages/tui/src/context/sync.tsx:154-426` |
| HTTP prompt_async 路由 | `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts:309-327` |
| SessionPrompt.runLoop | `packages/opencode/src/session/prompt.ts:1134-1402` |
| SessionProcessor.process + handleEvent | `packages/opencode/src/session/processor.ts:371-844, 960-1034` |
| LLM 流 | `packages/llm/src/route/client.ts:272-288, 337-373` |
| HTTP transport frames | `packages/llm/src/route/transport/http.ts:83-102` |
| EventV2 publish bridge | `packages/opencode/src/event-v2-bridge.ts:22-28` |
| Worker 转发事件 | `packages/opencode/src/cli/tui/worker.ts:17-19` |
| /global/event SSE | `packages/opencode/src/server/routes/instance/httpapi/handlers/global.ts:33-66` |

---

## 十五、值得注意的设计点

1. **HTTP 异步化**：`promptAsync` 走 `forkIn(scope)` 立即 204，所有结果通过 SSE 推回，避免长连接断流导致 prompt 失败
2. **dual-write**：当前 session message 走 v1（`updateMessage` / `updatePart`）+ V2 event publish 两套并存（`processor.ts:147` 注释 "temporary dual-write while migrating session messages to v2 events"）
3. **批渲染**：TUI 16ms 批事件（`sdk.tsx:75-79`）保证 SSE 高频 delta 不会爆 Solid 渲染
4. **增量 part update**：text-delta 用 `updatePartDelta` 追加而不是 replace，model 切回流时不会"瞬移"
5. **dedup via worker 转发**：所有 server event 都进 GlobalBus，worker 用一条 `GlobalBus.on("event", ...)` 统一转 RPC，简化 TUI 侧的 event source 抽象
6. **loose `useEvent` / `useSync` 边界**：TUI 组件不直接订阅 SDK event stream，统一经 SyncProvider 同步到 store 后用 `createMemo` 读
7. **dual transport**：internal 模式下 worker 不暴露真实端口，但 `createWorkerFetch` 把每次 fetch 都 RPC 到 worker 内的 `Server.Default().app.fetch` — 对 SDK 完全透明
8. **LLM 协议规范化**：所有 provider（SSE、AWS event-stream、WebSocket）都被 `route.streamPrepared` 归一为统一 `LLMEvent`，上层 processor 只看 12 种事件类型
9. **stream 安全**：`Stream.takeUntil(() => ctx.needsCompaction)` 让压缩在事件流边界处安全截断，不丢消息
10. **doom loop 检测**（`processor.ts:522-545`）：同 tool 同 input 连续出现 `DOOM_LOOP_THRESHOLD` 次就触发 `permission.ask` 给用户确认
