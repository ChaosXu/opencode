# Slash 命令实现分析

> 分析时间: 2026-02-28
> 项目: OpenCode

---

## 一、Slash 命令的工作原理

OpenCode 的 slash 命令系统有两种添加方式：

1. **配置文件方式** - 通过 `.md` 文件定义（推荐，最简单）
2. **代码方式** - 硬编码到源码中

配置文件方式搜索以下目录：

- `<project>/.opencode/command/`
- `<project>/.opencode/commands/`
- `<project>/command/`
- `<project>/commands/`

---

### 1.1 完整执行流程（从用户输入到 AI 处理）

```
用户输入 "/command arg1 arg2"
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  前端检测 (prompt-input.tsx)                                 │
│  - 检测以 "/" 开头的输入                                    │
│  - 解析命令名和参数                                          │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  SDK 发送请求 (submit.ts)                                   │
│  - client.session.command({ sessionID, command, arguments })│
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  API 路由处理 (session.ts)                                   │
│  - POST /:sessionID/command                                │
│  - 验证 CommandInput                                        │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  命令执行 (prompt.ts command 函数)                           │
│  - 获取命令配置                                             │
│  - 解析参数并替换模板占位符                                  │
│  - 执行 shell 命令（如果有）                                 │
│  - 调用 prompt() 进行 AI 处理                               │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 前端命令检测与提交

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

### 1.3 API 路由处理

**文件**: `packages/opencode/src/server/routes/session.ts`

```typescript
.post(
  "/:sessionID/command",
  validator("json", SessionPrompt.CommandInput.omit({ sessionID: true })),
  async (c) => {
    const sessionID = c.req.valid("param").sessionID
    const body = c.req.valid("json")
    const msg = await SessionPrompt.command({ ...body, sessionID })
    return c.json(msg)
  }
)
```

### 1.4 CommandInput Schema

**文件**: `packages/opencode/src/session/prompt.ts`

```typescript
export const CommandInput = z.object({
  sessionID: Identifier.schema("session"),
  command: z.string(),
  arguments: z.string(),
  agent: z.string().optional(),
  model: ModelId.optional(),
  variant: z.string().optional(),
  parts: z.array(Part).optional(),
  messageID: Identifier.schema("message"),
})
```

### 1.5 命令执行核心逻辑

**文件**: `packages/opencode/src/session/prompt.ts` (command 函数)

```typescript
export async function command(input: CommandInput) {
  // 1. 获取命令配置
  const command = await Command.get(input.command)
  
  // 2. 解析参数
  const raw = input.arguments.match(argsRegex) ?? []
  const args = raw.map((arg) => arg.replace(quoteTrimRegex, ""))
  
  // 3. 替换模板占位符
  let template = withArgs.replaceAll("$ARGUMENTS", input.arguments)
  
  // 4. 执行 shell 命令（如 `!`echo hello``）
  if (shell.length > 0) {
    const results = await Promise.all(...)
    template = template.replace(bashRegex, () => results[index++])
  }
  
  // 5. 确定使用 subtask 还是普通消息
  const isSubtask = (agent.mode === "subagent" && command.subtask !== false) || command.subtask === true
  
  // 6. 调用 prompt() 执行 AI 处理
  const result = await prompt({
    sessionID: input.sessionID,
    messageID: input.messageID,
    model: userModel,
    agent: userAgent,
    parts: isSubtask ? subtaskParts : templateParts,
    variant: input.variant,
  })
  
  // 7. 发布命令执行事件
  Bus.publish(Command.Event.Executed, { name: input.command, ... })
  
  return result
}
```

### 1.6 参数处理与模板替换

**支持的占位符**：

| 占位符 | 说明 | 示例 |
|--------|------|------|
| `$ARGUMENTS` | 用户输入的全部参数 | `/commit fix bug` → `$ARGUMENTS` = `fix bug` |
| `$1`, `$2`, `$3`... | 位置参数 | MCP prompts 支持 |
| `!`bash`code`` | Shell 执行 | `!`echo hello`` → 执行并替换为结果 |

**替换逻辑**：

```typescript
// $ARGUMENTS 替换
template = template.replaceAll("$ARGUMENTS", input.arguments)

// Shell 命令执行
const bashRegex = /!`([^`]+)`/g
template = template.replace(bashRegex, async (_, cmd) => {
  const result = await Bun.spawn(['sh', '-c', cmd]).text()
  return result
})
```

### 1.7 前端命令注册

**文件**: `packages/app/src/pages/session/use-session-commands.tsx`

```typescript
// 注册命令，带有 slash 属性表示可以通过 "/" 触发
command.register("session", () => [
  sessionCommand({
    id: "session.new",
    slash: "new",  // 用户输入 "/new" 触发
    onSelect: () => navigate(`/${params.dir}/session`),
  }),
])
```

**CommandOption 结构**：

```typescript
interface CommandOption {
  id: string
  title: string
  description?: string
  category?: string
  keybind?: string
  slash?: string        // "/" 触发关键字
  suggested?: boolean
  disabled?: boolean
  onSelect?: (source: "palette" | "keybind" | "slash") => void
}
```

## 二、核心文件位置

### 1. 命令定义与注册

| 文件                                     | 作用                          |
| ---------------------------------------- | ----------------------------- |
| `packages/opencode/src/command/index.ts` | 命令的核心定义、注册、获取    |
| `packages/opencode/src/config/config.ts` | 命令配置加载、Zod schema 定义 |

### 2. 命令执行

| 文件                                             | 作用                                 |
| ------------------------------------------------ | ------------------------------------ |
| `packages/opencode/src/session/prompt.ts`        | 命令执行逻辑 (SessionPrompt.command) |
| `packages/opencode/src/server/routes/session.ts` | API 路由处理                         |

### 3. 前端触发

| 文件                                      | 作用               |
| ----------------------------------------- | ------------------ |
| `packages/app/src/routes/chat/prompt.tsx` | TUI 中命令输入处理 |

---

## 三、核心代码结构

### 1. 命令类型定义 (command/index.ts)

```typescript
export const Info = z.object({
  name: z.string(),
  description: z.string().optional(),
  agent: z.string().optional(),
  model: z.string().optional(),
  source: z.enum(["command", "mcp", "skill"]).optional(),
  template: z.promise(z.string()).or(z.string()),
  subtask: z.boolean().optional(),
  hints: z.array(z.string()),
})
```

### 2. 命令配置 Schema (config/config.ts)

```typescript
export const Command = z.object({
  template: z.string(),
  description: z.string().optional(),
  agent: z.string().optional(),
  model: ModelId.optional(),
  subtask: z.boolean().optional(),
})
```

### 3. 命令加载逻辑

```typescript
async function loadCommand(dir: string) {
  const result: Record<string, Command> = {}
  for (const item of await Glob.scan("{command,commands}/**/*.md", {
    cwd: dir,
    absolute: true,
    dot: true,
    symlink: true,
  })) {
    const md = await ConfigMarkdown.parse(item)
    const name = path.basename(item, ".md")
    const config = {
      name,
      ...md.data,
      template: md.content.trim(),
    }
    result[config.name] = Command.safeParse(config).data
  }
  return result
}
```

### 4. 命令执行逻辑 (session/prompt.ts)

```typescript
const command = await Command.get(input.command)
const prompt = await command.template
await SessionPrompt.command({
  session,
  prompt,
  messageID,
  arguments: input.arguments,
})
```

---

## 四、添加 Slash 命令的方法

### 方法一：配置文件方式（推荐）

1. 在项目根目录创建 `.opencode/commands/` 目录
2. 创建 `命令名.md` 文件
3. 编写 frontmatter 和 prompt 模板

示例：

```markdown
---
description: 这是一个示例命令
agent: build
model: anthropic/claude-sonnet-4-20250514
subtask: false
---

请执行以下操作：

1. 第一步操作说明
2. 第二步操作说明

$ARGUMENTS
```

**支持的特殊变量**：

- `$ARGUMENTS` - 用户在命令后面输入的全部参数
- `$1`, `$2`, `$3`... - 位置参数（通过 MCP prompts 支持）

### 方法二：代码方式（内置命令）

修改 `packages/opencode/src/command/index.ts`：

1. 在 `Default` 中添加命令名
2. 在 `state` 中注册命令

```typescript
export const Default = {
  INIT: "init",
  REVIEW: "review",
  MY_COMMAND: "my_command",
} as const

// 在 state 中
[Default.MY_COMMAND]: {
  name: Default.MY_COMMAND,
  description: "我的自定义命令描述",
  source: "command",
  get template() {
    return "这里是指令模板内容..."
  },
  hints: ["$ARGUMENTS"],
},
```

---

## 五、测试与验证

### 1. 单元测试

```bash
cd packages/opencode
bun test
```

### 2. 调试命令加载

```bash
# 查看加载的配置
opencode debug config

# 列出可用命令
opencode --list-commands
```

### 3. 手动测试

在 TUI 中直接输入：

```
/your-command arg1 arg2
```

---

## 六、构建与打包

### 1. 本地构建

```bash
cd packages/opencode
bun run build
```

### 2. 测试本地版本

```bash
# 创建软链接
npm link

# 或直接运行
bun run dev -- your-command
```

---

## 七、注意事项

1. **命令名称**：建议使用小写和下划线
2. **模板内容**：必须是有效的 prompt
3. **参数处理**：使用 `$ARGUMENTS` 捕获用户输入
4. **调试**：使用 `opencode debug config` 查看命令是否正确加载
5. **优先级**：配置文件的命令会覆盖内置同名命令

---

## 八、相关资源

- 命令配置文档: https://opencode.ai/docs/commands
- 配置文件格式: Zod schema 定义在 `config/config.ts`
- 模板变量: `hints()` 函数解析 `$1`, `$2` 等占位符
