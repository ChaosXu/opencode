# App 包分析

## 1. 包概述

**包名**: `@opencode-ai/app`
**版本**: 1.2.15
**类型**: Web 应用（SolidJS）
**用途**: OpenCode 的 Web 界面

## 2. 入口点

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./vite": "./vite.js",
    "./index.css": "./src/index.css"
  }
}
```

## 3. 导出接口

```typescript
// src/index.ts
export { AppBaseProviders, AppInterface } from "./app"
export { useCommand } from "./context/command"
export { type DisplayBackend, type Platform, PlatformProvider } from "./context/platform"
export { ServerConnection } from "./context/server"
export { handleNotificationClick } from "./utils/notification-click"
```

## 4. 主要模块

### 4.1 核心模块

| 模块         | 路径              | 说明                  |
| ------------ | ----------------- | --------------------- |
| `app`        | `src/app.tsx`     | 主应用组件            |
| `context`    | `src/context/`    | 各种 Context Provider |
| `components` | `src/components/` | UI 组件               |
| `pages`      | `src/pages/`      | 页面组件              |
| `utils`      | `src/utils/`      | 工具函数              |

### 4.2 Context 模块

- `command` - 命令系统
- `platform` - 平台适配
- `server` - 服务器连接
- `settings` - 设置管理
- `sync` - 同步状态
- `layout` - 布局管理
- `file` - 文件状态
- `permission` - 权限

### 4.3 页面模块

- `session` - 会话页面
- `layout` - 布局页面

## 5. 技术栈

- **框架**: SolidJS
- **路由**: `@solidjs/router`
- **UI 组件**: `@opencode-ai/ui`（基于 Kobalte）
- **样式**: Tailwind CSS
- **构建**: Vite
- **测试**: Playwright (E2E) + Happy DOM (Unit)

## 6. 开发模式

### 本地开发

```bash
# 后端 (packages/opencode)
bun run --conditions=browser ./src/index.ts serve --port 4096

# 前端 (packages/app)
bun dev -- --port 4444

# 访问 http://localhost:4444
```

### E2E 测试

```bash
bun run test:e2e:local
```

## 7. 静态结构

```
src/
├── index.ts              # 导出入口
├── index.tsx             # 应用入口
├── app.tsx               # 主应用组件
├── app.css               # 全局样式
├── happydom.ts           # 测试配置
├── vite.js               # Vite 配置
├── context/              # Context Providers
│   ├── command.ts
│   ├── platform.ts
│   ├── server.ts
│   ├── settings.ts
│   ├── sync.ts
│   ├── layout.ts
│   ├── file.ts
│   └── permission.ts
├── components/            # 通用组件
├── pages/                # 页面
│   ├── session/
│   └── layout/
└── utils/                # 工具函数
```

## 8. 动态运行过程

```
1. Vite 启动开发服务器
2. 加载 index.tsx
3. 初始化 AppBaseProviders (Context)
4. 渲染 App 组件
5. 路由匹配
6. 加载对应页面组件
7. 建立 ServerConnection (WebSocket)
8. 等待用户交互
```

## 9. 与 OpenCode 包的关系

- `@opencode-ai/app` 是 Web 客户端
- 通过 WebSocket 与 `@opencode` 服务器通信
- `@opencode-ai/ui` 提供 UI 组件库

## 10. 依赖

### 核心依赖

- `@opencode-ai/sdk` - SDK
- `@opencode-ai/ui` - UI 组件
- `@opencode-ai/util` - 工具库
- `solid-js` - 框架
- `@solidjs/router` - 路由
- `@kobalte/core` - UI 组件库
- `@solid-primitives/*` - 工具库

### 开发依赖

- `vite` - 构建
- `playwright` - E2E 测试
- `@happy-dom/global-registrator` - 单元测试
- `tailwindcss` - 样式
