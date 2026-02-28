# Desktop 包分析

## 1. 包概述

**包名**: `@opencode-ai/desktop`
**版本**: 1.2.15
**类型**: 桌面应用 (Tauri)
**用途**: OpenCode 原生桌面客户端

## 2. 入口点

通过 Tauri 配置和 Vite 定义

```json
{
  "scripts": {
    "dev": "vite",
    "build": "bun run typecheck && vite build",
    "tauri": "tauri"
  }
}
```

## 3. 技术栈

- **框架**: Tauri v2
- **前端**: SolidJS + @opencode-ai/app
- **UI**: @opencode-ai/ui
- **桌面**: @tauri-apps/api

## 4. Tauri 插件

| 插件                                   | 说明     |
| -------------------------------------- | -------- |
| `@tauri-apps/plugin-clipboard-manager` | 剪贴板   |
| `@tauri-apps/plugin-deep-link`         | 深度链接 |
| `@tauri-apps/plugin-dialog`            | 对话框   |
| `@tauri-apps/plugin-opener`            | 打开外部 |
| `@tauri-apps/plugin-os`                | 操作系统 |
| `@tauri-apps/plugin-notification`      | 通知     |
| `@tauri-apps/plugin-process`           | 进程     |
| `@tauri-apps/plugin-shell`             | Shell    |
| `@tauri-apps/plugin-store`             | 存储     |
| `@tauri-apps/plugin-updater`           | 更新     |
| `@tauri-apps/plugin-http`              | HTTP     |
| `@tauri-apps/plugin-window-state`      | 窗口状态 |

## 5. 开发

### 开发命令

```bash
bun run --cwd packages/desktop tauri dev
```

### 构建命令

```bash
bun run --cwd packages/desktop tauri build
```

## 6. 平台支持

- macOS (Apple Silicon/Intel)
- Windows
- Linux (.deb, .rpm, AppImage)

## 7. 依赖

### 生产依赖

- `@opencode-ai/app` - 应用
- `@opencode-ai/ui` - UI
- `@solid-primitives/i18n` - 国际化
- `@solid-primitives/storage` - 存储
- `@tauri-apps/api` - Tauri API

### 开发依赖

- `@tauri-apps/cli` - Tauri CLI
- `vite` - 构建
- `typescript`

## 8. 静态结构

```
packages/desktop/
├── src/                # 源码
├── scripts/            # 构建脚本
├── src-tauri/          # Rust 代码
│   ├── src/
│   ├── Cargo.toml
│   └── tauri.conf.json
└── package.json
```

## 9. 使用注意

- 不能直接调用 `invoke`，需要使用生成的 bindings
- bindings 位于 `packages/desktop/src/bindings.ts`

## 10. 总结

Desktop 包提供：

- 原生桌面应用
- Tauri v2 架构
- 多平台支持
- 丰富的系统集成
