# UI 包分析

## 1. 包概述

**包名**: `@opencode-ai/ui`
**版本**: 1.2.15
**类型**: UI 组件库
**用途**: OpenCode 共享 UI 组件

## 2. 入口点

```json
{
  "exports": {
    "./package.json": "./package.json",
    "./*": "./src/components/*.tsx",
    "./i18n/*": "./src/i18n/*.ts",
    "./hooks": "./src/hooks/index.ts",
    "./context": "./src/context/index.ts",
    "./context/*": "./src/context/*.tsx",
    "./styles": "./src/styles/index.css",
    "./styles/tailwind": "./src/styles/tailwind/index.css",
    "./theme": "./src/theme/index.ts",
    "./theme/*": "./src/theme/*.ts",
    "./theme/context": "./src/theme/context.tsx"
  }
}
```

## 3. 组件库 (100+ 组件)

### 表单组件

| 组件       | 文件            | 说明     |
| ---------- | --------------- | -------- |
| Button     | button.tsx      | 按钮     |
| TextField  | text-field.tsx  | 文本输入 |
| Checkbox   | checkbox.tsx    | 复选框   |
| Switch     | switch.tsx      | 开关     |
| Select     | select.tsx      | 选择器   |
| RadioGroup | radio-group.tsx | 单选组   |

### 布局组件

| 组件        | 文件            | 说明   |
| ----------- | --------------- | ------ |
| Tabs        | tabs.tsx        | 标签页 |
| Accordion   | accordion.tsx   | 手风琴 |
| Collapsible | collapsible.tsx | 可折叠 |
| List        | list.tsx        | 列表   |

### 弹出层组件

| 组件         | 文件              | 说明     |
| ------------ | ----------------- | -------- |
| Dialog       | dialog.tsx        | 对话框   |
| DropdownMenu | dropdown-menu.tsx | 下拉菜单 |
| ContextMenu  | context-menu.tsx  | 右键菜单 |
| Popover      | popover.tsx       | 弹出层   |
| Tooltip      | tooltip.tsx       | 提示     |
| HoverCard    | hover-card.tsx    | 悬停卡片 |

### 信息展示组件

| 组件           | 文件                | 说明     |
| -------------- | ------------------- | -------- |
| Toast          | toast.tsx           | 通知     |
| Progress       | progress.tsx        | 进度条   |
| ProgressCircle | progress-circle.tsx | 圆形进度 |
| Spinner        | spinner.tsx         | 加载中   |
| Avatar         | avatar.tsx          | 头像     |
| Badge          | tag.tsx             | 标签     |

### 会话组件

| 组件          | 文件               | 说明     |
| ------------- | ------------------ | -------- |
| SessionTurn   | session-turn.tsx   | 消息     |
| SessionReview | session-review.tsx | 审核     |
| MessagePart   | message-part.tsx   | 消息片段 |
| MessageNav    | message-nav.tsx    | 消息导航 |

### 代码相关组件

| 组件        | 文件             | 说明      |
| ----------- | ---------------- | --------- |
| Diff        | diff.tsx         | Diff 展示 |
| Code        | code.tsx         | 代码展示  |
| DiffChanges | diff-changes.tsx | Diff 变更 |

### 其他组件

| 组件         | 文件              | 说明          |
| ------------ | ----------------- | ------------- |
| File         | file.tsx          | 文件          |
| FileIcon     | file-icon.tsx     | 文件图标      |
| ProviderIcon | provider-icon.tsx | 提供商图标    |
| Markdown     | markdown.tsx      | Markdown 渲染 |
| ImagePreview | image-preview.tsx | 图片预览      |

## 4. 主题系统

### 核心文件

- `src/theme/index.ts` - 主题入口
- `src/theme/loader.ts` - 运行时主题加载
- `src/theme/resolve.ts` - Token 解析
- `src/theme/types.ts` - 类型定义

### 颜色主题

12 种预定义主题：

- gray, smoke, yuzu, cobalt, apple, ember
- solaris, lilac, coral, mint, blue, ink, amber

## 5. 样式系统

### 配置文件

- `src/styles/index.css` - 主样式入口
- `src/styles/tailwind/index.css` - Tailwind v4 配置
- `src/styles/theme.css` - 主题变量
- `src/styles/colors.css` - 颜色变量

### 设计原则

- 使用 CSS 自定义属性
- 不直接使用 Tailwind 工具类
- 基于语义化 token 构建组件

## 6. Context 和 Hooks

### Context

- `theme/context.tsx` - 主题 Context
- `i18n.tsx` - 国际化

### Hooks

- `src/hooks/index.ts` - 导出所有 hooks

## 7. 技术栈

- **框架**: SolidJS
- **UI 库**: Kobalte
- **样式**: Tailwind CSS v4
- **构建**: Vite

## 8. 静态结构

```
src/
├── components/           # 100+ 组件
│   ├── button.tsx
│   ├── dialog.tsx
│   ├── select.tsx
│   └── ...
├── hooks/              # 自定义 Hooks
├── i18n/               # 国际化
├── context/            # Context
├── theme/              # 主题系统
│   ├── index.ts
│   ├── loader.ts
│   ├── resolve.ts
│   ├── types.ts
│   └── context.tsx
├── styles/             # 样式
│   ├── index.css
│   ├── theme.css
│   ├── colors.css
│   └── tailwind/
├── assets/             # 静态资源
│   ├── fonts/
│   └── audio/
└── pierre/            # Diff 库封装
```

## 9. 组件导出模式

### 方式一：直接导出

```typescript
// 通过 package.json exports 直接导出
import { Button } from "@opencode-ai/ui/button"
import { Dialog } from "@opencode-ai/ui/dialog"
```

### 方式二：按目录导出

```typescript
// 通过通配符导出
import { Button } from "@opencode-ai/ui/button"
```

## 10. Kobalte 集成

项目使用 Kobalte 作为底层 UI 库：

```typescript
import { Dialog as Kobalte } from "@kobalte/core/dialog"
import { Select as Kobalte } from "@kobalte/core/select"
```

所有组件都基于 Kobalte 封装，添加了：

- 语义化 CSS 变量
- 项目特定的设计风格
- 类型安全的 Props

## 11. 依赖

### 核心依赖

- `@kobalte/core` - UI 组件库
- `solid-js` - 框架
- `tailwindcss` - 样式
- `shiki` - 代码高亮
- `marked` - Markdown 解析

### 开发依赖

- `vite` - 构建
- `@tailwindcss/vite` - Vite 插件
