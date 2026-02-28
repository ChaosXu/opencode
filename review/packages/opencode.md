# OpenCode 核心包分析

## 1. 包概述

**包名**: `opencode` (CLI)
**版本**: 1.2.15
**类型**: CLI 工具 + 核心库
**运行时**: Bun

## 2. 入口点

### 2.1 CLI 入口

```json
"bin": {
  "opencode": "./bin/opencode"
}
```

### 2.2 主入口文件

- `src/index.ts` - CLI 主入口，使用 yargs 解析命令行参数

### 2.3 开发入口

```json
"dev": "bun run --conditions=browser ./src/index.ts"
```

## 3. 导出模块

### 核心模块（23个子模块）

| 模块         | 文件路径                  | 职责                      |
| ------------ | ------------------------- | ------------------------- |
| `session`    | `src/session/index.ts`    | 会话管理，最复杂的模块    |
| `agent`      | `src/agent/index.ts`      | Agent 系统实现            |
| `tool`       | `src/tool/index.ts`       | 工具注册与执行 (46个工具) |
| `server`     | `src/server/server.ts`    | HTTP 服务器               |
| `provider`   | `src/provider/index.ts`   | AI 提供商集成             |
| `lsp`        | `src/lsp/index.ts`        | 语言服务器协议支持        |
| `mcp`        | `src/mcp/index.ts`        | MCP 支持                  |
| `config`     | `src/config/index.ts`     | 配置管理                  |
| `project`    | `src/project/index.ts`    | 项目管理                  |
| `storage`    | `src/storage/index.ts`    | 存储层                    |
| `auth`       | `src/auth/index.ts`       | 认证系统                  |
| `file`       | `src/file/index.ts`       | 文件操作                  |
| `bus`        | `src/bus/index.ts`        | 事件总线                  |
| `bun`        | `src/bun/index.ts`        | Bun 运行时封装            |
| `skill`      | `src/skill/index.ts`      | Skill 系统                |
| `snapshot`   | `src/snapshot/index.ts`   | 快照管理                  |
| `pty`        | `src/pty/index.ts`        | 终端管理                  |
| `plugin`     | `src/plugin/index.ts`     | 插件系统                  |
| `permission` | `src/permission/index.ts` | 权限管理                  |
| `patch`      | `src/patch/index.ts`      | Patch 管理                |
| `worktree`   | `src/worktree/index.ts`   | Worktree 管理             |
| `question`   | `src/question/index.ts`   | 问答系统                  |
| `scheduler`  | `src/scheduler/index.ts`  | 调度器                    |

## 4. CLI 命令

共有 **20 个 CLI 命令**：

| 命令              | 文件                             | 说明                |
| ----------------- | -------------------------------- | ------------------- |
| `run`             | `src/cli/cmd/run.ts`             | 运行 AI 对话（TUI） |
| `serve`           | `src/cli/cmd/serve.ts`           | 启动 HTTP 服务器    |
| `workspace-serve` | `src/cli/cmd/workspace-serve.ts` | Workspace 服务器    |
| `auth`            | `src/cli/cmd/auth.ts`            | 认证管理            |
| `agent`           | `src/cli/cmd/agent.ts`           | Agent 操作          |
| `upgrade`         | `src/cli/cmd/upgrade.ts`         | 升级                |
| `uninstall`       | `src/cli/cmd/uninstall.ts`       | 卸载                |
| `models`          | `src/cli/cmd/models.ts`          | 模型管理            |
| `generate`        | `src/cli/cmd/generate.ts`        | 代码生成            |
| `web`             | `src/cli/cmd/web.ts`             | Web 界面            |
| `stats`           | `src/cli/cmd/stats.ts`           | 统计信息            |
| `mcp`             | `src/cli/cmd/mcp.ts`             | MCP 管理            |
| `github`          | `src/cli/cmd/github.ts`          | GitHub 集成         |
| `pr`              | `src/cli/cmd/pr.ts`              | PR 管理             |
| `export`          | `src/cli/cmd/export.ts`          | 导出                |
| `import`          | `src/cli/cmd/import.ts`          | 导入                |
| `session`         | `src/cli/cmd/session.ts`         | 会话管理            |
| `db`              | `src/cli/cmd/db.ts`              | 数据库操作          |
| `acp`             | `src/cli/cmd/acp.ts`             | ACP 命令            |
| `debug`           | `src/cli/cmd/debug/*`            | 调试命令            |

## 5. 工具系统

### 46 个内置工具

- **文件操作**: read, write, edit, glob, ls
- **代码搜索**: grep, codesearch
- **Web**: webfetch, websearch
- **执行**: bash, task
- **开发辅助**: lsp, todo, skill
- **其他**: patch, question, truncation

## 6. Agent 类型

| 类型      | 说明                                   |
| --------- | -------------------------------------- |
| `build`   | 默认的完整访问权限 Agent               |
| `plan`    | 只读 Agent，用于代码探索和规划         |
| `general` | 通用子 Agent，用于复杂搜索和多步骤任务 |

## 7. 静态结构

```
src/
├── index.ts              # CLI 主入口
├── cli/                  # 命令行接口
│   ├── cmd/             # 20 个命令
│   ├── ui.ts            # TUI 实现
│   └── ...
├── agent/                # Agent 系统
├── session/              # 会话管理 (最复杂)
├── tool/                 # 工具系统 (46个工具)
├── server/               # HTTP 服务器
├── provider/             # AI 提供商
├── storage/              # 存储层 (Drizzle ORM)
├── config/               # 配置管理
├── project/              # 项目管理
├── lsp/                  # LSP 支持
├── mcp/                  # MCP 协议
├── auth/                 # 认证
├── file/                 # 文件操作
├── bus/                  # 事件总线
├── bun/                  # Bun 封装
├── skill/                # Skill 系统
├── snapshot/             # 快照
├── pty/                  # 终端
├── plugin/               # 插件系统
├── permission/           # 权限
├── patch/                # Patch
├── worktree/             # Worktree
├── question/             # 问答
└── scheduler/            # 调度
```

## 8. 动态运行过程

### 8.1 启动流程

```typescript
// src/index.ts
1. 设置全局异常处理
2. yargs 解析命令行参数
3. 加载配置文件
4. 初始化数据库
5. 加载插件和 Skill
6. 执行对应命令
```

### 8.2 核心命令 run 执行流程

```
run 命令
  ↓
初始化 Project (项目目录)
  ↓
初始化 Session (会话)
  ↓
启动 TUI (终端界面)
  ↓
等待用户输入
  ↓
构建 Message
  ↓
调用 Agent.process()
  ↓
LLM.stream() → AI Provider
  ↓
处理流式响应
  ↓
工具调用处理
  ↓
更新 UI 和存储
```

### 8.3 Server 模式

```
serve 命令
  ↓
Bun.serve() 启动 HTTP 服务器
  ↓
Hono 中间件配置
  ↓
注册路由 (session, project, file, etc.)
  ↓
处理 HTTP 请求
  ↓
流式响应 / WebSocket
```

## 9. 数据库

使用 **Drizzle ORM** + **SQLite**：

- 表定义: `src/**/*.sql.ts`
- 迁移: `bun run db generate`
- 存储路径: `~/.local/share/opencode/opencode.db`

## 10. 依赖概览

### 核心依赖

- `ai` - Vercel AI SDK
- `@ai-sdk/*` - 多种模型提供商
- `hono` - Web 框架
- `drizzle-orm` - ORM
- `solid-js` - UI 框架
- `zod` - 验证
- `yargs` - CLI 解析
- `@modelcontextprotocol/sdk` - MCP
- `@kobalte/core` - UI 组件

## 11. 总结

OpenCode 包的架构特点：

1. **模块化设计**: 23 个独立模块各司其职
2. **CLI 优先**: 20 个子命令满足各种操作需求
3. **工具系统**: 46 个内置工具覆盖开发任务
4. **多 Provider 支持**: 集成多种 AI 模型提供商
5. **插件机制**: 支持扩展能力
6. **本地优先**: SQLite 存储保证离线可用
