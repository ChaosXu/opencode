# OpenCode TUI `/skills` 命令实现分析

> 分析时间: 2026-06-15
> 范围: `/skills` 这个 slash 命令从哪里定义、怎么注册、点击后做了什么
>
> 路径以 `packages/tui/src/component/prompt/` 为根。

---

## 一、关键文件与行号

| 关注点 | 文件 / 行号 |
| ------ | ----------- |
| **命令定义**（"Skills" 项） | `packages/tui/src/component/prompt/index.tsx:510-529` |
| 命令数组 `promptCommands` | `packages/tui/src/component/prompt/index.tsx:330-555` |
| 注册到 keymap（`namespace: "palette"`） | `packages/tui/src/component/prompt/index.tsx:551-559` |
| 触发动作（`dialog.replace(<DialogSkill />)`） | `packages/tui/src/component/prompt/index.tsx:515-528` |
| Dialog UI 实现 | `packages/tui/src/component/dialog-skill.tsx:1-36`（36 行）|
| Dialog 拉 skill 列表 | `packages/tui/src/component/dialog-skill.tsx:15-18`（`sdk.client.app.skills()`）|
| Dialog 选中后注入 `/<skill> ` | `packages/tui/src/component/prompt/index.tsx:518-525` |
| keybind id | `packages/tui/src/config/keybind.ts:353`（`prompt_skills: "prompt.skills"`） |
| slash 别名 `skills` | `packages/tui/src/component/prompt/index.tsx:514`（`slashName: "skills"`）|

---

## 二、命令定义本体

**文件**: `packages/tui/src/component/prompt/index.tsx:510-529`

```ts
{
  title: "Skills",
  name: "prompt.skills",          // keymap command id
  category: "Prompt",
  slashName: "skills",            // 触发 slash autocomplete 显示 "/skills"
  run: () => {
    dialog.replace(() => (
      <DialogSkill
        onSelect={(skill) => {
          input.setText(`/${skill} `)
          setStore("prompt", {
            input: `/${skill} `,
            parts: [],
          })
          input.gotoBufferEnd()
        }}
      />
    ))
  },
},
```

**注意**：`run` 不直接提交 — 它把 `<DialogSkill />` 塞进 dialog 栈；用户选 skill 后，DialogSkill 的 `onSelect(skill.name)` 回调才会把 `/${skill} ` 文本注入 prompt 框（用户回车才会真提交）。

---

## 三、注册到 keymap

**文件**: `packages/tui/src/component/prompt/index.tsx:551-559`

```ts
].map((entry) => ({
  namespace: "palette",
  ...entry,
})),
)

useBindings(() => ({
  commands: promptCommands(),
}))
```

`promptCommands()` 是 `createMemo(() => [...])` 包装的命令列表（`prompt/index.tsx:330-555`），里面包含 `prompt.clear` / `prompt.submit` / `prompt.editor` / **`prompt.skills`** / `session.move` / `workspace.set` 等十几条 TUI 本地命令。

`useBindings` 来自 `@opentui/keymap/solid`（`packages/tui/src/keymap.tsx:15, 29`），把命令数组注入到当前 keymap provider。`namespace: "palette"` 表示这些命令出现在命令面板（"Command Palette"）里。

---

## 四、DialogSkill 本身

**文件**: `packages/tui/src/component/dialog-skill.tsx`（36 行全文）

```ts
import { DialogSelect, type DialogSelectOption } from "../ui/dialog-select"
import { createResource, createMemo } from "solid-js"
import { useDialog } from "../ui/dialog"
import { useSDK } from "../context/sdk"

export type DialogSkillProps = {
  onSelect: (skill: string) => void
}

export function DialogSkill(props: DialogSkillProps) {
  const dialog = useDialog()
  const sdk = useSDK()
  dialog.setSize("large")

  const [skills] = createResource(async () => {
    const result = await sdk.client.app.skills()       // ← GET /skill
    return result.data ?? []
  })

  const options = createMemo<DialogSelectOption<string>[]>(() => {
    const list = skills() ?? []
    const maxWidth = Math.max(0, ...list.map((s) => s.name.length))
    return list.map((skill) => ({
      title: skill.name.padEnd(maxWidth),
      description: skill.description?.replace(/\s+/g, " ").trim(),
      value: skill.name,
      category: "Skills",
      onSelect: () => {
        props.onSelect(skill.name)                    // ← 回调到 prompt/index.tsx
        dialog.clear()
      },
    }))
  })

  return <DialogSelect title="Skills" placeholder="Search skills..." options={options()} />
}
```

**关键点**：
- `sdk.client.app.skills()` 走 **HTTP `GET /skill`**（`groups/instance.ts:159-168`）→ 服务端 `InstanceHttpApi.skill` handler（`handlers/instance.ts:84-86`）→ `skill.all()`（**注意**这里是 `all()` 不是 `available(agent)`，不过滤 agent 权限 — 因为这是给人看的）
- 返回 `Skill.Info[]`：`{ name, description?, location, content }`
- 复用通用 `DialogSelect` 组件做搜索 + 选中
- 选中后 `props.onSelect(skill.name)` 回调 + `dialog.clear()`

---

## 五、触发 `/skills` 的三条路径

| 路径 | 怎么到达 `prompt.skills` | 走哪条 API |
| ---- | ----------------------- | ---------- |
| **命令面板** | Ctrl+Shift+P（默认）打开 palette → 输入"Skills" → Enter | `keymap.dispatchCommand("prompt.skills")` 直接调 `run` |
| **Slash autocomplete** | 输入 `/` → 看到 `/skills`（来自 `useCommandSlashes()`，因为 `slashName: "skills"`） → Enter / Tab 选中 | `keymap.dispatchCommand(entry.command.name)` 调 `run` |
| **键盘快捷键** | 配置文件 `keybind.ts:353` 把 `prompt_skills: "prompt.skills"` 注册到快捷键层 | 触发时走 `keymap.dispatchCommand` |

> 三条路径**完全等价** — 都通过 keymap 体系最终调到 `run` 回调。

---

## 六、用户选中 skill 后真正"跑"了什么

用户在 DialogSkill 里点 `commit` → `onSelect("commit")` → 回到 `prompt/index.tsx:518-525`：

```ts
input.setText(`/${skill} `)            // prompt 框显示 "/commit "
setStore("prompt", {
  input: `/${skill} `,
  parts: [],
})
input.gotoBufferEnd()                 // 光标移到末尾
```

然后**用户自己按回车** → `submit()` → `submitInner()`（`prompt/index.tsx:1065-1085`）：

```ts
} else if (
  inputText.startsWith("/") &&
  sync.data.command.some((x) => x.name === inputText.split("\n")[0].split(" ")[0].slice(1))
) {
  ...
  void sdk.client.session.command({
    sessionID, command: command.slice(1), arguments: args, ...
  })
}
```

- `commit` 是从 `sync.data.command` 里查的 — 服务端 `Command.Service` 把每个 skill 也注册成了 `source: "skill"` 的 command（`packages/opencode/src/command/index.ts:142-153`）
- 命中 → 走 `sdk.client.session.command({ command: "commit", ... })`
- → 服务端 `SessionPrompt.command`（`packages/opencode/src/session/prompt.ts:1417-1542`）
- → `commands.get("commit")` 拿到 `template = item.content`（SKILL.md 正文）
- → 模板渲染 → `prompt()` → 走普通 LLM 流 → SKILL.md 内容被塞回 user message

> **关键洞察**：`/skills` **本身**不执行任何东西 — 它只起一个 picker。`/skills` 选完之后的 `/${skill} ` 文本才是**真正**的"跑 skill"入口 — 走的是**用户输入斜杠命令**的标准流程（参见 `review/process/tui-command-flow.md`）。

---

## 七、`/skills` 在 Autocomplete 里显示来源

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:437-464`

`commands` memo 的合并逻辑：

```ts
const commands = createMemo((): AutocompleteOption[] => {
  const results: AutocompleteOption[] = [...slashes()]   // ← (a) 本地 keymap slash

  for (const serverCommand of sync.data.command) {    // ← (b) 服务端 command
    if (serverCommand.source === "skill") continue     // 服务端 skill command 不进 autocomplete
    ...
  }
  ...
})
```

| 数据源 | 是否进 `/` 弹窗 | 说明 |
| ------ | --------------- | ---- |
| 本地 keymap `slashName` = "skills"（`prompt.skills`） | ✅ | 显示为 `/skills` |
| 服务端 `Command.Service` 的 skill 列表（`source: "skill"`） | ❌ 主动过滤 | 防止弹窗里 skill 名和服务端 command 名重复（参见 `tui-command-list-flow.md` 第三节） |
| 服务端 `Command.Service` 的 `init` / `review` / config / MCP | ✅ | `source !== "skill"` 都进 |

所以 `/skills` 是**唯一**用户能选的"以 skill 为目的的本地命令"。其他 skill 名字（`commit` / `review-code` / `customize-opencode` 等）只能通过 DialogSkill 选中后由 prompt 自动填入，不会直接在 `/` 弹窗里出现。

---

## 八、值得注意的设计

1. **`/skills` 不会出现在 `sync.data.command` 里** — 它是 TUI keymap 层的本地命令，跟服务端 `Command.Service` 完全无关
2. **TUI 不把 service 端 skill command 在 autocomplete 显示** — 否则用户能从弹窗直接选 `commit`，绕开 DialogSkill 引导，UX 不一致
3. **DialogSkill 不传 `agent` 参数给 `/skill` 端点** — 所以 TUI 拿到的列表**不过滤 agent 权限**（服务器端 `skill.all()` 不过 agent）；真正的"用户对该 agent 可见哪些 skill"过滤发生在模型侧（`Skill.available(agent)` 给 system prompt 用）
4. **DialogSkill 用通用 `DialogSelect`** — 跟 `DialogProvider` / `DialogAgent` / `DialogModel` 等是同一个组件，自动获得搜索 + 鼠标 + 键盘支持
5. **触发后只注入文本不提交** — 给用户最后一次"我要不要这个 skill"的机会（光标在 `/commit ` 末尾，用户可以继续打 args 如 `/commit focus on backend`）

---

## 九、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| `/skills` 命令定义 | `packages/tui/src/component/prompt/index.tsx:510-529` |
| `promptCommands` 数组起点 | `packages/tui/src/component/prompt/index.tsx:330` |
| 注册到 keymap | `packages/tui/src/component/prompt/index.tsx:551-559` |
| Dialog UI | `packages/tui/src/component/dialog-skill.tsx:1-36` |
| 服务端 `GET /skill` 端点 | `packages/opencode/src/server/routes/instance/httpapi/handlers/instance.ts:84-86` |
| 服务端 skill 注册 | `packages/opencode/src/skill/index.ts:97-329` |
| 选中后真正的 slash command 路径 | `packages/tui/src/component/prompt/index.tsx:1065-1085` |
| slash autocomplete 合并 | `packages/tui/src/component/prompt/autocomplete.tsx:437-464` |
| `Command.Service` 装配 | `packages/opencode/src/command/index.ts:73-158` |
| skill 整体机制 | `review/process/tui-skill.md` |
| command 整体机制 | `review/process/tui-command-flow.md` |
