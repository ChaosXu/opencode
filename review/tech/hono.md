# Hono 框架使用分析

## 1. 核心架构

项目使用 `hono-openapi` 扩展库实现 OpenAPI 3.1 规范支持。

### 主应用创建

```typescript
// server.ts
const app = new Hono()

// 路由模块化组织 - 每个路由文件返回一个 Hono 实例
export const SessionRoutes = lazy(() =>
  new Hono()
    .get("/", ...handler)
    .get("/:sessionID", ...handler)
    .post("/", ...handler),
)
```

## 2. 中间件使用模式

### 全局错误处理

```typescript
app.onError((err, c) => {
  if (err instanceof NamedError) {
    let status: ContentfulStatusCode
    if (err instanceof NotFoundError) status = 404
    else if (err instanceof Provider.ModelNotFoundError) status = 400
    else if (err.name.startsWith("Worktree")) status = 400
    else status = 500
    return c.json(err.toObject(), { status })
  }
  if (err instanceof HTTPException) return err.getResponse()
  return c.json(new NamedError.Unknown({ message: err.stack }).toObject(), { status: 500 })
})
```

### 基础认证

```typescript
app.use((c, next) => {
  if (c.req.method === "OPTIONS") return next() // CORS 预检
  const password = Flag.OPENCODE_SERVER_PASSWORD
  if (!password) return next()
  return basicAuth({ username, password })(c, next)
})
```

### 请求日志

```typescript
app.use(async (c, next) => {
  const timer = log.time("request", { method: c.req.method, path: c.req.path })
  await next()
  timer.stop()
})
```

### CORS 配置

```typescript
app.use(
  cors({
    origin(input) {
      if (input.startsWith("http://localhost:")) return input
      if (/^https:\/\/([a-z0-9-]+\.)*opencode\.ai$/.test(input)) return input
      return
    },
  }),
)
```

### 实例上下文注入

```typescript
app.use(async (c, next) => {
  const directory = c.req.query("directory") || c.req.header("x-opencode-directory")
  return Instance.provide({ directory, init: InstanceBootstrap, fn: next })
})
```

## 3. API 端点定义模式

### 标准 REST API 结构

```typescript
// GET 列表端点
.get("/", describeRoute({...}), validator("query", z.object({...})), async (c) => {
  const query = c.req.valid("query")
  const sessions = await Session.list(query)
  return c.json(sessions)
})

// GET 单个资源
.get("/:sessionID", describeRoute({...}), validator("param", z.object({ sessionID: z.string() })), async (c) => {
  const sessionID = c.req.valid("param").sessionID
  const session = await Session.get(sessionID)
  return c.json(session)
})

// POST 创建
.post("/", describeRoute({...}), validator("json", Session.create.schema), async (c) => {
  const body = c.req.valid("json")
  const session = await Session.create(body)
  return c.json(session)
})

// PATCH 更新
.patch("/:sessionID", describeRoute({...}),
  validator("param", z.object({ sessionID: z.string() })),
  validator("json", z.object({ title: z.string().optional() })),
  async (c) => {
    const sessionID = c.req.valid("param").sessionID
    const updates = c.req.valid("json")
    return c.json(await Session.update(sessionID, updates))
  })

// DELETE 删除
.delete("/:sessionID", describeRoute({...}), validator("param", z.object({ sessionID: z.string() })), async (c) => {
  await Session.remove(c.req.valid("param").sessionID)
  return c.json(true)
})
```

### OpenAPI 文档定义

```typescript
describeRoute({
  summary: "List sessions",
  description: "Get a list of all OpenCode sessions",
  operationId: "session.list",
  tags: ["Session"],
  responses: {
    200: {
      description: "List of sessions",
      content: {
        "application/json": {
          schema: resolver(Session.Info.array()),
        },
      },
    },
    ...errors(400, 404),
  },
})
```

## 4. 错误处理和验证方式

### Zod + hono-openapi 验证

```typescript
// 查询参数验证
validator(
  "query",
  z.object({
    directory: z.string().optional().meta({ description: "Filter by project directory" }),
    roots: z.coerce.boolean().optional().meta({ description: "Only return root sessions" }),
    start: z.coerce.number().optional(),
    search: z.string().optional(),
    limit: z.coerce.number().optional(),
  }),
)

// 路径参数验证
validator(
  "param",
  z.object({
    sessionID: Session.get.schema,
  }),
)

// JSON body 验证
validator("json", Session.create.schema.optional())
```

### 统一错误响应格式

```typescript
// error.ts
export const ERRORS = {
  400: {
    description: "Bad request",
    content: {
      "application/json": {
        schema: resolver(
          z
            .object({
              data: z.any(),
              errors: z.array(z.record(z.string(), z.any())),
              success: z.literal(false),
            })
            .meta({ ref: "BadRequestError" }),
        ),
      },
    },
  },
  404: {
    description: "Not found",
    content: {
      "application/json": {
        schema: resolver(NotFoundError.Schema),
      },
    },
  },
} as const

export function errors(...codes: number[]) {
  return Object.fromEntries(codes.map((code) => [code, ERRORS[code as keyof typeof ERRORS]]))
}
```

## 5. 客户端交互典型代码

### 常规 JSON 响应

```typescript
;async (c) => {
  const path = c.req.valid("query").path
  const content = await File.read(path)
  return c.json(content)
}
```

### 流式响应 - Server-Sent Events

```typescript
.get("/event", describeRoute({...}), async (c) => {
  c.header("X-Accel-Buffering", "no")
  c.header("X-Content-Type-Options", "nosniff")
  return streamSSE(c, async (stream) => {
    stream.writeSSE({ data: JSON.stringify({ type: "server.connected", properties: {} }) })

    const unsub = Bus.subscribeAll(async (event) => {
      await stream.writeSSE({ data: JSON.stringify(event) })
      if (event.type === Bus.InstanceDisposed.type) stream.close()
    })

    // 心跳保活
    const heartbeat = setInterval(() => {
      stream.writeSSE({ data: JSON.stringify({ type: "server.heartbeat", properties: {} }) })
    }, 10_000)

    await new Promise<void>((resolve) => {
      stream.onAbort(() => { clearInterval(heartbeat); unsub(); resolve() })
    })
  })
})
```

### 流式响应 - 普通流

```typescript
.post("/:sessionID/message", describeRoute({...}), async (c) => {
  c.status(200)
  c.header("Content-Type", "application/json")
  return stream(c, async (stream) => {
    const msg = await SessionPrompt.prompt({ ...body, sessionID })
    stream.write(JSON.stringify(msg))
  })
})
```

### WebSocket 连接

```typescript
.get("/:ptyID/connect", describeRoute({...}), upgradeWebSocket((c) => {
  const id = c.req.param("ptyID")
  return {
    onOpen(_event, ws) {
      const socket = ws.raw
      handler = Pty.connect(id, socket, cursor)
    },
    onMessage(event) {
      if (typeof event.data !== "string") return
      handler?.onMessage(event.data)
    },
    onClose() { handler?.onClose() },
    onError() { handler?.onClose() },
  }
}))
```

## 6. 项目结构总结

| 目录                        | 说明                                       |
| --------------------------- | ------------------------------------------ |
| `/server/server.ts`         | 主入口，包含全局中间件、路由注册、错误处理 |
| `/server/routes/*.ts`       | 独立路由模块，使用 lazy 延迟加载           |
| `/server/error.ts`          | 统一错误响应格式定义                       |
| `/server/routes/session.ts` | 最完整的 CRUD + 流式示例 (972 行)          |
| `/server/routes/tui.ts`     | TUI 事件交互示例                           |
| `/server/routes/pty.ts`     | WebSocket 连接示例                         |

### 关键依赖

- `hono` - 核心框架
- `hono-openapi` - OpenAPI 3.1 支持 (`describeRoute`, `validator`, `resolver`)
- `zod` - Schema 验证
- `hono/bun` - Bun 运行时适配 (WebSocket, proxy)
- `hono/cors` - CORS 支持
- `hono/streaming` - 流式响应
