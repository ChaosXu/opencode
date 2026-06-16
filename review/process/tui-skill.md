# OpenCode TUI Skill 系统完整流转

> 分析时间: 2026-06-15
> 范围: skill 来源 → 模型被告知 → 模型触发 → TUI 显示
>
> 路径以 `packages/` 为根。"LLM"指 OpenCode 服务端而非 TUI；TUI 端组件标注 `tui/src/`。

---

## 一、整体链路

```
[1] 技能来源 ──────────────► 内置 1 个 + 外部扫描 N 个 + 配置路径 N 个 + URL 下载 N 个
       │                   ↳ 落进 Skill.Service.state
       │
[2] 模型被告知有哪些 skill ─► 两条独立通道：
       │                      ① SystemPrompt.skills(agent) → 注入到 system prompt
       │                      ② ToolRegistry 注册 "skill" 工具 → 出现在 tools 列表
       │
[3] 模型触发 skill ────────► LLM 调 tool_use skill({name})  (原生工具调用)
       │                      路由: ToolRegistry → SessionTools.resolve → ai.tool() 包装
       │                      执行: SkillTool.execute:
       │                          1. Skill.require(name) — 查 skill
       │                          2. ctx.ask(permission="skill", patterns=[name]) — 问用户
       │                          3. Ripgrep 找 skill 目录下非 SKILL.md 的文件
       │                          4. 返回 <skill_content>...</skill_content> 给模型
       │                      路由: SessionProcessor.handleEvent("tool-result") → 写 part → V2 事件
       │
[4] TUI 显示 ──────────────► SyncProvider 收到 message.part.updated → Solid store 更新
       │                      Session 路由 <For each={messages()}> 重渲
       │                      用户看到 tool call 卡片 + skill 内容回灌
       │
[5] TUI 主动选 skill ──────► prompt.slashName="skills" 命令（`/skills`）
                              ↳ DialogSkill 拉 sdk.client.app.skills() 列表
                              ↳ 用户选 → setText("/<skill-name> ") + 提交
                              ↳ 走 command 路径（`session.command`）→ 模板里通常会
                                含 "/skill <name>" 之类的语义，落到 command 的执行逻辑
```

---

## 二、Skill 的发现与注册

**文件**: `packages/opencode/src/skill/index.ts:1-329`

### 2.1 四种 skill 来源（按优先级排列）

| 来源 | 路径 | 模式 | 行号 |
| ---- | ---- | ---- | ---- |
| 内置（始终在） | `packages/core/src/plugin/skill.ts` + `packages/core/src/plugin/skill/customize-opencode.md` | `customize-opencode` 一个 | `skill/index.ts:32-35, 278-283` |
| 外部 Claude 兼容 | `~/.claude/skills/**/SKILL.md`、`~/.agents/skills/**/SKILL.md` | `EXTERNAL_SKILL_PATTERN` | `skill/index.ts:21-23, 186-203` |
| OpenCode config 目录 | 每个 config 目录下 `{skill,skills}/**/SKILL.md` | `OPENCODE_SKILL_PATTERN` | `skill/index.ts:24, 205-208` |
| 配置显式 path | `config.skills.paths: string[]` | `SKILL_PATTERN` | `skill/index.ts:211-220` |
| 配置显式 URL | `config.skills.urls: string[]` | `Discovery.pull` 下载到 `~/.cache/opencode/skills/<name>/` | `skill/index.ts:222-227` + `discovery.ts` |

### 2.2 加载流程

```
SkillService.layer 装配
  ├── discovered: InstanceState.make(  // 每个 project dir 一份缓存
  │   └── discoverSkills(config, discovery, fsys, global, flags, dir, worktree)
  │       ├── Glob.scan 在各 root + config dirs + 路径 + URL 缓存里找 SKILL.md
  │       └── 累加 matches 到 ScanState
  │
  └── state: InstanceState.make(
        └── loadSkills(state, discovered, events)
            ├── add(state, match, events) for each match
            │   ├── ConfigMarkdown.parse(match) 解析 frontmatter + body
            │   ├── 校验 {name, description?}
            │   ├── 重复 name → logWarning 但不报错
            │   └── state.skills[name] = {name, description, location, content}
            ├── **先注册内置 customize-opencode**（disk 可覆盖）
            └── yield* logInfo("init", {count})
```

注意 `:276-283` 的注释：

```ts
// Register the built-in skill BEFORE disk discovery so a user-disk
// skill with the same name can override it.
```

实现上其实顺序反了 — 这里是先放内置名进入 `s.skills`，再走 `loadSkills` 扫盘；同名 disk 版本会覆盖内置（`add` 是无脑赋值）。

### 2.3 权限过滤

```ts
// skill/index.ts:310-315
const available = Effect.fn("Skill.available")(function* (agent?: Agent.Info) {
  const s = yield* InstanceState.get(state)
  const list = Object.values(s.skills).toSorted((a, b) => a.name.localeCompare(b.name))
  if (!agent) return list
  return list.filter((skill) => Permission.evaluate("skill", skill.name, agent.permission).action !== "deny")
})
```

`Permission.evaluate("skill", <skillName>, agent.permission)` 看 agent 的 ruleset 里有没有 `permission: "skill"` + `pattern: <name>` + `action: "deny"` 的规则。

TUI 显示也走 `available(agent)`，所以同一 agent 的"用户可见 skill"和"模型可见 skill"是一致的。

### 2.4 Flag 屏蔽

`flags.disableExternalSkills` / `flags.disableClaudeCodeSkills`（`RuntimeFlags.Service`）— 通过 `OPENCODE_DISABLE_EXTERNAL_SKILLS` / `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS` 等 env 关闭外部 skill 来源。

---

## 三、模型如何被告知有哪些 skill

### 3.1 通道 ①：系统提示注入（主要方式）

**文件**: `packages/opencode/src/session/system.ts:94-106`

```ts
skills: Effect.fn("SystemPrompt.skills")(function* (agent: Agent.Info) {
  if (Permission.disabled(["skill"], agent.permission).has("skill")) return   // 完全 disable 则连提示都省

  const list = yield* skill.available(agent)   // 已应用 agent 权限过滤

  return [
    "Skills provide specialized instructions and workflows for specific tasks.",
    "Use the skill tool to load a skill when a task matches its description.",
    // the agents seem to ingest the information about skills a bit better if we present a more verbose
    // version of them here and a less verbose version in tool description, rather than vice versa.
    Skill.fmt(list, { verbose: true }),
  ].join("\n")
}),
```

格式化函数 `Skill.fmt(list, { verbose: true })`（`skill/index.ts:330-355`）生成 XML：

```xml
<available_skills>
  <skill>
    <name>customize-opencode</name>
    <description>Use ONLY when the user is editing or creating opencode's own configuration...</description>
    <location>file:///path/to/SKILL.md</location>
  </skill>
  <skill>
    <name>commit</name>
    <description>Generate a commit message and create a git commit</description>
    <location>file:///Users/.../skills/commit/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

### 3.2 注入到 LLM 请求的 system 字段

**文件**: `packages/opencode/src/session/prompt.ts:1327-1333`（在 `runLoop` 每轮）

```ts
const [skills, env, instructions, modelMsgs] = yield* Effect.all([
  sys.skills(agent),
  sys.environment(model),
  instruction.system().pipe(Effect.orDie),
  MessageV2.toModelMessagesEffect(msgs, model),
])
const system = [...env, ...instructions, ...(skills ? [skills] : [])]
// ...然后 system 作为 llm.stream 的 system 字段
const result = yield* handle.process({
  user, agent, permission, sessionID, parentSessionID,
  system,
  messages: [...modelMsgs, ...(isLastStep ? [{role:"assistant", content: MAX_STEPS}] : [])],
  tools,
  model,
  toolChoice: format.type === "json_schema" ? "required" : undefined,
})
```

> 这意味着每个 turn 都重新渲染一次"当前可用 skill 列表" — 如果用户/agent 切换导致 skill 集合变化，下一轮立刻生效。

### 3.3 通道 ②：`skill` 工具注册

**文件**: `packages/opencode/src/tool/registry.ts:198-232`

```ts
const tool = yield* Effect.all({
  ...
  skill: Tool.init(skilltool),
  ...
})

return {
  custom,
  builtin: [
    tool.invalid,
    ...(questionEnabled ? [tool.question] : []),
    tool.shell,
    tool.read,
    tool.glob,
    tool.grep,
    tool.edit,
    tool.write,
    tool.task,
    tool.fetch,
    tool.todo,
    tool.search,
    tool.skill,           // ← 内置工具之一
    tool.patch,
    ...(flags.experimentalLspTool ? [tool.lsp] : []),
    ...(flags.experimentalPlanMode && flags.client === "cli" ? [tool.plan] : []),
  ],
  task: tool.task,
  read: tool.read,
}
```

工具描述（`packages/opencode/src/tool/skill.txt`）非常简短：

```
Load a specialized skill when the task at hand matches one of the skills listed in the system prompt.

Use this tool to inject the skill's instructions and resources into current conversation. The output may contain detailed workflow guidance as well as references to scripts, files, etc in the same directory as the skill.

The skill name must match one of the skills listed in your system prompt.
```

> 设计上 "system prompt 里 verbose 列名 + tool 描述里 terse 说怎么用"（`system.ts:102-103` 注释明确说明：模型对这种"两个地方各管一边"的形式吸收得更好）。

### 3.4 工具 schema

`packages/opencode/src/tool/skill.ts:9-12`：

```ts
export const Parameters = Schema.Struct({
  name: Schema.String.annotate({ description: "The name of the skill from available_skills" }),
})
```

模型只需要传一个 `name` 字段（从 `<available_skills>` 列表里挑）。

---

## 四、模型如何触发 skill

### 4.1 工具调用生命周期

```
LLM 在某个 turn 输出:
  tool_use: { id, name: "skill", input: { name: "commit" } }

↓
[Provider stream → LLMEvent] (packages/llm)
  protocol.step 解析 → LLMEvent{ type: "tool-call", id, name:"skill", input:{name:"commit"} }

↓
[SessionProcessor.handleEvent]  packages/opencode/src/session/processor.ts:468-547
  case "tool-call":
    ├── ensureToolCall(value) → 创建 tool part (status: pending → running)
    ├── updateToolCall(id, (m) => ({...m, tool:"skill", state:{status:"running", input, time:{start}}}))
    ├── events.publish(SessionEvent.Tool.Called, { sessionID, callID, tool:"skill", input })
    ├── doom-loop 检测（连发同 tool 同 input 触发 permission ask）
    └── return

↓
[SessionTools.resolve]  packages/opencode/src/session/tools.ts:74-115
  registry.tools() 返回的 tool 被包成 ai.tool({ description, inputSchema, execute(args, options) {...} })
  execute 内 run.promise(Effect.gen(...)):
    ├── plugin.trigger("tool.execute.before", { tool:"skill", ... }, { args })
    ├── item.execute(args, ctx)   ← 这里就是 SkillTool.execute
    ├── plugin.trigger("tool.execute.after", ...)
    └── 返回 ToolResult { title, output, metadata, attachments? }

↓
[SkillTool.execute]  packages/opencode/src/tool/skill.ts:22-68
  1. Skill.require(params.name)
       - 查 state.skills[name]
       - 缺失抛 Skill.NotFoundError → 透传成 Error（OrDie）
  2. ctx.ask({ permission:"skill", patterns:[name], always:[name], metadata:{} })
       - 走 Permission.ask → 发 permission.asked 事件 → TUI 弹授权 dialog
       - 用户在 TUI 选 "once" / "always" / "reject"
       - reject → permission.asked 事件带 reject reply → 工具执行中断
  3. Ripgrep.find({ cwd: dir, pattern: "!**/SKILL.md", hidden:true, follow:false, signal, limit:10 })
       - 找 skill 目录下其它附属文件（脚本、参考等）
  4. 返回：
     {
       title: `Loaded skill: ${info.name}`,
       output: `<skill_content name="${name}">\n# Skill: ...\n${info.content}\n\nBase directory: file://...\n<skill_files>\n<file>...</file>\n</skill_files>\n</skill_content>`,
       metadata: { name, dir }
     }
```

### 4.2 结果回灌

工具的 `output` 字段是 `<skill_content>` 块：

```xml
<skill_content name="commit">
# Skill: commit

<SKILL.md 的内容，含 YAML frontmatter 之外的 markdown 正文>

Base directory for this skill: file:///Users/.../skills/commit/
Relative paths in this skill (e.g., scripts/, reference/) are relative to this base directory.
Note: file list is sampled.

<skill_files>
<file>/Users/.../skills/commit/scripts/commit-msg.sh</file>
<file>/Users/.../skills/commit/references/style.md</file>
</skill_files>
</skill_content>
```

这个 output 走完与普通 tool 一样的回灌路径：

```
[ai SDK execute] → run.promise 返回值
↓
[SessionTools.execute wrapper] 把 output 走 tool-result event
↓
[SessionProcessor.handleEvent case "tool-result"]  packages/opencode/src/session/processor.ts:549-647
  ├── image normalize (如果有 image attachment)
  ├── completeToolCall(value.id, output) 写 part state = completed
  ├── events.publish(SessionEvent.Tool.Success, { sessionID, callID, content, result, ...})
  └── return

↓
[下一轮 LLM call]  MessageV2.toModelMessagesEffect 把 tool_use + tool_result 装成 model 消息
   模型现在能看到 SKILL.md 的内容 + 同目录文件列表
   就能按 skill 里的指令去执行（例如 "/commit" skill 教它生成 commit message 然后 git commit）
```

### 4.3 模型"消化"skill 后能做的事

- 直接调 `bash` 跑 `git commit -m "..."`
- 调 `read` / `write` 编辑文件
- 调 `task` 派子 agent
- 任何 skill 文本里教的操作，模型都能用现成的 tools 实现

---

## 五、TUI 端表现

### 5.1 TUI 拉 skill 列表

`packages/tui/src/component/dialog-skill.tsx:10-35`：

```ts
export function DialogSkill(props: DialogSkillProps) {
  const [skills] = createResource(async () => {
    const result = await sdk.client.app.skills()
    return result.data ?? []
  })
  // 渲染成 DialogSelect 列表
}
```

HTTP 走 `GET /skill`（`groups/instance.ts:159-168`）→ `InstanceHttpApi.skill` handler（`handlers/instance.ts:84-86`）→ `skill.all()`（注意：这里是 `all()` 而不是 `available(agent)`，所以 TUI 拉到的列表不过滤 agent 权限 — 真正的过滤在模型侧）。

### 5.2 用户主动选 skill

**文件**: `packages/tui/src/component/prompt/index.tsx:510-529`

```ts
{
  title: "Skills",
  name: "prompt.skills",
  category: "Prompt",
  slashName: "skills",
  run: () => {
    dialog.replace(() => (
      <DialogSkill
        onSelect={(skill) => {
          input.setText(`/${skill} `)
          setStore("prompt", { input: `/${skill} `, parts: [] })
          input.gotoBufferEnd()
        }}
      />
    ))
  },
},
```

注册为 slash command `skills`（命令面板里有 "Skills"），触发表单填写 `/<skill_name> `，用户回车后走 `submit()` → `submitInner()` 走命令分支（`prompt/index.tsx:1065-1085`）→ `sdk.client.session.command({ sessionID, command, arguments, agent, model, ... })`。

> 注意：这是"用户**主动**触发"——跟模型触发是两条不同的路径。模型侧走 `skill` tool call；用户侧走 `/skillname` command call。两条路径在服务端都最终把 SKILL.md 的内容回灌到模型上下文。

### 5.3 权限询问 UI

工具的 `ctx.ask({ permission:"skill", patterns:[name] })` 在 TUI 端走和所有权限一样的弹窗（`packages/tui/src/routes/session/permission.tsx`）：

- 标题"Permission requested: skill (commit)"
- 选项 "once" / "always" / "reject"
- "always" 在 `ctx.ask` 里已经把 `always:[name]` 传过去，所以"总是允许"按 skill 粒度记
- 用户选 reject → 工具抛错 → 走 `tool-error` event → 下一轮模型看到错误自纠

### 5.4 skill 内容显示

工具成功执行后，`<skill_content>...</skill_content>` 出现在 TUI 的 tool 卡片中（折叠状态），用户可展开看完整内容。模型则在下一轮 turn 的 messages 里"看见"这段文字。

---

## 六、关键设计要点

1. **"提示 + 工具"双通道**：模型在 system prompt 看到 `<available_skills>` 列表（带 description），又看到 `skill` 工具描述（带简短说明）— 两者职责分明，前者负责"有哪些、做什么"，后者负责"怎么调用"
2. **description 是模型路由的关键**：description 写得好不好决定模型能否选对 skill；`Skill.fmt` 强制要求有 description（`fmt` 把没 description 的过滤掉）
3. **permission 双向过滤**：`SystemPrompt.skills(agent)` 跑 `Permission.disabled(["skill"], ...)` 早退；`Skill.available(agent)` 跑 `Permission.evaluate("skill", name, ...)` 单个 filter
4. **运行时动态性**：每 turn 重新调 `sys.skills(agent)`，所以"用户新增了一个 skill 文件"在下个 turn 立即可用（前提是 project 已重新发现 — `InstanceState` 缓存）
5. **注入而非拼装**：skill content 是"塞回工具结果"而不是塞进 system prompt — 这样模型知道"这是 tool 结果"语义，能更可靠地遵守 skill 里的指令
6. **built-in 兜底**：始终有 `customize-opencode` skill，确保用户编辑 opencode 自身配置时模型有权威 schema 指导

---

## 七、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| Skill Service + 发现 | `packages/opencode/src/skill/index.ts:97-329` |
| 内置 skill plugin | `packages/core/src/plugin/skill.ts:1-34` |
| 内置 skill 内容 | `packages/core/src/plugin/skill/customize-opencode.md` |
| URL 下载 | `packages/opencode/src/skill/discovery.ts:1-110` |
| System prompt 注入 | `packages/opencode/src/session/system.ts:94-106` |
| 拼装到 LLM 请求 | `packages/opencode/src/session/prompt.ts:1327-1333` |
| `skill` 工具实现 | `packages/opencode/src/tool/skill.ts:13-71` |
| 工具描述 | `packages/opencode/src/tool/skill.txt` |
| 工具注册 | `packages/opencode/src/tool/registry.ts:198-232` |
| 工具到 ai SDK | `packages/opencode/src/session/tools.ts:74-115` |
| 工具执行（handleEvent） | `packages/opencode/src/session/processor.ts:468-647` |
| HTTP `/skill` 列表 | `packages/opencode/src/server/routes/instance/httpapi/handlers/instance.ts:84-86` |
| TUI 选 skill dialog | `packages/tui/src/component/dialog-skill.tsx:10-35` |
| TUI `/skills` 命令 | `packages/tui/src/component/prompt/index.tsx:510-529` |
| Permission 决策 | `packages/opencode/src/permission/index.ts:215-224` |
| TUI 权限弹窗 | `packages/tui/src/routes/session/permission.tsx` |
