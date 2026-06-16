# OpenCode TUI 加载与调用 MCP 完整链路

> 分析时间: 2026-06-15
> 范围: TUI 启动后如何加载 MCP server 状态、用户在 TUI 操作 MCP、LLM 如何调用 MCP tool、tool 结果如何回到 TUI
>
> 路径以 `packages/opencode/src/mcp/` 和 `packages/opencode/src/session/tools.ts` 为根。TUI 端**不直接连接** MCP — 全部走 server 端 `MCP.Service`。
> 涉及三个 npm 包：`@modelcontextprotocol/sdk`（MCP 协议客户端）、`@opencode-ai/llm`（opencode 自己的 LLM 协议层）、`ai`（Vercel AI SDK）。

---

## 一、整体 6 阶段总览

```
[1] TUI 启动 → SyncProvider 拉 MCP 状态
       │  GET /mcp → InstanceContextMiddleware → MCP.Service.status()
       │  → 返回 { name: { status: "connected" | "failed" | ... } } 写进 sync.data.mcp
       │
       ▼
[2] 服务端 MCP 装配（per-project, lazy）
       │  MCP.Service.layer 装配, InstanceState.make<State>(init)
       │  init(ctx):
       │    ├── 读 config.mcp 列表
       │    ├── 并行 create(key, mcp):
       │    │     ├── type: "local"  → spawn 子进程 (StdioClientTransport)
       │    │     └── type: "remote" → StreamableHTTP / SSE Transport
       │    ├── client.listTools() → 拿 MCP server 暴露的 tools
       │    ├── s.clients[key] = client
       │    ├── s.defs[key] = toolDefs
       │    └── watch(client) 注册 onclose + ToolListChanged 监听
       │
       ▼
[3] MCP tools 暴露给 LLM
       │  SessionTools.resolve (session/tools.ts:117-202):
       │    for ([key, item] of mcp.tools()):
       │      key = "<clientName>_<toolName>"  (sanitize 后的复合 key)
       │      schema = ProviderTransform.schema(model, asSchema(item.inputSchema).jsonSchema)
       │      item.execute = wrap  (add tool.execute.before/after hooks, ctx.ask)
       │      tools[key] = item
       │  → tools: Record<string, AITool> 传给 LLM streamText
       │
       ▼
[4] LLM 调 MCP tool
       │  AI SDK streamText 内部循环:
       │    解析 LLM 输出 tool-call
       │    找到 tools[<clientName>_<toolName>].execute
       │    调 execute(args, options)  ← 这是 SessionTools.ts:124-201 包装的 execute
       │      ├─ ctx.ask(permission: <mcpName>_<toolName>, patterns: ["*"])
       │      │   → Permission.Service.ask → 发 permission.asked 事件 → TUI 弹授权
       │      ├─ client.callTool({name, args}, schema, {timeout, signal, resetTimeoutOnProgress})
       │      │   → StdioClient / StreamableHTTP / SSE 通信
       │      │   → MCP server 执行 tool, 返回 CallToolResult
       │      ├─ 把 MCP result.content 转 SessionV1.FilePart[] (text/image/resource)
       │      └─ truncate.output 截断
       │  → Promise resolve
       │  → AI SDK 编码为 tool-result 消息
       │  → 下一轮 LLM call
       │
       ▼
[5] 结果回 TUI 渲染
       │  AI SDK fullStream 吐 tool-result 事件
       │  → LLMAISDK.toLLMEvents 翻译
       │  → SessionProcessor.handleEvent("tool-result")
       │  → completeToolCall 写 part
       │  → publish Tool.Success 事件
       │  → EventV2Bridge → GlobalBus → worker Rpc.emit → TUI SDKProvider SSE
       │  → SyncProvider 更新 store
       │  → TUI Session 路由渲染 tool card
       │
       ▼
[6] TUI 主动操作 MCP (sidebar / dialog / 命令面板)
       │  DialogMcp / sidebar/mcp.tsx 触发 local.mcp.toggle / connect / disconnect
       │  → POST /mcp/:name/{connect|disconnect} 或 POST /mcp
       │  → 服务端 MCP.Service.add / connect / disconnect
       │  → 重置 InstanceState → 重连或重配置
```

---

## 二、阶段 1 — TUI 拉 MCP 状态

**文件**: `packages/tui/src/context/sync.tsx:493`

```ts
sdk.client.mcp.status({ workspace }).then((x) => setStore("mcp", reconcile(x.data ?? {})))
```

**TUI 拉的是 MCP **状态** 列表（不是工具列表）**：

| 状态 | 含义 |
| ---- | ---- |
| `connected` | 客户端连上了 MCP server，工具可用 |
| `disabled` | 配置里 `enabled: false` |
| `failed` | 连接失败，附带 `error` 描述 |
| `needs_auth` | 远程 MCP 需要 OAuth 认证 |
| `needs_client_registration` | 服务器不支持动态 client 注册 |

TUI 端用 `sync.data.mcp: Record<name, Status>` 存；通过 SSE 事件 `mcp.tools.changed` 增量更新（`mcp/index.ts:62-67`）。

---

## 三、阶段 2 — 服务端 MCP 装配

### 3.1 `MCP.Service` 接口（`mcp/index.ts:159-186`）

```ts
export interface Interface {
  readonly status: () => Effect<Record<string, Status>>                          // GET /mcp
  readonly clients: () => Effect<Record<string, MCPClient>>                       // 原始 client map
  readonly tools: () => Effect<Record<string, Tool>>                              // ★ 给 LLM 的 tool map
  readonly prompts: () => Effect<Record<string, PromptInfo & { client: string }>>  // 给 TUI 当 command
  readonly resources: () => Effect<Record<string, ResourceInfo & { client: string }>>  // 给 TUI @ 引用
  readonly add: (name, mcp) => Effect<{status}>
  readonly connect: (name) => Effect<void, NotFoundError>
  readonly disconnect: (name) => Effect<void, NotFoundError>
  readonly getPrompt: (clientName, name, args) => Effect<...>                    // 给 Command.Service 用
  readonly readResource: (clientName, resourceUri) => Effect<...>                // 给 TUI @ 引用用
  readonly startAuth: (mcpName) => Effect<{authorizationUrl, oauthState}, NotFoundError>
  readonly authenticate: (mcpName) => Effect<Status, NotFoundError>               // POST /mcp/:name/auth/authenticate
  readonly finishAuth: (mcpName, code) => Effect<Status, NotFoundError>          // POST /mcp/:name/auth/callback
  readonly removeAuth: (mcpName) => Effect<void>
  readonly supportsOAuth, hasStoredTokens, getAuthStatus: ...
}
```

### 3.2 State（`mcp/index.ts:152-157`）

```ts
interface State {
  config: Record<string, ConfigMCPV1.Info>   // MCP server 配置
  status: Record<string, Status>             // 状态
  clients: Record<string, MCPClient>          // ★ 活的 MCP client
  defs: Record<string, MCPToolDef[]>          // ★ 工具定义（listTools 缓存）
}
```

### 3.3 per-project 装配（`mcp/index.ts:473-538`）

```ts
const state = yield* InstanceState.make<State>(
  Effect.fn("MCP.state")(function* () {
    const cfg = yield* cfgSvc.get()
    const config = cfg.mcp ?? {}
    const s: State = { config: {}, status: {}, clients: {}, defs: {} }

    yield* Effect.forEach(
      Object.entries(config),
      ([key, mcp]) =>
        Effect.gen(function* () {
          if (!isMcpConfigured(mcp)) { ... return }
          if (mcp.enabled === false) { s.status[key] = { status: "disabled" }; return }
          const result = yield* create(key, mcp)        // ★ 连接 MCP server
          s.status[key] = result.status
          if (result.mcpClient) {
            s.clients[key] = result.mcpClient
            s.defs[key] = result.defs!                  // ★ 缓存 tool defs
            watch(s, key, result.mcpClient, bridge, mcp.timeout)
          }
        }),
      { concurrency: "unbounded" },                      // 并行连接所有 MCP server
    )

    yield* Effect.addFinalizer(() => /* close clients, kill child PIDs */)
    return s
  }),
)
```

### 3.4 连接 — `create(key, mcp)`（`mcp/index.ts:359-397`）

按 `mcp.type` 分支：

**Local 模式**（`mcp/index.ts:327-357` `connectLocal`）：
```ts
const transport = new StdioClientTransport({
  stderr: "pipe",
  command: cmd,        // ["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
  args,
  cwd: baseDir,        // 默认 workspace dir
  env: { ...process.env, ...mcp.environment },
})
```

**Remote 模式**（`mcp/index.ts:223-325` `connectRemote`）：
```ts
const transports = [
  { name: "StreamableHTTP", transport: new StreamableHTTPClientTransport(url, { authProvider, requestInit }) },
  { name: "SSE",            transport: new SSEClientTransport(url, ...) },
]
// 依次尝试, 哪个连成功用哪个
for (const { name, transport } of transports) {
  const result = yield* connectTransport(transport, timeout)
  if (result) return { client: result.client, status: { status: "connected" } }
}
```

**OAuth 处理**：失败如果是 `UnauthorizedError` → `status: "needs_auth"`，transport 存到 `pendingOAuthTransports` 供后续 `finishAuth` 复用；如果是 client registration 错误 → `status: "needs_client_registration"`。

### 3.5 工具拉取（`mcp/index.ts:377-382`）

```ts
const listed = mcpClient.getServerCapabilities()?.tools
  ? yield* McpCatalog.defs(mcpClient, mcp.timeout)
  : []
```

`McpCatalog.defs`（`mcp/catalog.ts:38-40`）→ `listTools` → 走 MCP 的 `tools/list` JSON-RPC 方法 → 返回 `MCPToolDef[]`。

### 3.6 watch（`mcp/index.ts:424-453`）

每个 client 注册两个 handler：
- `client.onclose` → 清理 `s.clients[name]`, `s.defs[name]`，标 `status: "failed"`，发 `mcp.tools.changed` 事件
- `ToolListChangedNotificationSchema` → 重新 listTools → 重新 publish `mcp.tools.changed`

> `mcp.tools.changed` 事件让 TUI 端能感知"工具列表变了"，触发 SyncProvider 重新拉。

---

## 四、阶段 3 — MCP tools 暴露给 LLM

**文件**: `packages/opencode/src/session/tools.ts:117-202`

```ts
for (const [key, item] of Object.entries(yield* mcp.tools())) {
  const execute = item.execute
  if (!execute) continue

  const schema = yield* Effect.promise(() => Promise.resolve(asSchema(item.inputSchema).jsonSchema))
  const transformed = ProviderTransform.schema(input.model, schema)
  item.inputSchema = jsonSchema(transformed)
  item.execute = (args, opts) =>
    run.promise(
      Effect.gen(function* () {
        const ctx = context(args, opts)
        yield* plugin.trigger("tool.execute.before", { tool: key, sessionID: ctx.sessionID, callID: opts.toolCallId }, { args })
        const result = yield* Effect.gen(function* {
          yield* ctx.ask({ permission: key, metadata: {}, patterns: ["*"], always: ["*"] })
          return yield* Effect.promise(() => execute(args, opts))
        }).pipe(Effect.withSpan("Tool.execute", { attributes: { "tool.name": key, ... } }))
        yield* plugin.trigger("tool.execute.after", { tool: key, sessionID: ctx.sessionID, callID: opts.toolCallId, args }, result)

        // 把 MCP result.content 转 SessionV1 attachments
        const textParts: string[] = []
        const attachments: Omit<SessionV1.FilePart, "id" | "sessionID" | "messageID">[] = []
        for (const contentItem of result.content) {
          if (contentItem.type === "text")  textParts.push(contentItem.text)
          else if (contentItem.type === "image") {
            attachments.push({ type: "file", mime: contentItem.mimeType, url: `data:${contentItem.mimeType};base64,${contentItem.data}` })
          }
          else if (contentItem.type === "resource") {
            if (contentItem.resource.text) textParts.push(contentItem.resource.text)
            if (contentItem.resource.blob) {
              attachments.push({ type: "file", mime: ..., url: `data:...;base64,...`, filename: contentItem.resource.uri })
            }
          }
        }

        const truncated = yield* truncate.output(textParts.join("\n\n"), {}, input.agent)
        const output = {
          title: "", metadata: { ...result.metadata, truncated: truncated.truncated, ... },
          output: truncated.content,
          attachments: attachments.map((a) => ({ ...a, id: PartID.ascending(), sessionID: ctx.sessionID, messageID: input.processor.message.id })),
          content: result.content,
        }
        if (opts.abortSignal?.aborted) yield* input.processor.completeToolCall(opts.toolCallId, output)
        return output
      }),
    )
  tools[key] = item
}
```

### 关键点

**MCP tool key**：`"<clientName>_<toolName>"`（`mcp/catalog.ts:102` `sanitize` 替换非字母数字为 `_`）
- 例：MCP server `filesystem` 的 tool `read_file` → `filesystem_read_file`
- 这个 key 是 LLM 看到的 tool 名字

**`mcp.tools()` 返回**（`mcp/index.ts:628-651`）：
```ts
const tools = Effect.fn("MCP.tools")(function* () {
  const result: Record<string, Tool> = {}
  for (const [clientName, client] of Object.entries(s.clients)) {
    if (s.status[clientName]?.status !== "connected") continue
    const listed = s.defs[clientName]
    for (const mcpTool of listed) {
      const key = McpCatalog.sanitize(clientName) + "_" + McpCatalog.sanitize(mcpTool.name)
      result[key] = McpCatalog.convertTool(mcpTool, client, timeout)
    }
  }
  return result
})
```

**`McpCatalog.convertTool`**（`mcp/catalog.ts:42-74`）把 MCP tool 转 AI SDK 的 `Tool`：
```ts
return dynamicTool({
  description: mcpTool.description ?? "",
  inputSchema: jsonSchema(inputSchema),
  execute: async (args, options) => {
    const result = await client.callTool(
      { name: mcpTool.name, arguments: (args || {}) as Record<string, unknown> },
      CallToolResultSchema,
      { resetTimeoutOnProgress: true, signal: options.abortSignal, timeout },
    )
    if (result.isError) throw new Error(formatToolErrorContent(result.content))
    if (result.structuredContent === undefined || result.structuredContent === null) return result
    return {
      ...result,
      content: [{ type: "text" as const, text: JSON.stringify(result.structuredContent) }],
    }
  },
})
```

**重要的时序**：
1. `McpCatalog.convertTool` 的 `execute` 调 `client.callTool`（实际 MCP 通信）
2. **但是**这层 `execute` 在 `session/tools.ts:124` 被**包了一层**（`item.execute = (args, opts) => run.promise(...)`）— 这层是真正 AI SDK 调的入口
3. **包的那层先**调 `plugin.trigger("tool.execute.before")` 和 `ctx.ask(permission)`，**再**调上面那个 execute
4. `ctx.ask(permission: key, ...)` 这里的 `key` 是 `"<clientName>_<toolName>"` 形式 — TUI 权限弹窗显示这个完整 key

---

## 五、阶段 4 — LLM 调 MCP tool 实际跑流程

```
LLM 决定调 tool, 输出 tool-call: { name: "filesystem_read_file", input: { path: "/foo" } }
  ↓
AI SDK streamText 内部: 找到 tools["filesystem_read_file"].execute
  ↓
调 session/tools.ts:124 包的那层 execute
  │
  ├── ctx = context(args, options)                     // 构造 Tool.Context
  ├── yield* plugin.trigger("tool.execute.before", { tool: "filesystem_read_file", ... }, { args })
  │       插件可改 args
  ├── yield* ctx.ask({ permission: "filesystem_read_file", patterns: ["*"], always: ["*"] })
  │       → Permission.Service.ask → 发 permission.asked 事件
  │       → TUI 弹授权 dialog
  │       → 等用户选 once/always/reject
  │
  ├── yield* Effect.promise(() => execute(args, opts))  // ← 真正调 McpCatalog.convertTool 包的 execute
  │       │
  │       └── client.callTool({ name: "read_file", arguments: { path: "/foo" } }, schema, options)
  │             │
  │             │  StdioClientTransport (本地子进程)
  │             │    → 写一行 JSON-RPC 到 stdin
  │             │    → 从 stdout 读 SSE / line-delimited JSON 响应
  │             │
  │             │  StreamableHTTPClientTransport (HTTP)
  │             │    → POST {url}  body {jsonrpc:"2.0", method:"tools/call", params:{...}}
  │             │    → 读 response (SSE 流)
  │             │
  │             → 拿到 CallToolResult { content: [{type:"text", text:"..."}], isError: false, ... }
  │
  │   return result  (CallToolResult)
  │
  ├── content 转 SessionV1.FilePart
  ├── yield* truncate.output(textParts.join("\n\n"))
  ├── yield* plugin.trigger("tool.execute.after", ...)
  │
  ├── if (abort) yield* processor.completeToolCall
  │
  └── return { title, output, metadata, attachments, content }
        ↓
        AI SDK 编码为 tool-result 消息
        ↓
        下一轮 LLM call 带上 tool_result
```

---

## 六、阶段 5 — 结果回 TUI 渲染

走**完全相同**的路径（详见 `review/process/tui-prompt-flow.md` / `tool-system.md`）：

```
CallToolResult → execute 返回 Promise
  ↓
AI SDK fullStream 吐 tool-result 事件
  ↓
LLMAISDK.toLLMEvents 翻译成 opencode LLMEvent
  ↓
SessionProcessor.handleEvent "tool-result" (processor.ts:549-647)
  ├── completeToolCall → 写 part state = completed, output + attachments + metadata
  ├── publish SessionEvent.Tool.Success
  ↓
EventV2Bridge → GlobalBus → worker Rpc.emit
  ↓
TUI SDKProvider SSE 收到
  ↓
SyncProvider 收到 Tool.Success
  ↓
session.data.message / session.data.part 更新
  ↓
TUI Session 路由 <For each={messages()}> 重渲
  ↓
tool card 显示输出 + attachments (image / file)
```

**TUI 端 tool 卡片显示**：
- tool name: `filesystem_read_file`（MCP key 显示完整）
- 状态: completed / error
- output: 截断的 stdout
- attachments: 图片（base64）/ 文件（file://）

---

## 七、阶段 6 — TUI 主动操作 MCP

### 7.1 Sidebar 状态展示（`packages/tui/src/feature-plugins/sidebar/mcp.tsx`）

TUI 内置 plugin `SidebarMcp`（`feature-plugins/builtins.ts:24` 之一）：

```ts
const list = createMemo(() => props.api.state.mcp())  // 读 sync.data.mcp
const on = () => list().filter((item) => item.status === "connected").length
const bad = () => list().filter((item) =>
  item.status === "failed" || item.status === "needs_auth" || item.status === "needs_client_registration"
).length
```

显示在 session 侧边栏：每个 MCP server 一个状态点（绿=connected / 红=failed / 黄=needs_auth）。

### 7.2 DialogMcp（`packages/tui/src/component/dialog-mcp.tsx`）

打开"MCPs" dialog：

```ts
const mcpData = sync.data.mcp
const options = pipe(mcpData ?? {}, entries(), sortBy(([name]) => name), ...)
// 每行显示 server name + status
// 触发 action: "dialog.mcp.toggle"
await local.mcp.toggle(option.value)
const status = await sdk.client.mcp.status()  // 重新拉状态
sync.set("mcp", status.data)
```

### 7.3 `local.mcp` 操作（`packages/tui/src/context/local.tsx:513-516`）

```ts
await sdk.client.mcp.disconnect({ name })
await sdk.client.mcp.connect({ name })
```

**HTTP 端点**（`packages/opencode/src/server/routes/instance/httpapi/groups/mcp.ts:32-39`）：

| Path | Method | 作用 |
| ---- | ------ | ---- |
| `/mcp` | GET | 拉所有 server 状态 |
| `/mcp` | POST (name + config) | 动态加一个 MCP server |
| `/mcp/:name/connect` | POST | 启用 + 连接 |
| `/mcp/:name/disconnect` | POST | 关闭连接 |
| `/mcp/:name/auth` | POST | 启动 OAuth 流 |
| `/mcp/:name/auth/callback` | POST | 完成 OAuth（用 code）|
| `/mcp/:name/auth/authenticate` | POST | 一步 OAuth（开浏览器）|
| `/mcp/:name/auth` | DELETE | 清 OAuth credentials |

### 7.4 OAuth 流（`mcp/oauth-provider.ts` + `mcp/oauth-callback.ts`）

- `startAuth` → 启动本地 callback HTTP server（`McpOAuthCallback.ensureRunning`）→ 让 MCP server 重定向到 `http://127.0.0.1:<port>/mcp-oauth-callback`
- `authenticate`（一步流）→ 调 `startAuth` → 用 `open` 包开浏览器 → 等 callback promise（`McpOAuthCallback.waitForCallback`）→ 拿 authorization code → 调 `finishAuth`
- `finishAuth` → 调 `transport.finishAuth(code)` → `createAndStore` 重新连接

**`opencode mcp auth <server>` CLI**（`packages/opencode/src/cli/cmd/mcp.ts:275`）调 `MCP.authenticate(serverName)` 走同样流程。

### 7.5 MCP prompts → command（`command/index.ts:113-140`）

```ts
for (const [name, prompt] of Object.entries(yield* mcp.prompts())) {
  commands[name] = {
    name,
    source: "mcp",
    description: prompt.description,
    get template() {
      return bridge.promise(
        mcp.getPrompt(prompt.client, prompt.name, ...).pipe(
          Effect.map((template) =>
            template?.messages.map((m) => (m.content.type === "text" ? m.content.text : "")).join("\n") || ""
          )
        )
      )
    },
    hints: prompt.arguments?.map((_, i) => `$${i + 1}`) ?? [],
  }
}
```

→ MCP server 暴露的 prompt 也会作为 **slash command** 出现在 TUI `/` 弹窗里（参见 `tui-command-list-flow.md` 第三节）。用户 `/<prompt-name> <args>` → `Command.Service` → 模板渲染 → LLM 拿到 prompt 内容。

### 7.6 MCP resources → `@` 引用（`autocomplete.tsx:355-390`）

TUI 的 `@` autocomplete 也会拉 MCP resources（`mcp.resources()`），作为可引用的 file part。`autocomplete.tsx:361`：
```ts
for (const res of Object.values(sync.data.mcp_resource)) {
  // 渲染为 @<name> 引用选项
}
```

`mcp_resource` 是 `SyncProvider` 拉的另一个 store 字段（`sync.tsx:494`：`sdk.client.experimental.resource.list({ workspace })`）。

---

## 八、关键设计点

1. **TUI 不直连 MCP** — 全部走 server 端 `MCP.Service` + HTTP API；TUI 端只读 `sync.data.mcp` 状态 / 触发 HTTP 端点
2. **per-project 配置** — MCP 配置是 `InstanceState` per-project 缓存；切 project dir 会重连
3. **MCP tool key = `<client>_<tool>`** — 避免不同 server 工具名冲突；TUI 显示完整 key 让用户知道是哪个 server 的 tool
4. **每次 MCP tool call 都触发 `ctx.ask`**（`tools.ts:134`）— `permission: "<mcpClient>_<toolName>"`, `patterns: ["*"]`, `always: ["*"]` — MCP tool 一律问用户（不像内置 tool 可以从 agent permission 推断）；`always: ["*"]` 让"总是允许"按 key 粒度记
5. **`mcp.tools.changed` 事件驱动** — 服务端 onclose / ToolListChanged → publish → TUI SSE 收到 → SyncProvider 重拉
6. **Local 模式 spawn 子进程** — `StdioClientTransport` 用 `ChildProcessSpawner`（不是 Bun.spawn 也不是 `node:child_process`，走 effect/unstable/process 抽象，跨平台一致）；finalizer 用 `pgrep` 找所有子进程一起 `SIGTERM`（`descendants` 函数）
7. **OAuth 双模式** — 一步流（`authenticate`：服务端 open 浏览器 + 等 callback）/ 两步流（`startAuth` + `finishAuth`：客户端自己 open 浏览器）
8. **MCP prompts 当 command** — TUI `/` 弹窗里出现；`source: "mcp"` 标签 + 显示 `:mcp` 后缀（`autocomplete.tsx:442`）
9. **MCP resources 当 `@` 引用** — TUI `@` autocomplete 列出；选完插 `<file data:base64>` part 进 prompt
10. **`McpCatalog.defs` 容错** — `outputSchema` 验证失败时 fallback 到不带 `outputSchema` 的 schema（`catalog.ts:144-147`）
11. **`@modelcontextprotocol/sdk` 包的 3 个 transport** — `StdioClientTransport`（本地子进程）/ `StreamableHTTPClientTransport`（HTTP POST+SSE）/ `SSEClientTransport`（旧 HTTP+SSE）
12. **MCP tool 输出转 SessionV1.FilePart** — `text` → output 文本；`image` → base64 data URL → `type: "file"` part；`resource` → text/Blob 同样处理（`tools.ts:152-174`）
13. **`TuiEvent.ToastShow` 通知 OAuth 需要认证** — `connectRemote` 失败时发 toast 提示用户跑 `opencode mcp auth <server>`（`mcp/index.ts:291-309`）
14. **Plugin 不影响 MCP tool description** — MCP tool 的 description 直接来自 server，`tool.definition` 钩子跳过 `source: "mcp"` 的（参见 `registry.ts:289-299`，但这里更精细 — MCP tool 是单独分支不在 builtin list）

---

## 九、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| MCP Service 主体 | `packages/opencode/src/mcp/index.ts:1-953` |
| 状态 + per-project 装配 | `packages/opencode/src/mcp/index.ts:152-538` |
| Local 连接 | `packages/opencode/src/mcp/index.ts:327-357` |
| Remote 连接（HTTP/SSE） | `packages/opencode/src/mcp/index.ts:223-325` |
| `mcp.tools()` 暴露给 LLM | `packages/opencode/src/mcp/index.ts:628-651` |
| MCP tool 转 AI SDK tool | `packages/opencode/src/mcp/catalog.ts:42-74` |
| Tool defs 拉取 | `packages/opencode/src/mcp/catalog.ts:38-40, 136-153` |
| MCP schema sanitize | `packages/opencode/src/mcp/catalog.ts:102` |
| SessionTools 包装 MCP | `packages/opencode/src/session/tools.ts:117-202` |
| MCP config schema | `packages/core/src/config/mcp.ts:1-39` |
| OAuth provider | `packages/opencode/src/mcp/oauth-provider.ts` |
| OAuth callback server | `packages/opencode/src/mcp/oauth-callback.ts` |
| MCP auth 状态 | `packages/opencode/src/mcp/auth.ts` |
| `opencode mcp` CLI | `packages/opencode/src/cli/cmd/mcp.ts` |
| MCP HTTP API | `packages/opencode/src/server/routes/instance/httpapi/groups/mcp.ts` |
| MCP HTTP handlers | `packages/opencode/src/server/routes/instance/httpapi/handlers/mcp.ts` |
| MCP prompts → command | `packages/opencode/src/command/index.ts:113-140` |
| TUI sidebar MCP 状态 | `packages/tui/src/feature-plugins/sidebar/mcp.tsx` |
| TUI MCP dialog | `packages/tui/src/component/dialog-mcp.tsx` |
| TUI MCP local actions | `packages/tui/src/context/local.tsx:513-516` |
| TUI SyncProvider 拉 MCP 状态 | `packages/tui/src/context/sync.tsx:493` |
| TUI `@` autocomplete 拉 MCP resources | `packages/tui/src/component/prompt/autocomplete.tsx:355-390` + `sync.tsx:494` |
