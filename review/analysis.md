# OpenCode 项目分析文档

## 项目概述

OpenCode 是一个开源的 AI 编程助手，采用客户端/服务器架构设计。项目使用 Bun 作为运行时，TypeScript 作为主要开发语言，采用 monorepo 方式进行项目管理。

### 核心特性

- **100% 开源** - 不与任何特定模型提供商绑定
- **开箱即用的 LSP 支持** - 内置语言服务器协议支持
- **TUI 优先** - 终端用户界面优先设计
- **客户端/服务器架构** - 支持远程驱动

### 技术栈

| 类别      | 技术             |
| --------- | ---------------- |
| 运行时    | Bun 1.3.10       |
| 语言      | TypeScript 5.8.2 |
| 构建工具  | Turbo 2.5.6      |
| 包管理    | npm workspaces   |
| 前端框架  | SolidJS          |
| UI 组件库 | Kobalte          |
| 样式方案  | Tailwind CSS 4.1 |
| 数据库    | Drizzle ORM      |
| API 框架  | Hono             |
| AI SDK    | Vercel AI SDK    |

---

## 包结构分析

### 1. 核心包 (packages/opencode)

**用途**: OpenCode CLI 核心工具

**架构模块**:

| 模块        | 职责                                 |
| ----------- | ------------------------------------ |
| `agent/`    | Agent 系统实现，包含 agent.ts 主逻辑 |
| `session/`  | 会话管理 (18个子目录，最复杂的模块)  |
| `tool/`     | 工具注册与执行 (46个工具文件)        |
| `server/`   | 服务器实现                           |
| `cli/`      | 命令行接口                           |
| `config/`   | 配置管理                             |
| `project/`  | 项目管理                             |
| `provider/` | AI 提供商集成                        |
| `lsp/`      | 语言服务器协议支持                   |
| `mcp/`      | MCP (Model Context Protocol) 支持    |
| `storage/`  | 存储层                               |
| `util/`     | 工具函数 (28个子目录)                |

**工具系统** (46个工具):

- 文件操作: read, write, edit, glob, ls
- 代码搜索: grep, codesearch
- Web: webfetch, websearch
- 执行: bash, task
- 开发辅助: lsp, todo, skill
- 其他: patch, question, truncation

**Agent 类型**:

- `build` - 默认的完整访问权限 Agent
- `plan` - 只读 Agent，用于代码探索和规划
- `general` - 通用子 Agent，用于复杂搜索和多步骤任务

### 2. UI 包

#### packages/ui

**用途**: 共享 UI 组件库

**技术栈**:

- SolidJS 1.9.10
- Kobalte 0.13.11 (UI 组件库)
- Tailwind CSS 4.1.11
- Vite 7.1.4

**组件结构**:

- `src/components/` - 组件实现
- `src/theme/` - 主题系统
- `src/context/` - React Context
- `src/hooks/` - 自定义 Hooks

#### packages/app

**用途**: 独立的应用程序包

**技术栈**: 同 packages/ui

### 3. Web 相关包

#### packages/web

**用途**: 文档网站

**技术栈**:

- Astro 5.x
- Starlight (文档框架)
- Tailwind CSS

**功能**:

- 公开文档 (https://opencode.ai)
- landing page
- 分享功能 (s/[id] 路由)
- 多语言支持 (i18n)

#### packages/console

**用途**: 后端管理控制台

**子包结构**:

- `console/app/` - 主 Web 应用 (SolidJS)
- `console/function/` - Serverless 函数 (Hono)
- `console/core/` - 核心逻辑
- `console/mail/` - 邮件模板
- `console/resource/` - 资源定义

**功能**:

- Zen API 路由
- Workspace 管理
- Stripe 集成 (支付)
- 邮件功能

### 4. 支持包

#### packages/sdk

**用途**: SDK 生成

**内容**:

- TypeScript SDK (packages/sdk/js)
- VSCode 扩展 SDK (sdks/vscode)
- OpenAPI 规范 (openapi.json)

#### packages/script

**用途**: 脚本工具

**功能**:

- JavaScript SDK 构建脚本
- 其他构建/部署脚本

#### packages/util

**用途**: 共享工具函数库

**子目录** (8个):

- 各类工具函数
- 供其他包复用

#### packages/function

**用途**: 云函数/Hono 服务

**技术栈**: Hono + Zod

### 5. 企业级包

#### packages/enterprise

**用途**: 企业功能

**功能**: 企业级特性实现

#### packages/identity

**用途**: 身份图标资源

**内容**: SVG 图标文件

#### packages/plugin

**用途**: 插件系统

**功能**: 扩展机制支持

#### packages/slack

**用途**: Slack 集成

**功能**: Slack 机器人/应用集成

#### packages/docs

**用途**: 文档资源

#### packages/containers

**用途**: 容器化配置

**内容**: Docker 相关配置

#### packages/desktop

**用途**: 桌面应用

**技术栈**:

- Tauri 2.x
- SolidJS
- Tailwind CSS

**功能**:

- macOS (Apple Silicon/Intel)
- Windows
- Linux (.deb, .rpm, AppImage)

### 6. 其他包

#### packages/extensions

**用途**: 扩展相关

#### packages/storybook

**用途**: 组件文档

---

## 架构特点

### 1. Monorepo 结构

使用 Turbo 进行工作空间管理，配置于 root package.json:

```json
"workspaces": {
  "packages": [
    "packages/*",
    "packages/console/*",
    "packages/sdk/js",
    "packages/slack"
  ]
}
```

### 2. 代码风格

遵循 AGENTS.md 中的规范:

- 单词变量名
- 避免 `any` 类型
- 使用 Bun APIs
- 函数式数组方法
- 避免不必要的解构
- 优先使用 `const`
- 避免 `else` 语句 (早返回)

### 3. 数据库

使用 Drizzle ORM:

- SQLite 表定义
- snake_case 字段命名
- TypeScript 类型推断

### 4. API 设计

- 使用 Hono 框架
- Zod 进行验证
- OpenAPI 规范

---

## 依赖关系

### 核心依赖

| 包                  | 依赖          |
| ------------------- | ------------- |
| @opencode-ai/sdk    | workspace:\*  |
| @opencode-ai/script | workspace:\*  |
| @opencode-ai/plugin | workspace:\*  |
| hono                | 4.10.7        |
| ai (AI SDK)         | 5.0.124       |
| solid-js            | 1.9.10        |
| drizzle-orm         | 1.0.0-beta.12 |
| zod                 | 4.1.8         |

---

## 部署方式

### CLI 工具

```bash
# YOLO
curl -fsSL https://opencode.ai/install | bash

# npm
npm i -g opencode-ai@latest

# Homebrew
brew install opencode
```

### 桌面应用

- DMG (macOS)
- EXE (Windows)
- DEB/RPM/AppImage (Linux)

### 云端服务

- Console 后端 (Hono serverless)
- Web 文档 (Astro)

---

## 总结

OpenCode 是一个架构清晰的 AI 编程助手项目:

1. **模块化设计**: 核心逻辑与 UI、文档、企业功能分离
2. **技术选型现代**: Bun + SolidJS + Drizzle
3. **工具系统丰富**: 46个内置工具支持各类开发任务
4. **多平台支持**: CLI、Web、Desktop 多端覆盖
5. **代码规范统一**: 明确的编码风格指南

项目采用 monorepo 方式管理，使用 Turbo 进行构建协调，通过 workspaces 整合多个相关包。
