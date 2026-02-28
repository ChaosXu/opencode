# Vercel AI SDK 使用分析

## 1. 依赖概览

项目使用 **AI SDK v5** 核心包和多个官方提供商适配器：

```json
{
  "ai": "catalog:",
  "@ai-sdk/provider": "2.0.1",
  "@ai-sdk/anthropic": "2.0.65",
  "@ai-sdk/openai": "2.0.89",
  "@ai-sdk/openai-compatible": "1.0.32",
  "@ai-sdk/azure": "2.0.91",
  "@ai-sdk/google": "2.0.54",
  "@ai-sdk/google-vertex": "3.0.106",
  "@ai-sdk/amazon-bedrock": "3.0.82"
  // ... 其他 12 个提供商
}
```

## 2. 核心 API 使用模式

### streamText - 主要的流式响应调用

```typescript
// src/session/llm.ts
import { streamText, wrapLanguageModel, type ModelMessage, type Tool, tool, jsonSchema } from "ai"

export async function stream(input: StreamInput) {
  return streamText({
    // 模型配置
    model: wrapLanguageModel({
      model: language,
      middleware: [
        {
          async transformParams(args) {
            if (args.type === "stream") {
              args.params.prompt = ProviderTransform.message(args.params.prompt, input.model, options)
            }
            return args.params
          },
        },
      ],
    }),

    // 消息构建
    messages: [...system.map((x): ModelMessage => ({ role: "system", content: x })), ...input.messages],

    // 工具定义
    tools,
    toolChoice: input.toolChoice,
    activeTools: Object.keys(tools).filter((x) => x !== "invalid"),

    // 生成参数
    temperature: params.temperature,
    topP: params.topP,
    topK: params.topK,
    maxOutputTokens,

    // 错误处理和修复
    onError(error) {
      l.error("stream error", { error })
    },
    async experimental_repairToolCall(failed) {
      // 工具调用修复逻辑
    },

    // 遥测
    experimental_telemetry: {
      isEnabled: cfg.experimental?.openTelemetry,
      metadata: { userId, sessionId },
    },

    abortSignal: input.abort,
    maxRetries: input.retries ?? 0,
  })
}
```

## 3. 消息构建 - ModelMessage 转换

```typescript
// src/session/message-v2.ts
import { convertToModelMessages, type ModelMessage, type UIMessage } from "ai"

export function toModelMessages(input: WithParts[], model: Provider.Model): ModelMessage[] {
  const result: UIMessage[] = []

  for (const msg of input) {
    if (msg.info.role === "user") {
      const userMessage: UIMessage = { id: msg.info.id, role: "user", parts: [] }
      for (const part of msg.parts) {
        if (part.type === "text" && !part.ignored) {
          userMessage.parts.push({ type: "text", text: part.text })
        }
        if (part.type === "file" && part.mime !== "text/plain") {
          userMessage.parts.push({ type: "file", url: part.url, mediaType: part.mime })
        }
      }
      result.push(userMessage)
    }

    if (msg.info.role === "assistant") {
      const assistantMessage: UIMessage = { id: msg.info.id, role: "assistant", parts: [] }
      for (const part of msg.parts) {
        if (part.type === "text") {
          assistantMessage.parts.push({ type: "text", text: part.text })
        }
        if (part.type === "tool" && part.state.status === "completed") {
          assistantMessage.parts.push({
            type: `tool-${part.tool}` as `tool-${string}`,
            state: "output-available",
            toolCallId: part.callID,
            input: part.state.input,
            output: part.state.output,
          })
        }
        if (part.type === "reasoning") {
          assistantMessage.parts.push({ type: "reasoning", text: part.text })
        }
      }
      result.push(assistantMessage)
    }
  }

  const tools = Object.fromEntries(toolNames.map((toolName) => [toolName, { toModelOutput }]))
  return convertToModelMessages(result, { tools })
}
```

## 4. 流式响应处理模式

```typescript
// src/session/processor.ts
export function create(input) {
  return {
    async process(streamInput) {
      const stream = await LLM.stream(streamInput)

      for await (const value of stream.fullStream) {
        switch (value.type) {
          case "start":
            // 会话开始
            break

          case "reasoning-start":
            // 推理开始
            break

          case "reasoning-delta":
            // 推理增量
            part.text += value.text
            break

          case "text-start":
            // 文本开始
            currentText = { type: "text", text: "", ... }
            break

          case "text-delta":
            // 文本增量
            currentText.text += value.text
            break

          case "tool-input-start":
            // 工具输入开始
            toolcalls[value.id] = { type: "tool", status: "pending", ... }
            break

          case "tool-call":
            // 工具调用
            toolcalls[value.toolCallId] = { status: "running", input: value.input, ... }
            break

          case "tool-result":
            // 工具结果
            await Session.updatePart({ status: "completed", output: value.output.output, ... })
            break

          case "tool-error":
            // 工具错误
            break

          case "finish-step":
            // 步骤完成
            const usage = Session.getUsage({ model, usage: value.usage })
            break

          case "error":
            throw value.error
        }
      }
    },
  }
}
```

## 5. 工具调用实现

### 工具定义模式

```typescript
// src/tool/tool.ts 和 src/tool/bash.ts
export const Tool = {
  define(id: string, init: InitFn): Tool.Info {
    return {
      id,
      init,
    }
  },
}

// 实际工具示例
export const BashTool = Tool.define("bash", async () => {
  return {
    description: DESCRIPTION,
    parameters: z.object({
      command: z.string().describe("The command to execute"),
      timeout: z.number().describe("Optional timeout").optional(),
      workdir: z.string().describe("Working directory").optional(),
      description: z.string().describe("Command description"),
    }),
    async execute(params, ctx) {
      // 执行逻辑
      return { output, title, metadata }
    },
  }
})
```

### LiteLLM 兼容性处理

```typescript
const isLiteLLMProxy =
  provider.options?.["litellmProxy"] === true || input.model.providerID.toLowerCase().includes("litellm")

if (isLiteLLMProxy && Object.keys(tools).length === 0 && hasToolCalls(input.messages)) {
  tools["_noop"] = tool({
    description: "Placeholder for LiteLLM/Anthropic proxy compatibility",
    inputSchema: jsonSchema({ type: "object", properties: {} }),
    execute: async () => ({ output: "", title: "", metadata: {} }),
  })
}
```

## 6. 模型提供商集成

### 提供商注册表

```typescript
// src/provider/provider.ts
const BUNDLED_PROVIDERS: Record<string, (options) => SDK> = {
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/openai-compatible": createOpenAICompatible,
  "@openrouter/ai-sdk-provider": createOpenRouter,
  // ... 更多
}
```

### 自定义加载器模式

```typescript
const CUSTOM_LOADERS = {
  async anthropic() {
    return {
      autoload: false,
      options: { headers: { "anthropic-beta": "..." } },
    }
  },

  async openai() {
    return {
      autoload: false,
      async getModel(sdk, modelID) {
        return sdk.responses(modelID)
      },
    }
  },

  async github-copilot() {
    return {
      autoload: false,
      async getModel(sdk, modelID) {
        // GPT-5 使用 Responses API，旧模型使用 Chat API
        return shouldUseCopilotResponsesApi(modelID)
          ? sdk.responses(modelID)
          : sdk.chat(modelID)
      },
    }
  },
}
```

## 7. 设计模式总结

| 模式           | 实现位置                  | 说明                                       |
| -------------- | ------------------------- | ------------------------------------------ |
| **流式响应**   | `llm.ts` + `processor.ts` | 使用 `streamText` + `fullStream` 迭代器    |
| **消息转换**   | `message-v2.ts`           | 使用 `convertToModelMessages` 统一消息格式 |
| **工具定义**   | `tool/*.ts`               | 使用 `Tool.define()` 工厂方法              |
| **提供商适配** | `provider.ts`             | bundled providers + custom loaders 双模式  |
| **参数转换**   | `transform.ts`            | 统一参数格式适配不同提供商                 |
| **工具修复**   | `llm.ts`                  | `experimental_repairToolCall` 处理调用错误 |
| **遥测**       | `llm.ts`                  | `experimental_telemetry` 记录使用情况      |

## 8. 典型代码流程

```
用户输入 → SessionPrompt.prompt()
  ↓
创建 User Message
  ↓
LLM.stream() → streamText()
  ↓
for await (value of stream.fullStream)
  ├── text-delta → 更新 TextPart
  ├── reasoning-delta → 更新 ReasoningPart
  ├── tool-call → 创建 ToolPart (running)
  ├── tool-result → 更新 ToolPart (completed)
  └── finish-step → 更新 usage 和成本
  ↓
循环处理直到完成
```
