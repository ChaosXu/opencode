# Tailwind CSS 使用分析

## 1. 配置方式

项目使用 **Tailwind CSS v4**，采用 **CSS-first 配置** 模式，没有传统的 `tailwind.config.js` 文件。

### 主题配置

```css
/* packages/ui/src/styles/tailwind/index.css */
@theme {
  --spacing: 0.25rem;
  --breakpoint-sm: 40rem;
  --breakpoint-md: 48rem;
  --breakpoint-lg: 64rem;
  --font-sans: var(--font-family-sans);
  --text-sm: var(--font-size-small);
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  /* ...更多变量映射 */
}
```

### 依赖配置

- `@tailwindcss/vite` - Vite 插件
- `tailwindcss` - 核心库

## 2. 样式系统架构

项目**不直接使用 Tailwind 工具类**（如 `className="flex p-4"`），而是基于 **CSS 自定义属性** 构建了一套语义化设计系统。

### 核心文件结构

- `theme.css` - 基础设计 token（字体、间距、圆角、阴影）
- `colors.css` - 12 种预定义颜色主题
- `index.css` - 组件样式导入（30+ 组件）

## 3. 自定义主题和样式定义

### 语义化 Token 命名

```css
/* 背景色 */
--background-base, --background-weak, --background-strong

/* 表面色 */
--surface-base, --surface-raised-base, --surface-float-base

/* 文本色 */
--text-base, --text-weak, --text-interactive-base

/* 边框色 */
--border-base, --border-hover, --border-active

/* 语义色 */
--surface-success-base, --surface-warning-base, --surface-critical-base
```

### 颜色主题种子

```typescript
// packages/ui/src/theme/types.ts
interface ThemeSeedColors {
  neutral: HexColor // 中性灰
  primary: HexColor // 主色
  success: HexColor // 成功绿
  warning: HexColor // 警告黄
  error: HexColor // 错误红
  info: HexColor // 信息蓝
  interactive: HexColor // 交互色
  diffAdd: HexColor // Diff 增加
  diffDelete: HexColor // Diff 删除
}
```

### 预定义颜色主题

项目提供 12 种颜色主题：

- gray, smoke, yuzu, cobalt, apple, ember
- solaris, lilac, coral, mint, blue, ink, amber

## 4. 响应式设计

### 自定义断点

```css
/* theme.css */
--breakpoint-sm: 40rem; /* 640px */
--breakpoint-md: 48rem; /* 768px */
--breakpoint-lg: 64rem; /* 1024px */
--breakpoint-xl: 80rem; /* 1280px */
```

### 使用方式

```css
/* CSS Modules 中 */
@media (max-width: 48rem) {
}
@media (min-width: 768px) {
}
@media (max-width: 40rem) {
}
```

## 5. 暗色模式实现

### 双重机制

#### CSS 原生暗色模式

```css
/* theme.css */
:root {
  color-scheme: light;
  /* 亮色变量定义 */

  @media (prefers-color-scheme: dark) {
    color-scheme: dark;
    /* 暗色变量覆盖 */
  }
}
```

#### 运行时主题切换

```typescript
// packages/ui/src/theme/loader.ts
function buildThemeCss(light: ResolvedTheme, dark: ResolvedTheme, themeId: string) {
  return `
    html[data-theme="${themeId}"] {
      color-scheme: light;
      ${lightCss}
      
      @media (prefers-color-scheme: dark) {
        color-scheme: dark;
        ${darkCss}
      }
    }
  `
}

export function setColorScheme(scheme: "light" | "dark" | "auto") {
  if (scheme === "auto") {
    document.documentElement.style.removeProperty("color-scheme")
  } else {
    document.documentElement.style.setProperty("color-scheme", scheme)
  }
}
```

## 6. 组件样式模式

组件使用 **CSS 属性选择器** 而非 Tailwind 类：

```css
/* button.css */
[data-component="button"] {
  display: inline-flex;
  align-items: center;
  border-radius: var(--radius-md);

  &[data-variant="primary"] {
    background-color: var(--button-primary-base);
    color: var(--icon-invert-base);
  }

  &[data-variant="ghost"] {
    background-color: transparent;
    color: var(--text-strong);
  }
}
```

## 7. 使用模式总结

| 特性     | 实现方式                       |
| -------- | ------------------------------ |
| 配置方式 | Tailwind v4 CSS-first (@theme) |
| 类使用   | 不使用工具类，使用 CSS 变量    |
| 主题系统 | 12 种预定义颜色 + 自定义种子   |
| 暗色模式 | CSS @media + JS 运行时切换     |
| 响应式   | 自定义断点 + CSS @media        |
| 组件样式 | CSS 属性选择器 + 语义化变量    |

## 8. 最佳实践

这种方式的优点：

1. **样式隔离**：不依赖 Tailwind 工具类，避免类名冲突
2. **语义化命名**：使用有意义的 CSS 变量名，如 `--surface-base`
3. **主题灵活性**：支持 12 种颜色主题和运行时切换
4. **类型安全**：通过 TypeScript 类型定义主题 seed
5. **性能优化**：CSS-first 配置在构建时生成最优 CSS
