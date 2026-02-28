# Kobalte 组件库使用分析

## 1. 组件导入方式

项目使用 `@kobalte/core` 作为核心依赖，通过命名导入获取各个组件：

```typescript
import { Dialog as Kobalte } from "@kobalte/core/dialog"
import { Select as Kobalte } from "@kobalte/core/select"
import { DropdownMenu as Kobalte } from "@kobalte/core/dropdown-menu"
import { Tabs as Kobalte } from "@kobalte/core/tabs"
```

## 2. 组件使用统计

项目中共使用了 **21 个 Kobalte 组件**：

| 组件类型         | 文件               |
| ---------------- | ------------------ |
| Dialog           | dialog.tsx         |
| Select           | select.tsx         |
| DropdownMenu     | dropdown-menu.tsx  |
| Tooltip          | tooltip.tsx        |
| Accordion        | accordion.tsx      |
| Tabs             | tabs.tsx           |
| Checkbox         | checkbox.tsx       |
| Switch           | switch.tsx         |
| RadioGroup       | radio-group.tsx    |
| ContextMenu      | context-menu.tsx   |
| Popover          | popover.tsx        |
| Collapsible      | collapsible.tsx    |
| Toast            | toast.tsx          |
| Progress         | progress.tsx       |
| TextField        | text-field.tsx     |
| HoverCard        | hover-card.tsx     |
| ImagePreview     | image-preview.tsx  |
| IconButton       | icon-button.tsx    |
| Button           | button.tsx         |
| MessageNav       | message-nav.tsx    |
| Dialog (context) | context/dialog.tsx |

## 3. 可访问性实现

Kobalte 提供了完整的 WAI-ARIA 支持，项目中通过以下方式实现可访问性：

### 对话框可访问性

```typescript
<Kobalte.Title data-slot="dialog-title">{props.title}</Kobalte.Title>
<Kobalte.Description data-slot="dialog-description">{props.description}</Kobalte.Description>
<Kobalte.CloseButton aria-label={i18n.t("ui.common.close")} />
```

### 选择器可访问性

```typescript
<Kobalte.Trigger as={Button}>...</Kobalte.Trigger>
<Kobalte.Value>{(state) => ...}</Kobalte.Value>
<Kobalte.Item>...</Kobalte.Item>
<Kobalte.ItemIndicator>...</Kobalte.ItemIndicator>
```

### 表单控件可访问性

```typescript
<Kobalte.Input data-slot="checkbox-checkbox-input" />
<Kobalte.Control>...</Kobalte.Control>
<Kobalte.Label>...</Kobalte.Label>
<Kobalte.Description>...</Kobalte.Description>
<Kobalte.ErrorMessage>...</Kobalte.ErrorMessage>
```

## 4. 组件组合模式

### 模式一：完整封装

```typescript
// 导出类型
export interface DropdownMenuProps extends ComponentProps<typeof Kobalte> {}
export interface DropdownMenuItemProps extends ComponentProps<typeof Kobalte.Item> {}

// 封装组件
function DropdownMenuRoot(props: DropdownMenuProps) {
  return <Kobalte {...props} data-component="dropdown-menu" />
}

function DropdownMenuItem(props: ParentProps<DropdownMenuItemProps>) {
  const [local, rest] = splitProps(props, ["class", "classList", "children"])
  return (
    <Kobalte.Item {...rest} data-slot="dropdown-menu-item" classList={{...}}>
      {local.children}
    </Kobalte.Item>
  )
}

// 组合导出
export const DropdownMenu = Object.assign(DropdownMenuRoot, {
  Trigger: DropdownMenuTrigger,
  Content: DropdownMenuContent,
  Item: DropdownMenuItem,
  ...
})
```

### 模式二：直接使用 + 扩展

```typescript
<Kobalte<T, { category: string; options: T[] }>
  {...others}
  itemComponent={(itemProps) => (
    <Kobalte.Item {...itemProps} data-slot="select-item">
      ...
    </Kobalte.Item>
  )}
>
  <Kobalte.Trigger as={Button}>...</Kobalte.Trigger>
  <Kobalte.Portal>
    <Kobalte.Content>...</Kobalte.Content>
  </Kobalte.Portal>
</Kobalte>
```

### 模式三：Props 扩展

```typescript
export interface TabsProps extends ComponentProps<typeof Kobalte> {
  variant?: "normal" | "alt" | "pill" | "settings"
  orientation?: "horizontal" | "vertical"
}
```

## 5. 组件 API 使用方式

### Controlled/Uncontrolled API

```typescript
// 受控模式
<Kobalte open={local.open} onOpenChange={local.onOpenChange} />

// 非受控模式（默认）
<Kobalte defaultOpen={true} />
```

### 渲染属性 API (Render Props)

```typescript
// Select 的 value 渲染
<Kobalte.Value<T> data-slot="select-value">
  {(state) => {
    const selected = state.selectedOption()
    return local.label ? local.label(selected) : selected
  }}
</Kobalte.Value>

// Toast 的状态渲染
toaster.promise(promise, (props) => (
  <Toast data-variant={props.state === "pending" ? "loading" : ...}>
    ...
  </Toast>
))
```

### Portal API

```typescript
<Kobalte.Portal>
  <Kobalte.Content>...</Kobalte.Content>
</Kobalte.Portal>
```

### Slot 数据属性模式

项目使用 `data-slot` 属性来标识组件各部分：

```typescript
data-slot="dropdown-menu-trigger"
data-slot="dropdown-menu-content"
data-slot="dropdown-menu-item"
data-component="dropdown-menu"  // 根组件
```

## 6. Select 完整示例

```typescript
<Kobalte<T>
  placement="bottom-start"
  gutter={4}
  value={local.current}
  options={grouped()}
  optionValue={(x) => local.value?.(x) ?? x}
  optionTextValue={(x) => local.label?.(x) ?? x}
  itemComponent={(itemProps) => (
    <Kobalte.Item>
      <Kobalte.ItemLabel>{itemProps.item.rawValue}</Kobalte.ItemLabel>
      <Kobalte.ItemIndicator><Icon name="check" /></Kobalte.ItemIndicator>
    </Kobalte.Item>
  )}
>
  <Kobalte.Trigger as={Button}>...</Kobalte.Trigger>
  <Kobalte.Portal>
    <Kobalte.Content><Kobalte.Listbox /></Kobalte.Content>
  </Kobalte.Portal>
</Kobalte>
```

## 7. 最佳实践总结

| 实践             | 说明                                                          |
| ---------------- | ------------------------------------------------------------- |
| **类型安全**     | 通过 `ComponentProps<typeof Kobalte>` 继承 Kobalte 的类型定义 |
| **样式隔离**     | 使用 `data-component` 和 `data-slot` 属性进行样式 targeting   |
| **组合优于继承** | 使用 `Object.assign` 导出复合组件                             |
| **Props 透传**   | 使用 `splitProps` 分离自定义 props 和透传 props               |
| **可访问性优先** | 直接利用 Kobalte 的 ARIA 支持，无需额外处理                   |
