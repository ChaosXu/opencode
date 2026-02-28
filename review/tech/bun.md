# Bun 运行时使用分析

## 1. 概述

OpenCode 项目使用 **Bun 1.3.10** 作为运行时环境，充分利用了 Bun 的高性能特性和现代 API。

## 2. 核心 API 使用

### 2.1 Bun.$ (Shell 命令执行)

**文件**: `packages/opencode/src/tool/bash.ts`

```typescript
import { $ } from "bun"

const resolved = await $`realpath ${arg}`
  .cwd(cwd)
  .quiet()
  .nothrow()
  .text()
  .then((x) => x.trim())
```

**链式调用方法**:

- `.cwd(dir)` - 设置工作目录
- `.quiet()` - 静默输出
- `.nothrow()` - 不抛出异常
- `.text()` - 返回文本结果
- `.json()` - 返回 JSON 结果

### 2.2 Bun.file (文件系统操作)

**文件**: `packages/opencode/src/snapshot/index.ts`

```typescript
// 读取文件内容
const text = await Bun.file(file)
  .text()
  .catch(() => "")

// Bun.write 写入文件
await Bun.write(target, text)
await Bun.write(target, "") // 清空文件
```

### 2.3 Bun.serve (HTTP/WebSocket 服务器)

**文件**: `packages/opencode/src/server/server.ts`

```typescript
const server = Bun.serve({
  hostname: opts.hostname,
  port: opts.port,
  fetch: App().fetch,
  websocket: websocket,
})
```

### 2.4 Bun.spawn (子进程)

项目使用 `Process.spawn` 封装了 Bun 进程：

```typescript
const result = Process.spawn([which(), ...cmd], {
  cwd,
  stdout: "pipe",
  stderr: "pipe",
  env: {
    ...process.env,
    BUN_BE_BUN: "1", // 强制使用 Bun 运行
  },
})
```

### 2.5 Bun.env / Bun.stderr / Bun.stdout

```typescript
// 环境变量
BUN_BE_BUN: "1"

// 终端输出
Bun.stderr.write(EOL)
Bun.stderr.write(message.join(" "))
```

### 2.6 Bun.semver (版本比较)

**文件**: `packages/opencode/src/bun/registry.ts`

```typescript
import { semver } from "bun"

const isOutdated = await PackageRegistry.isOutdated(pkg, cachedVersion)
return semver.order(cachedVersion, latestVersion) === -1
```

## 3. Bun 模块入口

| 文件                                    | 作用                                            |
| --------------------------------------- | ----------------------------------------------- |
| `packages/opencode/src/bun/index.ts`    | BunProc 命名空间：封装命令执行和插件安装        |
| `packages/opencode/src/bun/registry.ts` | PackageRegistry：使用 Bun.semver 进行包版本检查 |

## 4. API 使用频率

| API               | 使用次数 | 主要文件                      |
| ----------------- | -------- | ----------------------------- |
| Bun.$             | 2        | plugin/index.ts, bash.ts      |
| Bun.file          | 19       | 多个脚本和工具文件            |
| Bun.serve         | 7        | server.ts, workspace-serve.ts |
| Bun.spawn         | 2        | bash.ts                       |
| Bun.env           | 36       | 测试和配置文件                |
| Bun.stderr/stdout | 3        | ui.ts                         |
| Bun.color         | 1        | ui.ts                         |
| Bun.randomUUIDv7  | 1        | workspace-serve.ts            |
| Bun.semver        | 1        | registry.ts                   |

## 5. 测试框架

项目使用 `bun:test` 作为测试框架：

```typescript
import { describe, expect, test } from "bun:test"

describe("BunProc registry configuration", () => {
  test("should not contain hardcoded registry parameters", async () => {
    // 测试逻辑
  })
})
```

## 6. 最佳实践总结

1. **优先使用 Bun 原生 API**：项目充分利用 Bun 的高性能 API
2. **Bun.$ 用于命令执行**：链式调用提供良好的开发者体验
3. **Bun.file/write 用于文件操作**：简洁的 API 设计
4. **Bun.serve 用于 HTTP 服务**：内置 WebSocket 支持
5. **bun:test 用于测试**：零配置的测试体验
