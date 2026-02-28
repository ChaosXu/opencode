# SDK 包分析

## 1. 包概述

**包名**: `@opencode-ai/sdk`
**版本**: 1.2.15
**类型**: TypeScript SDK
**用途**: 生成客户端/服务端 SDK

## 2. 入口点

```json
{
  "exports": {
    ".": "./src/index.ts",
    "./client": "./src/client.ts",
    "./server": "./src/server.ts",
    "./v2": {
      "types": "./dist/v2/index.d.ts",
      "default": "./src/v2/index.ts"
    },
    "./v2/client": {...},
    "./v2/gen/client": {...},
    "./v2/server": "./src/v2/server.ts"
  }
}
```

## 3. 导出模块

### 3.1 主入口

- `src/index.ts` - 主入口

### 3.2 V2 API

- `src/v2/index.ts` - V2 入口
- `src/v2/client.ts` - V2 客户端
- `src/v2/server.ts` - V2 服务器

## 4. 生成文件

SDK 包含生成的类型和客户端代码：

```
dist/
├── v2/
│   ├── index.d.ts        # 类型定义
│   ├── client.d.ts       # 客户端类型
│   └── gen/client/
│       └── index.ts      # 生成的客户端代码
```

## 5. 构建流程

```json
{
  "build": "bun ./script/build.ts"
}
```

构建脚本会：

1. 从 OpenAPI 规范生成类型
2. 生成客户端 SDK
3. 输出到 dist 目录

## 6. OpenAPI 规范

项目包含生成的 OpenAPI 规范：

- `openapi.json` - 完整的 API 定义

## 7. 使用方式

### 7.1 客户端

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
})

// 调用 API
const sessions = await client.session.list()
```

### 7.2 服务器端

```typescript
import { serve } from "@opencode-ai/sdk/server"

serve({
  // handler
})
```

## 8. V2 API 结构

V2 是新一代 API，使用代码生成：

```typescript
// 生成的类型
export interface Session {
  id: string
  title: string
  // ...
}

// 生成的客户端
export class Client {
  session: SessionAPI
  project: ProjectAPI
  // ...
}
```

## 9. 依赖

### 开发依赖

- `@hey-api/openapi-ts` - OpenAPI 生成
- `@tsconfig/node22` - TS 配置
- `typescript`

## 10. 静态结构

```
src/
├── index.ts          # 主入口
├── client.ts         # 客户端
├── server.ts         # 服务器
├── v2/               # V2 API
│   ├── index.ts
│   ├── client.ts
│   └── server.ts
└── script/
    └── build.ts      # 构建脚本
```

## 11. 总结

SDK 包提供：

1. **类型安全**: 完整的 TypeScript 类型
2. **代码生成**: 从 OpenAPI 自动生成
3. **客户端/服务器**: 双向支持
4. **版本管理**: V1 + V2 双版本
