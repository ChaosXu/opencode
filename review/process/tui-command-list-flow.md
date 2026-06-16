# OpenCode TUI 列出可用 Command 完整流程

> 分析时间: 2026-06-15
> 范围: 用户在 TUI 输入框敲下 `/` 后，弹窗如何显示可用 command 列表
>
> 路径以 `packages/tui/src/component/prompt/` 为根。两侧关键文件：`autocomplete.tsx`（弹窗组件）和 `index.tsx`（父 Prompt 组件）。
>
> 前置知识：`review/process/tui-startup.md`（provider 树）、`review/process/tui-command-flow.md`（服务端 command 列表拉取）。

---

## 一、整体时序

```
TUI 已挂载（参见 tui-startup.md）
SyncProvider 已拉取 sync.data.command = Command[]（参见 tui-command-flow.md 阶段 B）
Provider 树里 <Autocomplete> 已挂载，store.visible = false
  │
  ▼
[1] 用户在 textarea 敲下 "/"
       │
       ▼
[2] onContentChange 回调  @ prompt/index.tsx:1371-1377
       │  setStore("prompt", "input", "/")
       │  auto()?.onInput("/")        ← 通知 Autocomplete
       │  syncExtmarksWithPromptParts()
       │  setCursorVersion(...)
       │
       ▼
[3] Autocomplete.onInput("/")  @ autocomplete.tsx:665-697
       │  当前 not visible，检测 value.startsWith("/") 且 trigger 前无空格
       │  → show("/")
       │  → setStore({ visible: "/", index: 0 })   ← 关键状态变更
       │
       ▼
[4] Solid 响应式触发重渲
       │
       ├─ 弹窗 <box visible={store.visible !== false}> 显示（位置在 textarea 上方）
       │
       └─ commands memo 重新计算（依赖 sync.data.command）
  │
  ▼
[5] commands memo 构造候选列表  @ autocomplete.tsx:437-464
       │  const results = [...slashes()]   // 本地 keymap 里的 slash
       │  for (const serverCommand of sync.data.command) {
       │    if (serverCommand.source === "skill") continue   // 过滤
       │    const label = serverCommand.source === "mcp" ? ":mcp" : ""
       │    results.push({
       │      display: "/" + serverCommand.name + label,
       │      description: serverCommand.description,
       │      onSelect: () => { /* 删除当前输入 + 插入 "/<name> " */ },
       │    })
       │  }
       │  results.sort(按 display 字典序)
       │  padEnd 对齐到最长项
  │
  ▼
[6] options memo 拼最终可见项  @ autocomplete.tsx:466-514
       │  store.visible === "/" → nonFileOptions = [...commandsValue]
       │  + search() 实时过滤
       │  + fuzzysort 模糊匹配
       │  + frecency 加权
       │  → .slice(0, 10)
  │
  ▼
[7] <Index each={options()}> 渲染  @ autocomplete.tsx:729-766
       │  每个 option 一行:
       │  - 背景色 (选中 = theme.primary)
       │  - 文字 (display + 可选 description)
       │  - 鼠标 hover 切到 mouse 模式 + 高亮
  │
  ▼
[8] 用户行为
       │  8a. 继续打字 → fuzzysort 实时过滤
       │  8b. ↑↓ / Tab 选 → 移动 store.selected
       │  8c. Enter → select() → 调 option.onSelect() → 插入 "/<name> " 到 input
       │  8d. Esc / 字符不再以 / 开头 / 后面跟空格 / 删除 / 鼠标点别处 → hide()
```

---

## 二、阶段详解

### 2.1 Textarea onContentChange 触发

**文件**: `packages/tui/src/component/prompt/index.tsx:1363-1377`

```ts
<textarea
  width="100%"
  placeholder={placeholderText()}
  ...
  onContentChange={() => {
    const value = input.plainText
    setStore("prompt", "input", value)         // 同步到 prompt store
    auto()?.onInput(value)                     // ← 关键：通知 Autocomplete
    syncExtmarksWithPromptParts()
    setCursorVersion((value) => value + 1)
  }}
  onCursorChange={() => setCursorVersion((value) => value + 1)}
  onKeyDown={...}
>
```

`auto` 是个 signal，存的是 `AutocompleteRef`（`prompt/index.tsx:205` + `:1673-1694` 处通过 `setAuto(() => r)` 注册）。`onContentChange` 每次 textarea 文本变化都被 opentui 调起。

### 2.2 Autocomplete.onInput — 弹窗开关逻辑

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:665-697`

```ts
props.ref({
  get visible() { return store.visible },
  onInput(value) {
    if (store.visible) {
      // 当前已显示 → 检查是否该关
      if (
        // 光标在 trigger 前
        props.input().cursorOffset <= store.index ||
        // trigger 后出现空格
        props.input().getTextRange(store.index, props.input().cursorOffset).match(/\s/) ||
        // "/<cmd>" 不是唯一内容（/ 后面有空格又有别的东西）
        (store.visible === "/" && value.match(/^\S+\s+\S+\s*$/))
      ) {
        hide()
      }
      return
    }

    // 当前没显示 → 检查是否该开
    const offset = props.input().cursorOffset
    if (offset === 0) return

    // "/" 在位置 0 且 trigger 前无空格 → 重新打开 slash
    if (value.startsWith("/") && !value.slice(0, offset).match(/\s/)) {
      show("/")
      setStore("index", 0)
      return
    }

    // "@" 触发（文件/agent/MCP resource 引用）
    const idx = mentionTriggerIndex(value, offset)
    if (idx !== undefined) {
      show("@")
      setStore("index", idx)
    }
  },
})
```

`show(mode)`（`autocomplete.tsx:632-637`）：

```ts
function show(mode: "@" | "/") {
  setStore({
    visible: mode,
    index: props.input().cursorOffset,
  })
}
```

**两条触发路径**：
- 首字符 `/` → `show("/")` + `index: 0`（trigger 在位置 0）
- `@` 出现在中间 → `show("@")` + `index: @位置`

### 2.3 commands memo — 合并两个数据源

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:437-464`

```ts
const commands = createMemo((): AutocompleteOption[] => {
  const results: AutocompleteOption[] = [...slashes()]    // ← (a) 本地 slash

  for (const serverCommand of sync.data.command) {     // ← (b) 服务端 command
    if (serverCommand.source === "skill") continue       // 过滤 skill source
    const label = serverCommand.source === "mcp" ? ":mcp" : ""
    results.push({
      display: "/" + serverCommand.name + label,
      description: serverCommand.description,
      onSelect: () => {
        const newText = "/" + serverCommand.name + " "
        const cursor = props.input().logicalCursor
        props.input().deleteRange(0, 0, cursor.row, cursor.col)
        props.input().insertText(newText)
        props.input().cursorOffset = Bun.stringWidth(newText)
      },
    })
  }

  results.sort((a, b) => a.display.localeCompare(b.display))

  const max = firstBy(results, [(x) => x.display.length, "desc"])?.display.length
  if (!max) return results
  return results.map((item) => ({ ...item, display: item.display.padEnd(max + 2) }))
})
```

### 2.4 数据源 (a)：本地 slash

**文件**: `packages/tui/src/keymap.tsx:260-290`

```ts
export function useCommandSlashes(): Accessor<readonly CommandSlashEntry[]> {
  const keymap = useOpencodeKeymap()
  const entries = useKeymapSelector((keymap: OpenTuiKeymap) =>
    keymap.getCommandEntries({
      visibility: "reachable",
      namespace: "palette",
      filter: isVisiblePaletteCommand,
    }),
  )

  return createMemo<CommandSlashEntry[]>(() =>
    entries().flatMap((entry) => {
      const slashName = entry.command.slashName
      if (typeof slashName !== "string" || !slashName) return []
      const slashAliases = entry.command.slashAliases
      return {
        display: `/${slashName}`,
        description: ...,
        aliases: ...,
        onSelect: () => keymap.dispatchCommand(entry.command.name),
      }
    }),
  )
}
```

**这是 TUI 内部 UI 命令**（`@opentui/keymap` 体系），跟 `Command.Service` 完全独立：

- 通过 `registerOpencodeKeymap(...)` 时给每个 keymap command 设的 `slashName`（比如 `prompt.skills`、编辑器操作等）
- namespace 限定为 `"palette"`（命令面板里出现的）
- 选中后直接调 `keymap.dispatchCommand(name)` 跑本地 action（**不**走 HTTP）

例子（参见 `prompt/index.tsx:510-529` 的 "Skills"）：

```ts
{
  title: "Skills",
  name: "prompt.skills",       // keymap command name
  category: "Prompt",
  slashName: "skills",        // slash 别名（"prompt.skills" 去掉 "prompt." 前缀）
  run: () => {
    dialog.replace(() => <DialogSkill ... />)
  },
},
```

### 2.5 数据源 (b)：服务端 command

来自 `sync.data.command`（`packages/tui/src/context/sync.tsx:491` 拉取）：

```ts
sdk.client.command.list({ workspace }).then((x) => setStore("command", reconcile(x.data ?? [])))
```

类型是 `Command.Info[]`（来自 `@opencode-ai/sdk/v2`），每条形如：

```ts
{
  name: "init",
  description: "guided AGENTS.md setup",
  source: "command",  // "command" | "mcp" | "skill"
  hints: [],
  ...
}
```

TUI 过滤规则：
- `source === "skill"` → 跳过（避免和 skill tool 重复，skill 走 `prompt.skills` 命令面板）
- `source === "mcp"` → display 加 `:mcp` 后缀（视觉区分）
- `source === "command"` → 原样

### 2.6 options memo — 过滤 + 排序

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:466-514`

```ts
const options = createMemo((prev: AutocompleteOption[] | undefined) => {
  const filesValue = files()
  const referenceMatchValue = referenceMatch()
  const agentsValue = agents()
  const referenceAliasesValue = referenceAliases()
  const commandsValue = commands()
  const searchValue = search()

  if (store.visible === "@" && referenceMatchValue) {
    return referenceAliasesValue.filter((item) => item.display === `@${referenceMatchValue.name}`)
  }

  // 文件选项只在 @ trigger 下出现
  const fileOptions: AutocompleteOption[] = store.visible === "@" ? filesValue || [] : []
  // slash trigger 只显示命令；@ trigger 显示 reference + agent + MCP resource
  const nonFileOptions: AutocompleteOption[] =
    store.visible === "@" ? [...referenceAliasesValue, ...agentsValue, ...mcpResources()] : [...commandsValue]

  if (!searchValue) return [...nonFileOptions, ...fileOptions]

  if (files.loading && prev && prev.length > 0) return prev

  // fuzzysort 模糊匹配 + frecency
  const fuzziedNonFiles = fuzzysort
    .go(removeLineRange(searchValue), nonFileOptions, {
      keys: [
        (obj) => removeLineRange((obj.value ?? obj.display).trimEnd()),
        ...(store.visible === "/" ? ["description" as const] : []),  // slash 时还匹配 description
        (obj) => obj.aliases?.join(" ") ?? "",
      ],
      limit: 10,
      scoreFn: (objResults) => {
        const displayResult = objResults[0]
        let score = objResults.score
        if (displayResult && displayResult.target.startsWith(store.visible + searchValue)) {
          score *= 2    // 前缀匹配加权
        }
        const frecencyScore = objResults.obj.path ? frecency.getFrecency(objResults.obj.path) : 0
        return score * (1 + frecencyScore)
      },
    })
    .map((arr) => arr.obj)

  return [...fuzziedNonFiles, ...fileOptions].slice(0, 10)
})
```

**关键**：
- `/` trigger → `nonFileOptions = [...commandsValue]` — 纯命令列表
- `@` trigger → `nonFileOptions = [...referenceAliasesValue, ...agentsValue, ...mcpResources()]` — 跟命令无关
- `search` 来自 `filter()` memo，监听 `props.value` + 当前光标/文本状态
- 限 10 个结果
- 选中项 = 字典序第一个（无搜索词时）；有搜索词时 = fuzzysort top 1

### 2.7 弹窗渲染 + 位置

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:711-769`

```ts
return (
  <box
    visible={store.visible !== false}     // 状态门
    position="absolute"
    top={position().y - height()}          // 弹在 textarea 上方
    left={position().x}
    width={position().width}
    zIndex={100}
    {...SplitBorder}
    borderColor={theme.border}
  >
    <scrollbox ref={...} backgroundColor={theme.backgroundMenu} height={height()} ...>
      <Index each={options()} fallback={<text fg={theme.textMuted}>No matching items</text>}>
        {(option, index) => (
          <box
            paddingLeft={1} paddingRight={1}
            backgroundColor={index === store.selected ? theme.primary : undefined}
            flexDirection="row"
            onMouseMove={...} onMouseOver={...} onMouseDown={...} onMouseUp={() => select()}
          >
            <text fg={...}>{option().display}</text>
            <Show when={option().description}>
              <text fg={...} wrapMode="none">{option().description}</text>
            </Show>
          </box>
        )}
      </Index>
    </scrollbox>
  </box>
)
```

- `<box visible={...}>` 让弹窗在不显示时**完全不在渲染树里画帧**
- `position().y - height()` — 弹窗底部对齐到 textarea 顶部（从下往上展开）
- 高度自适应：`Math.min(10, options().length, terminal高度)`
- 选中项高亮用 `theme.primary` 背景
- 鼠标 / 键盘均可交互

### 2.8 选中 / 关闭

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:542-580, 632-650`

`select()` (`:542`) 调当前 `options()[store.selected].onSelect` — 对 `commands` 来说就是清空 input + 插入 `"/<name> "`，用户继续打 arguments。

`hide()` (`:639-650`)：

```ts
function hide() {
  const text = props.input().plainText
  if (store.visible === "/" && !text.endsWith(" ") && text.startsWith("/")) {
    // 用户撤销了 /cmd 没选 → 清掉残留的 /cmd 文本
    const cursor = props.input().logicalCursor
    props.input().deleteRange(0, 0, cursor.row, cursor.col)
    props.setPrompt((draft) => { draft.input = props.input().plainText })
  }
  setStore("visible", false)
}
```

**关键设计**：如果用户敲了 `/fo` 但没选命令，弹窗关闭时**主动删除输入框里的 `/fo`**（避免半残命令在文本里晃）。

### 2.9 keymap 拦截

**文件**: `packages/tui/src/component/prompt/autocomplete.tsx:618-630`

`createSimpleContext` 注册的 keybindings（`autocomplete.tsx:618-630`）：

- `prompt.autocomplete.next` — ↓ 下一项
- `prompt.autocomplete.previous` — ↑ 上一项
- `prompt.autocomplete.next.page` — PgDn
- `prompt.autocomplete.previous.page` — PgUp
- `prompt.autocomplete.next.half` / `previous.half` — Ctrl-U / Ctrl-D
- `prompt.autocomplete.first` — Home
- `prompt.autocomplete.last` — End
- `prompt.autocomplete.select` — Enter（选中）
- `prompt.autocomplete.complete` — Tab（补全当前选项）
- `prompt.autocomplete.hide` — Esc（关弹窗）

这些跟 keymap 注册绑定到 opentui 的键盘事件。

---

## 三、两个数据源的对比

| 维度 | 本地 slash | 服务端 command |
| ---- | --------- | --------------- |
| 来源 | `useCommandSlashes()` 从 keymap 注册表 | `sync.data.command` 从 `GET /command` |
| 注册位置 | `packages/tui/src/keymap.tsx` + `prompt/index.tsx:355-650` 一系列 `registerOpencodeKeymap` 调用 | `packages/opencode/src/command/index.ts:73-158` 4 类来源 |
| 触发后行为 | 本地 `keymap.dispatchCommand(name)` 跑 action（如弹 dialog、切换 mode）| HTTP `POST /session/:id/command` 走模板渲染 + LLM 流 |
| 延迟 | 即时（无网络）| 受 sync 列表拉取时延影响 |
| 例子 | `prompt.skills`（弹 DialogSkill）、`prompt.paste`、`prompt.editor_context.clear`、`prompt.compact` 等 | `init`（AGENTS.md 引导）、`review`、用户 config 里的 `command: { foo: ... }`、MCP prompt 暴露的命令 |
| 标识 | 在 keymap 命令名带 `slashName` | `source: "command" \| "mcp" \| "skill"` |
| 过滤 | 缺 `slashName` 的不进；其他全部进 | `source === "skill"` 不进 |

`commands` memo 把两源合并为一个去重列表（按 `display` 排序）。

---

## 四、几个值得注意的点

1. **三道数据流汇到一处**：
   - 服务端 `Command.Service`（HTTP 拉）
   - 本地 keymap slashName（注册表读）
   - 用户的文本输入 → 触发 fuzzysort 过滤
   - 三者汇入 `commands` memo，再被 `options` memo 拼成最终显示
2. **响应式触发链**：textarea 文本变化 → `onContentChange` → `setStore("prompt", "input", value)` + `auto()?.onInput(value)` → Autocomplete `setStore("visible", "/")` → Solid 重渲弹窗 + `commands` memo 重算（依赖 `sync.data.command` 早就 setStore 过）
3. **Solid batch**：弹窗显示 + 内容更新走同一个 tick，不会闪
4. **padEnd 对齐**（`autocomplete.tsx:458-463`）：`display` 字段补空格让多列对齐 — 视觉上" `/init    guided AGENTS.md setup`"和" `/review   review changes ...`" 描述列齐
5. **/mcp 后缀的语义**：从 `source === "mcp"` 推导的 display 加 `:mcp`，让用户一眼能区分"内置命令"和"MCP 服务器暴露的命令"（虽然底层都是 `Command.Info`）
6. **数据来源更新不是热刷新**：如果用户在 TUI 运行时改了 `opencode.json` 加了新 command 或者启用了新的 MCP server，TUI 的 `sync.data.command` **不会自动重拉**。需要 reload 或重新连接（这是当前实现的限制，参见 `tui-command-flow.md` 十二）
7. **本地 slash 也用 `dispatchCommand` 直接执行**：不走 HTTP，不走 `Command.Service`。是真正的本地 UI 动作
8. **隐藏时清理半残输入**：`hide()` 检查 `text.startsWith("/")` 且 `!text.endsWith(" ")` 时主动 `deleteRange` 删掉 — 防止用户撤销选择后留个孤儿 `/foo`

---

## 五、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| textarea → autocomplete 触发 | `packages/tui/src/component/prompt/index.tsx:1371-1377` |
| Autocomplete 挂载 + setAuto 注册 | `packages/tui/src/component/prompt/index.tsx:1673-1694` |
| AutocompleteRef 接口 | `packages/tui/src/component/prompt/autocomplete.tsx:57-60, 661-698` |
| 内部 `onInput` 状态机 | `packages/tui/src/component/prompt/autocomplete.tsx:665-697` |
| `show` / `hide` | `packages/tui/src/component/prompt/autocomplete.tsx:632-650` |
| `commands` memo（合并两源） | `packages/tui/src/component/prompt/autocomplete.tsx:437-464` |
| `options` memo（最终候选） | `packages/tui/src/component/prompt/autocomplete.tsx:466-514` |
| 弹窗渲染 + 位置 | `packages/tui/src/component/prompt/autocomplete.tsx:711-769` |
| `useCommandSlashes`（本地） | `packages/tui/src/keymap.tsx:260-290` |
| `useKeymapSelector` 实时选择 | `packages/tui/src/keymap.tsx:262-268` |
| sync.data.command 来源 | `packages/tui/src/context/sync.tsx:430-507` (`:491` 拉取) |
| 服务端 `Command.Service` | `packages/opencode/src/command/index.ts:66-181` |
| `Command.Service` 调用方全景 | 参见 `review/process/tui-command-flow.md` |
