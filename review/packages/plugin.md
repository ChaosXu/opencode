# Plugin 包分析

## 1. 包概述

**包名**: `@opencode-ai/plugin`
**版本**: 1.2.15
**类型**: 插件开发 SDK
**用途**: 为 OpenCode 开发扩展插件

## 2. 入口点

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./tool": "./src/tool.ts"
  }
}
```

## 3. 导出接口

### 3.1 主入口导出

```typescript
// src/index.ts
export * from "./tool"
export type { Plugin, PluginInput, ProviderContext, AuthHook, Hooks }
```

### 3.2 工具定义

```typescript
// src/tool.ts
export declare const tool: {
  <T extends z.ZodSchema, R>(config: {
    description: string
    args: T
    execute: (args: z.infer<T>, context: ToolContext) => Promise<R>
  }): ToolDefinition
}
```

## 4. 核心类型

### 4.1 Plugin 类型

```typescript
export type Plugin = (input: PluginInput) => Promise<Hooks>
```

### 4.2 PluginInput

```typescript
export type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>
  project: Project
  directory: string
  worktree: string
  serverUrl: URL
  $: BunShell
}
```

### 4.3 Hooks

```typescript
export type Hooks = {
  config?: (config: Config) => Config | Promise<Config>
  tool?: Record<string, ToolDefinition>
  auth?: Record<string, AuthHook>
  event?: (event: Event) => void
  "chat.message"?: (input: string, output: string) => string | Promise<string>
  "tool.execute.before"?: (input: any) => any
  "tool.execute.after"?: (input: any, output: any) => any
  // ... 更多 hooks
}
```

### 4.4 AuthHook

```typescript
export type AuthHook = {
  provider: string
  loader?: (auth: () => Promise<Auth>, provider: Provider) => Promise<Record<string, any>>
  methods: (
    | {
        type: "oauth"
        label: string
        prompts?: [...]
      }
    | {
        type: "api"
        label: string
        prompts: [...]
        authorize: (inputs: Record<string, string>) => Promise<{ type: "success", key: string } | { type: "error", message: string }>
      }
  )[]
}
```

## 5. 工具定义

### 5.1 tool 函数

```typescript
export const tool = {
  description: "工具描述",
  args: z.object({
    param: z.string().describe("参数描述"),
  }),
  async execute(args, context) {
    // 执行逻辑
    return { output, title, metadata }
  },
}
```

### 5.2 ToolContext

```typescript
context: {
  sessionID: string
  messageID: string
  agent: string
  directory: string
  worktree: string
  abort: AbortSignal
  metadata({ title?, metadata? })
  ask({ permission, patterns, always, metadata })
}
```

## 6. Hooks 列表

| Hook                  | 说明           |
| --------------------- | -------------- |
| `config`              | 配置加载时     |
| `tool`                | 注册自定义工具 |
| `auth`                | 自定义认证方式 |
| `event`               | 通用事件       |
| `chat.message`        | 消息接收       |
| `chat.params`         | 修改 LLM 参数  |
| `chat.headers`        | 修改请求头     |
| `tool.execute.before` | 工具执行前     |
| `tool.execute.after`  | 工具执行后     |
| `shell.env`           | 修改环境变量   |
| `permission.ask`      | 权限询问       |
| `tool.definition`     | 修改工具定义   |

## 7. 示例

### 7.1 简单插件

```typescript
import { Plugin, tool } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async (ctx) => {
  return {
    tool: {
      mytool: tool({
        description: "我的工具",
        args: z.object({
          input: z.string(),
        }),
        async execute(args, context) {
          return { output: args.input, title: "", metadata: {} }
        },
      }),
    },
  }
}
```

### 7.2 认证插件

```typescript
export const AuthPlugin: Plugin = async (ctx) => {
  return {
    auth: {
      "my-provider": {
        provider: "my-provider",
        methods: [
          {
            type: "api",
            label: "API Key",
            prompts: [{ type: "text", key: "apiKey", message: "Enter API Key" }],
            authorize: async (inputs) => {
              return { type: "success", key: inputs.apiKey }
            },
          },
        ],
      },
    },
  }
}
```

## 8. 依赖

### 生产依赖

- `@opencode-ai/sdk` - SDK
- `zod` - 验证

### 开发依赖

- `typescript`
- `@types/node`
- `@tsconfig/node22`

## 9. 静态结构

```
src/
├── index.ts        # 主入口，导出类型和 Hooks
├── tool.ts         # 工具定义
├── shell.ts        # Shell 封装
└── example.ts      # 示例
```

## 10. 总结

Plugin 包提供：

1. **类型安全**: 完整的 TypeScript 类型定义
2. **工具系统**: 定义自定义工具的 DSL
3. **认证钩子**: 添加自定义认证方式
4. **生命周期**: 多种 Hook 拦截处理流程
5. **上下文**: 提供丰富的运行时上下文
