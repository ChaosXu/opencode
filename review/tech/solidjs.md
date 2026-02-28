# SolidJS 使用分析

## 1. 核心信号使用

项目**不推荐**单独使用多个 `createSignal`，而是优先使用 `createStore` 管理状态。

### createSignal 示例

```typescript
// packages/app/src/components/prompt-input.tsx
const [store, setStore] = createStore<{
  popover: "at" | "slash" | null
  historyIndex: number
  savedPrompt: PromptHistoryEntry | null
  placeholder: number
  draggingType: "image" | "@mention" | null
  mode: "normal" | "shell"
  applyingHistory: boolean
}>({
  popover: null,
  historyIndex: -1,
  savedPrompt: null,
  placeholder: 0,
  draggingType: null,
  mode: "normal",
  applyingHistory: false,
})
```

## 2. createMemo 典型用法

项目中大量使用 `createMemo` 进行计算缓存：

```typescript
// 简单计算
const sessionKey = createMemo(() => `${params.dir}${params.id ? "/" + params.id : ""}`)
const tabs = createMemo(() => layout.tabs(sessionKey))
const view = createMemo(() => layout.view(sessionKey))

// 复杂计算
const recent = createMemo(() => {
  const all = tabs().all()
  const active = tabs().active()
  const order = active ? [active, ...all.filter((x) => x !== active)] : all
  return paths
})

const status = createMemo(() => sync.data.session_status[params.id ?? ""] ?? { type: "idle" })
```

## 3. createEffect 典型用法

```typescript
// 监听状态变化并重置
createEffect(() => {
  scope()
  inflight.clear()
  resetFileContentLru()
  batch(() => {
    setStore("file", reconcile({}))
    tree.reset()
  })
})

// 副作用：更新 DOM
createEffect(() => {
  if (typeof document === "undefined") return
  document.documentElement.style.setProperty("--font-family-mono", monoFontFamily(store.appearance?.font))
})
```

## 4. createStore 状态管理模式（推荐）

项目广泛使用 `createStore` 替代多个 `createSignal`：

```typescript
// 基础 store
const [store, setStore] = createStore<{
  file: Record<string, FileState>
}>({
  file: {},
})

// 使用 produce 更新嵌套状态
const setLoading = (file: string) => {
  setStore(
    "file",
    file,
    produce((draft) => {
      draft.loading = true
      draft.error = undefined
    }),
  )
}
```

### 全局状态示例

```typescript
const [globalStore, setGlobalStore] = createStore<GlobalStore>({
  ready: false,
  path: { state: "", config: "", worktree: "", directory: "", home: "" },
  project: projectCache.value,
  session_todo: {},
  provider: { all: [], connected: [], default: {} },
  provider_auth: {},
  config: {},
  reload: undefined,
})
```

## 5. Context 和 Store 使用模式

### 方式一：createSimpleContext（推荐）

项目自定义的简化 Context 模式：

```typescript
// packages/app/src/context/settings.tsx
export const { use: useSettings, provider: SettingsProvider } = createSimpleContext({
  name: "Settings",
  init: () => {
    const [store, setStore, _, ready] = persisted("settings.v3", createStore<Settings>(defaultSettings))
    return {
      ready,
      get current() {
        return store
      },
    }
  },
})
```

### 方式二：传统 createContext（packages/ui）

```typescript
const Context = createContext<UiI18n>(fallback)

export function I18nProvider(props: ParentProps<{ value: UiI18n }>) {
  return <Context.Provider value={props.value}>{props.children}</Context.Provider>
}

export function useI18n() {
  return useContext(Context)
}
```

## 6. 组件结构模式

```typescript
// packages/ui/src/components/button.tsx
import { Button as Kobalte } from "@kobalte/core/button"
import { type ComponentProps, Show, splitProps } from "solid-js"

export function Button(props: ButtonProps) {
  const [split, rest] = splitProps(props, ["variant", "size", "icon", "class", "classList"])
  return (
    <Kobalte {...rest} data-component="button" data-size={split.size || "normal"}>
      <Show when={split.icon}>
        <Icon name={split.icon!} size="small" />
      </Show>
      {props.children}
    </Kobalte>
  )
}
```

## 7. 持久化状态

```typescript
// 使用 persisted hook
const [store, setStore, _, ready] = persisted("settings.v3", createStore<Settings>(defaultSettings))

// 持久化到全局存储
const [projectCache, setProjectCache, , projectCacheReady] = persisted(
  Persist.global("globalSync.project", ["globalSync.project.v1"]),
  createStore({ value: [] as Project[] }),
)
```

## 8. 典型 Hooks 使用

```typescript
// 在组件中使用多个 context
export const PromptInput: Component<PromptInputProps> = (props) => {
  const sdk = useSDK()
  const sync = useSync()
  const local = useLocal()
  const files = useFile()
  const prompt = usePrompt()
  const layout = useLayout()
  // ...
}
```

## 9. 最佳实践总结

| 实践                    | 说明                                           |
| ----------------------- | ---------------------------------------------- |
| **优先 createStore**    | 项目明确偏好 createStore 替代多个 createSignal |
| **createMemo**          | 所有派生状态使用 createMemo 缓存               |
| **createEffect**        | 用于副作用：DOM 更新、订阅、外部交互           |
| **createSimpleContext** | 封装了传统 Context 的繁琐模式                  |
| **produce**             | 使用 Immer 风格的 produce 更新深层状态         |
| **batch**               | 多个状态变更合并为一次渲染                     |
| **splitProps**          | 解构组件 props 的推荐方式                      |
| **persisted**           | 使用自定义持久化机制保存状态到 localStorage    |
