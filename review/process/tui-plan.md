# OpenCode TUI Plan 模式实现分析

> 分析时间: 2026-06-16
> 范围: TUI 中 Plan 模式的完整实现 —— agent 定义、入口、执行流程、写操作禁止机制
>
> 路径以 `packages/` 为根。

---

## 一、核心结论

**Plan 模式是一个原生 Agent**（非独立模块），与 `build`/`general`/`explore` 等并列定义。它通过三个机制联合控制行为：

| 机制 | 作用 | 文件 |
|---|---|---|
| **权限规则静态过滤工具** | 把 `edit`/`write`/`apply_patch` 从 LLM 工具列表中剔除 | `packages/opencode/src/permission/index.ts:215-224` |
| **每回合注入 system-reminder** | 在 user 消息上追加 synthetic text part 传达只读约束 | `packages/opencode/src/session/reminders.ts:15-90` |
| **`plan_exit` 工具切换 agent** | 写入 `agent: "build"` 的新 user 消息切换回 build | `packages/opencode/src/tool/plan.ts:15-79` |

**关键事实**：
- plan agent 没有 `prompt` 字段 → 沿用 provider 默认系统提示（与 build 完全相同）
- plan agent 没有 `model` 字段 → 沿用会话当前模型
- 每个 user 消息独立记录 `agent` 字段；切换 agent 只影响后续新消息

---

## 二、Agent 定义

`packages/opencode/src/agent/agent.ts:154-179`：

```ts
plan: {
  name: "plan",
  description: "Plan mode. Disallows all edit tools.",
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",                                  // 覆盖默认 deny
      plan_exit: "allow",                                 // 覆盖默认 deny
      task: { general: "deny" },                          // 不能调 general 子 agent
      external_directory: { [data/plans/*]: "allow" },
      edit: {
        "*": "deny",                                      // 静态过滤命中
        ".opencode/plans/*.md": "allow",                  // plan 文件白名单
        [data/plans/*.md]: "allow",
      },
    }),
    user,
  ),
  mode: "primary",
  native: true,
},
```

### 2.1 默认权限

`packages/opencode/src/agent/agent.ts:117-134`：

```ts
const defaults = Permission.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: { "*": "ask", ...whitelistedDirs },
  question: "deny",
  plan_enter: "deny",
  plan_exit: "deny",
  read: {
    "*": "allow",
    "*.env": "ask",
    "*.env.*": "ask",
    "*.env.example": "allow",
  },
})
```

### 2.2 对照：build agent

`packages/opencode/src/agent/agent.ts:139-153` —— build agent 仅比 defaults 多两条：

```ts
build: {
  ...,
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",
      plan_enter: "allow",     // ← 允许 plan_enter，但后端未实现该工具
    }),
    user,
  ),
  mode: "primary",
  native: true,
}
```

build 没有覆盖 `edit` 权限，所以默认 `"*": "allow"` 直接生效，写工具可用。

---

## 三、入口与切换

TUI 中存在 **4 种触发方式**，所有路径最终都只更新 `local.agent.current()`（Solid store），不发任何网络请求：

### 3.1 入口列表

| 入口 | 代码位置 |
|---|---|
| `Tab` / `Shift+Tab` 循环 | `packages/tui/src/config/keybind.ts:127-128` → `packages/tui/src/app.tsx:687,721` → `local.agent.move(±1)` |
| `<leader>a` / `/agents` | `packages/tui/src/app.tsx:664-671` → `packages/tui/src/component/dialog-agent.tsx:6-31` |
| CLI `--agent plan` | `packages/opencode/src/cli/cmd/tui.ts:206` → `packages/tui/src/app.tsx:467` |
| 切会话恢复 | `packages/tui/src/component/prompt/index.tsx:306-328`（从最后一条 user 消息的 `agent` 字段恢复；除非 CLI 显式传了 `--agent`） |

### 3.2 local.agent 实现

`packages/tui/src/context/local.tsx:73-129`：

```ts
function createAgent() {
  const agents = createMemo(() => sync.data.agent.filter(a => a.mode !== "subagent" && !a.hidden))
  return {
    list()    { return agents() },
    current() { return agents().find(x => x.name === agentStore.current) ?? agents().at(0) },
    set(name) { setAgentStore("current", name) },
    move(direction) {
      const current = this.current()
      if (!current) return
      let next = agents().findIndex(x => x.name === current.name) + direction
      if (next < 0) next = agents().length - 1
      if (next >= agents().length) next = 0
      setAgentStore("current", agents()[next].name)
    },
  }
}
```

### 3.3 提交消息时附带 agent

`packages/tui/src/component/prompt/index.tsx`：
- 新建会话（`:994-997`）：`sdk.client.session.create({ agent: agent.name, ... })`
- 后续 prompt（`:1086-1093`）：`sdk.client.session.prompt({ ..., agent: agent.name, ... })`
- 斜杠命令（`:1077-1085`）：`session.command({ ..., agent: agent.name, ... })`

---

## 四、服务端执行流程

### 4.1 阶段总览

```
[1] TUI 提交
    └─► sdk.session.prompt({ agent: "plan", parts })
         │
[2] HTTP handler
    └─► handlers/session.ts:293
         │
[3] SessionPrompt.prompt (prompt.ts:1105)
    ├─ createUserMessage (prompt.ts:636)
    │   ├─ agents.get("plan") → agent.ts:154
    │   ├─ persist SessionV1.User { agent: "plan" }
    │   └─ publish AgentSwitched 事件
    │
    └─ loop(sessionID) → runLoop
         │
[4] runLoop (prompt.ts:1134, while true)
    ├─ agents.get(lastUser.agent) → plan
    ├─ SessionReminders.apply   → 追加 synthetic text part
    ├─ assistant msg { agent: "plan" }
    ├─ SessionTools.resolve     → 包装所有工具（含 plan_exit）
    ├─ LLMRequestPrep.prepare
    │   ├─ system = SystemPrompt.provider(model)   (与 build 相同)
    │   └─ resolveTools → Permission.disabled 剔除 edit/write/apply_patch
    │
    └─ ai.streamText({ system, messages, tools })
         │
    ┌────┴────┐
    ▼         ▼
  工具调用  plan_exit 调用
    │         │
    ▼         ├─ No ─► RejectedError → 继续 plan
  继续循环   │
              └─ Yes
                 ├─ question.ask(用户)
                 └─ 创建新 User { agent: "build" } + 合成 text
                      │
                      ▼
                  runLoop 下一轮
                  agents.get("build") → 完整工具可用

TUI 并行：
  event.on("message.part.updated", tool=plan_exit & completed)
       └─► local.agent.set("build")
  AgentSwitched 事件
       └─► transcript 顶部插入 "agent-switched" 消息
```

### 4.2 `SessionPrompt.prompt`

`packages/opencode/src/session/prompt.ts:1105-1124`：

```ts
const prompt: (input: PromptInput) => Effect.Effect<SessionV1.WithParts, Image.Error> = Effect.fn(
  "SessionPrompt.prompt",
)(function* (input: PromptInput) {
  const session = yield* sessions.get(input.sessionID).pipe(Effect.orDie)
  yield* revert.cleanup(session)
  const message = yield* createUserMessage(input)         // ← agent 在这里落库
  yield* sessions.touch(input.sessionID)

  const permissions: PermissionV1.Rule[] = []
  for (const [t, enabled] of Object.entries(input.tools ?? {})) {
    permissions.push({ permission: t, action: enabled ? "allow" : "deny", pattern: "*" })
  }
  if (permissions.length > 0) {
    session.permission = permissions
    yield* sessions.setPermission({ sessionID: session.id, permission: permissions })
  }

  if (input.noReply === true) return message
  return yield* loop({ sessionID: input.sessionID })
})
```

### 4.3 `createUserMessage`

`packages/opencode/src/session/prompt.ts:636-686`：

```ts
const createUserMessage = Effect.fn("SessionPrompt.createUserMessage")(function* (input: PromptInput) {
  const agentName = input.agent
  const ag = agentName ? yield* agents.get(agentName) : yield* agents.defaultInfo()
  // ...
  const model = input.model ?? ag.model ?? (yield* currentModel(input.sessionID))
  // ...

  const info: SessionV1.User = {
    id: input.messageID ?? MessageID.ascending(),
    role: "user",
    sessionID: input.sessionID,
    time: { created: Date.now() },
    tools: input.tools,
    agent: ag.name,                                        // ← 关键：agent 落库
    model: { providerID: model.providerID, modelID: model.modelID, variant },
    system: input.system,
    format: input.format,
  }

  if (current?.agent !== info.agent) {
    yield* events.publish(SessionEvent.AgentSwitched, {
      sessionID: input.sessionID,
      messageID: SessionMessage.ID.create(),
      timestamp: DateTime.makeUnsafe(info.time.created),
      agent: info.agent,
    })
  }
  // ...
})
```

### 4.4 `runLoop`

`packages/opencode/src/session/prompt.ts:1134-1254`：

```ts
const runLoop: (sessionID: SessionID) => Effect.Effect<SessionV1.WithParts> = Effect.fn("SessionPrompt.run")(
  function* (sessionID: SessionID) {
    const ctx = yield* InstanceState.context
    let structured: unknown
    let step = 0
    const session = yield* sessions.get(sessionID).pipe(Effect.orDie)

    while (true) {
      yield* status.set(sessionID, { type: "busy" })
      yield* Effect.logInfo("loop", { "session.id": sessionID, step })

      let msgs = yield* MessageV2.filterCompactedEffect(sessionID)
      const { user: lastUser, assistant: lastAssistant, finished: lastFinished, tasks } = MessageV2.latest(msgs)
      // ...退出口判定...

      step++
      const model = yield* getModel(lastUser.model.providerID, lastUser.model.modelID, sessionID)
      const task = tasks.pop()

      if (task?.type === "subtask") {
        yield* handleSubtask({ task, model, lastUser, sessionID, session, msgs })
        continue
      }
      // ...compaction 检查...

      const agent = yield* agents.get(lastUser.agent)        // ← 从最新 user 读取
      // ...
      const maxSteps = agent.steps ?? Infinity
      const isLastStep = step >= maxSteps

      // ← 核心：注入 plan reminder
      msgs = yield* SessionReminders.apply({ messages: msgs, agent, session }).pipe(
        Effect.provideService(RuntimeFlags.Service, flags),
        Effect.provideService(FSUtil.Service, fsys),
        Effect.provideService(Session.Service, sessions),
      )

      const msg: SessionV1.Assistant = {
        id: MessageID.ascending(),
        parentID: lastUser.id,
        role: "assistant",
        mode: agent.name,
        agent: agent.name,                                   // ← assistant 也记录 agent
        // ...
      }
      yield* sessions.updateMessage(msg)
      // ...processor + tools + LLM call...
    }
  },
)
```

---

## 五、`SessionReminders.apply` —— Plan 行为的关键中枢

`packages/opencode/src/session/reminders.ts:15-90` 在每轮循环中向最新 user 消息**追加 `synthetic: true` 的 text part**，把"计划阶段"约束以 system-reminder 形式塞进对话上下文。

### 5.1 默认模式

```ts
if (!flags.experimentalPlanMode) {
  if (input.agent.name === "plan") {
    userMessage.parts.push({
      id: PartID.ascending(),
      messageID: userMessage.info.id,
      sessionID: userMessage.info.sessionID,
      type: "text",
      text: PROMPT_PLAN,
      synthetic: true,
    })
  }
  const wasPlan = input.messages.some((msg) => msg.info.role === "assistant" && msg.info.agent === "plan")
  if (wasPlan && input.agent.name === "build") {
    userMessage.parts.push({
      // ...synthetic text part
      text: BUILD_SWITCH,
    })
  }
  return input.messages
}
```

### 5.2 实验模式（`OPENCODE_EXPERIMENTAL_PLAN_MODE=true`）

```ts
const assistantMessage = input.messages.findLast((msg) => msg.info.role === "assistant")

// 退出 plan
if (input.agent.name !== "plan" && assistantMessage?.info.agent === "plan") {
  const ctx = yield* InstanceState.context
  const plan = Session.plan(input.session, ctx)
  const exists = yield* fsys.existsSafe(plan)
  const part = yield* sessions.updatePart({
    // ...
    text: exists
      ? `${BUILD_SWITCH}\n\nA plan file exists at ${plan}. You should execute on the plan defined within it`
      : BUILD_SWITCH,
    synthetic: true,
  })
  userMessage.parts.push(part)
  return input.messages
}

// 进入/保持 plan
if (input.agent.name !== "plan" || assistantMessage?.info.agent === "plan") return input.messages

const ctx = yield* InstanceState.context
const plan = Session.plan(input.session, ctx)
const exists = yield* fsys.existsSafe(plan)
if (!exists) yield* fsys.ensureDir(path.dirname(plan)).pipe(Effect.catch(Effect.die))
const part = yield* sessions.updatePart({
  // ...
  text: PLAN_MODE.replace("${planInfo}", () =>
    exists
      ? `A plan file already exists at ${plan}. You can read it and make incremental edits using the edit tool.`
      : `No plan file exists yet. You should create your plan at ${plan} using the write tool.`,
  ),
  synthetic: true,
})
userMessage.parts.push(part)
return input.messages
```

### 5.3 模板内容

#### `prompt/plan.txt`（默认，26 行）

```
<system-reminder>
# Plan Mode - System Reminder

CRITICAL: Plan mode ACTIVE - you are in READ-ONLY phase. STRICTLY FORBIDDEN:
ANY file edits, modifications, or system changes. Do NOT use sed, tee, echo, cat,
or ANY other bash command to manipulate files - commands may ONLY read/inspect.
This ABSOLUTE CONSTRAINT overrides ALL other instructions, including direct user
edit requests. You may ONLY observe, analyze, and plan. Any modification attempt
is a critical violation. ZERO exceptions.

---
## Responsibility
Your current responsibility is to think, read, search, and delegate explore agents
to construct a well-formed plan...
```

#### `prompt/plan-mode.txt`（实验模式，70 行 —— 完整 5 阶段工作流）

```
<system-reminder>
Plan mode is active. The user indicated that they do not want you to execute yet --
you MUST NOT make any edits (with the exception of the plan file mentioned below),
run any non-readonly tools...

## Plan File Info:
${planInfo}

### Phase 1: Initial Understanding
Goal: Gain a comprehensive understanding of the user's request by reading through
code and asking them questions. Critical: In this phase you should only use the
explore subagent type.
1. Launch up to 3 explore agents IN PARALLEL...

### Phase 2: Design
Launch general agent(s) to design the implementation...

### Phase 3: Review
Read the critical files identified by agents to deepen your understanding...

### Phase 4: Final Plan
Write your final plan to the plan file (the only file you can edit).

### Phase 5: Call plan_exit tool
At the very end of your turn, once you have asked the user questions and are happy
with your final plan file - you should always call plan_exit to indicate to the user
that you are done planning.
</system-reminder>
```

#### `prompt/build-switch.txt`（5 行）

```
<system-reminder>
Your operational mode has changed from plan to build.
You are no longer in read-only mode.
You are permitted to make file changes, run shell commands, and utilize your arsenal
of tools as needed.
</system-reminder>
```

---

## 六、Plan 文件路径

`packages/opencode/src/session/session.ts:377-382`：

```ts
export function plan(input: { slug: string; time: { created: number } }, instance: InstanceContext) {
  const base = instance.project.vcs
    ? path.join(instance.worktree, ".opencode", "plans")
    : path.join(Global.Path.data, "plans")
  return path.join(base, [input.time.created, input.slug].join("-") + ".md")
}
```

这恰好对应 `agent.ts:166-173` 中白名单放行的两条路径：

```ts
edit: {
  "*": "deny",
  [path.join(".opencode", "plans", "*.md")]: "allow",
  [path.relative(ctx.worktree, path.join(Global.Path.data, path.join("plans", "*.md")))]: "allow",
},
```

⚠️ 但需要注意的是：plan 文件白名单**仅影响运行时 `Permission.ask` 的 pattern 匹配**，**不影响 `Permission.disabled` 的静态过滤**。详见第八节。

---

## 七、工具装配

### 7.1 注册（不区分 agent）

`packages/opencode/src/tool/registry.ts:217-239`：所有原生工具都无条件注册到 `builtin` 列表，**唯一例外**：

```ts
...(flags.experimentalPlanMode && flags.client === "cli" ? [tool.plan] : [])  // :235
```

`plan_exit` 工具**只在 CLI 客户端且实验 flag 开启时**才会出现在工具集里。这意味着 desktop/app 客户端下模型拿不到 `plan_exit`，必须由用户手动在 picker 里切回 build。

### 7.2 `SessionTools.resolve`

`packages/opencode/src/session/tools.ts:24-205`：每轮循环按 `(model, providerID, agent)` 调 `registry.tools(...)`，把每个工具包成 AI SDK `tool()`：

```ts
for (const item of yield* registry.tools({
  modelID: ModelV2.ID.make(input.model.api.id),
  providerID: input.model.providerID,
  agent: input.agent,
})) {
  const schema = ProviderTransform.schema(input.model, ToolJsonSchema.fromTool(item))
  tools[item.id] = tool({
    description: item.description,
    inputSchema: jsonSchema(schema),
    execute(args, options) {
      return run.promise(
        Effect.gen(function* () {
          const ctx = context(args, options)
          // ...
          const result = yield* item.execute(args, ctx)
          // ...
        }),
      )
    },
  })
}
```

`context.ask`（`:63-71`）把 `agent.permission` 与 `session.permission` 合并作为 ruleset 传给 `Permission.ask`：

```ts
ask: (req) =>
  permission
    .ask({
      ...req,
      sessionID: input.session.id,
      tool: { messageID: input.processor.message.id, callID: options.toolCallId },
      ruleset: Permission.merge(input.agent.permission, input.session.permission ?? []),
    })
    .pipe(Effect.orDie),
```

**注意**：`SessionTools.resolve` 本身**不过滤** edit 类工具，过滤发生在 `LLMRequestPrep.prepare`。

### 7.3 关键过滤 —— `LLMRequestPrep.resolveTools`

`packages/opencode/src/session/llm/request.ts:198-204`：

```ts
function resolveTools(input: Pick<PrepareInput, "tools" | "agent" | "permission" | "user">) {
  const disabled = Permission.disabled(
    Object.keys(input.tools),
    Permission.merge(input.agent.permission, input.permission ?? []),
  )
  return Record.filter(input.tools, (_, k) => input.user.tools?.[k] !== false && !disabled.has(k))
}
```

### 7.4 `Permission.disabled`

`packages/opencode/src/permission/index.ts:215-224`：

```ts
export function disabled(tools: string[], ruleset: PermissionV1.Ruleset): Set<string> {
  const edits = ["edit", "write", "apply_patch"]
  return new Set(
    tools.filter((tool) => {
      const permission = edits.includes(tool) ? "edit" : tool
      const rule = ruleset.findLast((rule) => Wildcard.match(permission, r.permission))
      return rule?.pattern === "*" && rule.action === "deny"
    }),
  )
}
```

由于 plan agent 规则中存在 `edit: { "*": "deny", ...白名单 }`，`Permission.disabled` 把 `edit`、`write`、`apply_patch` 三个工具**直接过滤掉**，最终发给 AI SDK 的 function-call schema 里**根本没有这三个工具名**。

---

## 八、写操作禁止机制（核心问题详解）

由 **两层强制 + 一层提示** 实现。

### 8.1 Layer 1：静态剔除（最强，100% 阻断）

**位置 1**：`packages/opencode/src/permission/index.ts:215-224` `Permission.disabled`

```ts
const edits = ["edit", "write", "apply_patch"]
// 三个写工具都映射到 "edit" 权限查询
const rule = ruleset.findLast((rule) => Wildcard.match(permission, rule.permission))
return rule?.pattern === "*" && rule.action === "deny"
```

**位置 2**：`packages/opencode/src/session/llm/request.ts:198-204` `resolveTools`

```ts
const disabled = Permission.disabled(
  Object.keys(input.tools),
  Permission.merge(input.agent.permission, input.permission ?? []),
)
return Record.filter(input.tools, (_, k) => input.user.tools?.[k] !== false && !disabled.has(k))
```

**位置 3**：plan agent 的规则（`packages/opencode/src/agent/agent.ts:169-173`）

```ts
edit: {
  "*": "deny",                                           // ← 命中 disabled
  ".opencode/plans/*.md": "allow",                       // ← 仅影响运行时 ask
  [data/plans/*.md]: "allow",
},
```

**调用链**：

```
LLMRequestPrep.prepare()        request.ts:56
  └─ resolveTools()              request.ts:198
       └─ Permission.disabled()  permission/index.ts:215
            └─ ruleset.findLast → 匹配 "edit" + pattern "*" + action "deny"
       └─ disabled = Set { "edit", "write", "apply_patch" }
       └─ Record.filter → tools map 中剔除三个
       └─ return 精简后的 tools
  └─ 传给 ai.streamText({ ..., tools })
       └─ LLM 的 function-call schema 不含这三个工具
```

**效果**：LLM 不可能调用它看不见的工具 —— 这是最强的强制。

### 8.2 Layer 2：运行时拦截（次强）

`packages/opencode/src/permission/index.ts:78-118` `Permission.ask`：

```ts
for (const pattern of request.patterns) {
  const rule = evaluate(request.permission, pattern, ruleset, approved)  // :84
  yield* Effect.logInfo("evaluated", { permission: request.permission, pattern, action: rule })
  if (rule.action === "deny") {
    return yield* new PermissionV1.DeniedError({
      ruleset: ruleset.filter((rule) => Wildcard.match(request.permission, rule.permission)),
    })
  }
  if (rule.action === "allow") continue
  needsAsk = true
}
if (!needsAsk) return
// ...发 permission.asked 事件 + Deferred.await
```

由 `SessionTools.resolve` 的 `context.ask`（`packages/opencode/src/session/tools.ts:63-71`）在每次工具执行前调用。

**注意**：这一层对走 `bash`/`write` 工具的情况防御相对薄弱：
- 走 `write` 工具：`evaluate("write", ...)` 查 `write` 权限，**默认规则里没有显式 `write` 条目**，会落到 defaults 的 `"*": "allow"` 兜底
- 走 `bash` 工具用 `>`/`tee` 改文件：`evaluate("bash", "src/foo.ts", ruleset)` 命中 defaults `"*": "allow"`

### 8.3 Layer 3：system-reminder 提示（最弱，靠模型守规矩）

`packages/opencode/src/session/reminders.ts:27-36` 每回合追加 synthetic text part：

`prompt/plan.txt:4-9`：
```
CRITICAL: Plan mode ACTIVE - you are in READ-ONLY phase. STRICTLY FORBIDDEN:
ANY file edits, modifications, or system changes. Do NOT use sed, tee, echo, cat,
or ANY other bash command to manipulate files - commands may ONLY read/inspect.
```

`prompt/plan-mode.txt:2`：
```
you MUST NOT make any edits (with the exception of the plan file mentioned below),
run any non-readonly tools (including changing configs or making commits)
```

### 8.4 三层对比

| 层面 | 工具对 LLM 可见 | 强制力 |
|---|---|---|
| Layer 1 静态剔除 | ❌ 不可见 | 100% 阻断 |
| Layer 2 运行时拦截 | ✅ 可见但被 deny | 接近 100%（依赖规则正确） |
| Layer 3 提示 | ✅ 可见 | 0%（靠模型遵守） |

**核心防御是 Layer 1**：plan agent 的 `edit: { "*": "deny" }` 规则在工具下发前就过滤掉了 `edit`/`write`/`apply_patch`，模型完全不知道这些工具存在。Layer 2 是兜底（防御 provider bug 或上下文混乱），Layer 3 是 prompt 引导。

---

## 九、`plan_exit` 工具

`packages/opencode/src/tool/plan.ts:15-79`：

```ts
export const PlanExitTool = Tool.define(
  "plan_exit",
  Effect.gen(function* () {
    const session = yield* Session.Service
    const question = yield* Question.Service
    const provider = yield* Provider.Service

    return {
      description: EXIT_DESCRIPTION,
      parameters: Parameters,
      execute: (_params: {}, ctx: Tool.Context) =>
        Effect.gen(function* () {
          const instance = yield* InstanceState.context
          const info = yield* session.get(ctx.sessionID)
          const plan = path.relative(instance.worktree, Session.plan(info, instance))

          const answers = yield* question.ask({
            sessionID: ctx.sessionID,
            questions: [
              {
                question: `Plan at ${plan} is complete. Would you like to switch to the build agent and start implementing?`,
                header: "Build Agent",
                custom: false,
                options: [
                  { label: "Yes", description: "Switch to build agent and start implementing the plan" },
                  { label: "No", description: "Stay with plan agent to continue refining the plan" },
                ],
              },
            ],
            tool: ctx.callID ? { messageID: ctx.messageID, callID: ctx.callID } : undefined,
          })

          if (answers[0]?.[0] === "No") yield* new Question.RejectedError()

          const messages = yield* session.messages({ sessionID: ctx.sessionID }).pipe(Effect.orDie)
          const lastUser = messages.findLast((item) => item.info.role === "user" && item.info.model)
          const model =
            lastUser?.info.role === "user" && lastUser.info.model ? lastUser.info.model : yield* provider.defaultModel()

          // 关键：写入新 user 消息，agent 切到 build
          const msg: SessionV1.User = {
            id: MessageID.ascending(),
            sessionID: ctx.sessionID,
            role: "user",
            time: { created: Date.now() },
            agent: "build",
            model,
          }
          yield* session.updateMessage(msg)
          yield* session.updatePart({
            id: PartID.ascending(),
            messageID: msg.id,
            sessionID: ctx.sessionID,
            type: "text",
            text: `The plan at ${plan} has been approved, you can now edit files. Execute the plan`,
            synthetic: true,
          } satisfies SessionV1.TextPart)

          return {
            title: "Switching to build agent",
            output: "User approved switching to build agent. Wait for further instructions.",
            metadata: {},
          }
        }).pipe(Effect.orDie),
    }
  }),
)
```

### 9.1 `plan_exit` 的核心机制

- **不直接修改前端 picker**，而是通过**在数据库里插入一条新的 user 消息**（`agent: "build"`），让下一轮 `runLoop` 自然读到新 agent
- 询问用户是否切换；选 `No` 则抛 `Question.RejectedError`，plan agent 继续
- 继承上一条带 model 的 user 消息的 model，否则用 `provider.defaultModel()`

### 9.2 TUI 监听工具完成事件

`packages/tui/src/routes/session/index.tsx:319-334`：

```ts
let lastSwitch: string | undefined = undefined
event.on("message.part.updated", (evt) => {
  const part = evt.properties.part
  if (part.type !== "tool") return
  if (part.sessionID !== route.sessionID) return
  if (part.state.status !== "completed") return
  if (part.id === lastSwitch) return

  if (part.tool === "plan_exit") {
    local.agent.set("build")
    lastSwitch = part.id
  } else if (part.tool === "plan_enter") {
    local.agent.set("plan")
    lastSwitch = part.id
  }
})
```

这是**前端 UX 同步**：plan_exit 完成后立刻把 picker 翻到 build，让用户下一条消息默认就是 build agent。注意这与后端状态一致 —— 后端写了一条 `agent: "build"` 的新 user 消息，前端更新本地 store。

---

## 十、系统提示与模型选择

### 10.1 System prompt 拼装

`packages/opencode/src/session/llm/request.ts:58-66`：

```ts
const system = [
  [
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    ...input.system,
    ...(input.user.system ? [input.user.system] : []),
  ]
    .filter((x) => x)
    .join("\n"),
]
```

plan agent 没有 `prompt` → 走 `SystemPrompt.provider(model)`（按模型选 anthropic/gpt/gemini/beast 等 default prompt，详见 `packages/opencode/src/session/system.ts:25-39`），**与 build 完全相同**。

**Plan 行为不在 system prompt 里**，全靠 `SessionReminders` 注入到 user 消息。

### 10.2 模型选择

plan agent 没有 `model` 字段 → 沿用 `currentModel(sessionID)`（`prompt.ts:615-634`）：

```ts
const currentModel = Effect.fnUntraced(function* (sessionID: SessionID) {
  const current = yield* db
    .select({ model: SessionTable.model })
    .from(SessionTable)
    .where(eq(SessionTable.id, sessionID))
    .get()
    .pipe(Effect.orDie)
  if (current?.model) return { ...current.model }
  const match = yield* sessions
    .findMessage(sessionID, (m) => m.info.role === "user" && !!m.info.model)
    .pipe(Effect.orDie)
  if (Option.isSome(match) && match.value.info.role === "user") return match.value.info.model
  return yield* provider.defaultModel().pipe(Effect.orDie)
})
```

---

## 十一、运行时权限评估（补充）

即使工具被 `disabled` 过滤，对于仍在列表中的工具（如 `bash`），运行时仍走 `Permission.ask`：

### 11.1 `evaluate`

`packages/opencode/src/permission/index.ts:39-49`：

```ts
export function evaluate(permission: string, pattern: string, ...rulesets: PermissionV1.Ruleset[]): PermissionV1.Rule {
  return (
    rulesets
      .flat()
      .findLast((rule) => Wildcard.match(permission, rule.permission) && Wildcard.match(pattern, rule.pattern)) ?? {
      action: "ask",
      permission,
      pattern: "*",
    }
  )
}
```

按 `(permission, pattern)` 通配符匹配，**取最末匹配规则**；无匹配则兜底为 `ask`（最安全）。

### 11.2 `Permission.ask` 行为

- `deny` → 抛 `DeniedError`
- `ask` → 发 `permission.asked` 事件，阻塞等用户回复
- 用户回复 `reject` → fail deferred
- `once` → succeed 仅本次
- `always` / `session` → 写入 `approved` 列表，永久放行同 session 同 pattern

详见 `permission/index.ts:120-178`。

---

## 十二、实现上的疑点

### 12.1 `plan-mode.txt` 与 `Permission.disabled` 冲突

`prompt/plan-mode.txt:6, 57, 83-84` 指示 LLM 用 `write`/`edit` 创建/修改 plan 文件，但 `Permission.disabled` 同时把 `write` 和 `edit` 从工具列表中**整体剔除**。plan 文件的白名单（`agent.ts:166-173`）只影响运行时 `Permission.ask` 评估，不影响 `disabled` 的静态过滤。

**实际可写入 plan 文件的方式**：只剩 LLM 自行用 `bash` 调 `cat > file.md <<EOF ...` 或类似 shell 重定向。

### 12.2 `plan_enter` 工具未实现

- 描述文件 `packages/opencode/src/tool/plan-enter.txt` 存在
- build agent 允许 `plan_enter: "allow"`（`agent.ts:147`）
- 但 `packages/opencode/src/tool/plan.ts` **只导出 `PlanExitTool`**，没有 `PlanEnterTool`
- `registry.ts:235` 也只把 `plan`（即 `PlanExitTool`）加入工具集
- 前端 `routes/session/index.tsx:330-332` 仍监听 `plan_enter` 完成事件 → **永不被触发的死分支**

### 12.3 `plan_exit` 工具的客户端限制

`registry.ts:235` 用 `flags.client === "cli"` 限制。desktop/app 客户端下模型拿不到 `plan_exit`，必须由用户手动在 picker 里切回 build。

---

## 十三、关键文件速查

### 13.1 Agent 配置
- `packages/opencode/src/agent/agent.ts:35-56` — `Agent.Info` schema
- `packages/opencode/src/agent/agent.ts:117-134` — 默认权限规则集
- `packages/opencode/src/agent/agent.ts:139-153` — `build` agent
- `packages/opencode/src/agent/agent.ts:154-179` — **`plan` agent**
- `packages/opencode/src/agent/agent.ts:180-216` — `general` / `explore` 子 agent

### 13.2 工具与注册
- `packages/opencode/src/tool/plan.ts:15-79` — `PlanExitTool` 实现
- `packages/opencode/src/tool/plan-exit.txt:1-13` — `plan_exit` 描述
- `packages/opencode/src/tool/plan-enter.txt:1-14` — `plan_enter` 描述（无实现）
- `packages/opencode/src/tool/registry.ts:98` — `PlanExitTool` 实例化
- `packages/opencode/src/tool/registry.ts:235` — `plan` 工具条件注册

### 13.3 权限评估
- `packages/opencode/src/permission/index.ts:39-49` — `evaluate()`
- `packages/opencode/src/permission/index.ts:78-118` — `ask()` 运行时
- `packages/opencode/src/permission/index.ts:215-224` — **`disabled()` 静态过滤**

### 13.4 Prompt 注入
- `packages/opencode/src/session/reminders.ts:15-90` — `SessionReminders.apply`
- `packages/opencode/src/session/prompt.ts:1233-1237` — 调用点
- `packages/opencode/src/session/prompt/plan.txt:1-26` — 默认 plan reminder
- `packages/opencode/src/session/prompt/plan-mode.txt:1-70` — 实验 plan reminder
- `packages/opencode/src/session/prompt/build-switch.txt:1-5` — build 切换 reminder

### 13.5 循环与请求
- `packages/opencode/src/session/prompt.ts:636-686` — `createUserMessage`
- `packages/opencode/src/session/prompt.ts:1105-1124` — `SessionPrompt.prompt`
- `packages/opencode/src/session/prompt.ts:1134-1254` — `runLoop`
- `packages/opencode/src/session/llm/request.ts:58-66` — system 拼装
- `packages/opencode/src/session/llm/request.ts:198-204` — **`resolveTools` 过滤**

### 13.6 Plan 文件路径
- `packages/opencode/src/session/session.ts:377-382` — `Session.plan()`

### 13.7 TUI
- `packages/tui/src/context/local.tsx:73-129` — `createAgent()`
- `packages/tui/src/component/dialog-agent.tsx:6-31` — `DialogAgent`
- `packages/tui/src/config/keybind.ts:126-128` — `agent_list`/`agent_cycle`
- `packages/tui/src/app.tsx:467, 664-723` — 启动参数 + 命令
- `packages/tui/src/component/prompt/index.tsx:306-328` — 切会话恢复 agent
- `packages/tui/src/component/prompt/index.tsx:994-997, 1077-1093` — 提交时发送 agent
- `packages/tui/src/routes/session/index.tsx:319-334` — 监听 `plan_exit`/`plan_enter`
- `packages/tui/src/context/data.tsx:132-141` — 渲染 `agent-switched` 消息

### 13.8 其他
- `packages/opencode/src/cli/cmd/tui.ts:206` — `--agent` CLI 参数
- `packages/opencode/src/effect/runtime-flags.ts:47` — `experimentalPlanMode` flag
