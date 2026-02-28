# Util 包分析

## 1. 包概述

**包名**: `@opencode-ai/util`
**版本**: 1.2.15
**类型**: 共享工具库
**用途**: 供其他包复用的工具函数

## 2. 入口点

```json
{
  "exports": {
    "./*": "./src/*.ts"
  }
}
```

## 3. 导出模块

共有 **11 个工具模块**：

| 模块            | 文件          | 说明       |
| --------------- | ------------- | ---------- |
| `error.ts`      | error.ts      | 错误处理   |
| `encode.ts`     | encode.ts     | 编码/解码  |
| `retry.ts`      | retry.ts      | 重试机制   |
| `fn.ts`         | fn.ts         | 函数工具   |
| `lazy.ts`       | lazy.ts       | 懒加载     |
| `identifier.ts` | identifier.ts | 标识符     |
| `path.ts`       | path.ts       | 路径处理   |
| `slug.ts`       | slug.ts       | Slug 生成  |
| `iife.ts`       | iife.ts       | IIFE 工具  |
| `binary.ts`     | binary.ts     | 二进制处理 |
| `array.ts`      | array.ts      | 数组工具   |

## 4. 核心模块详解

### 4.1 Error 模块

```typescript
// 错误类定义
export class NamedError extends Error {
  name: string
  data?: any
  cause?: Error

  static create<T>(name: string, schema?: z.ZodType): (data: T, options?: { cause?: Error }) => NamedError
}
```

### 4.2 Lazy 模块

```typescript
// 懒加载实现
export function lazy<T>(fn: () => T): () => T
export function lazyasync<T>(fn: () => Promise<T>): () => Promise<T>
```

### 4.3 Retry 模块

```typescript
// 重试机制
export async function retry<T>(
  fn: () => Promise<T>,
  options?: {
    retries?: number
    delay?: number
    backoff?: number
  },
): Promise<T>
```

### 4.4 Identifier 模块

```typescript
// 标识符处理
export function ulid(): string
export function uuidv7(): string
```

### 4.5 Path 模块

```typescript
// 路径工具
export function normalize(path: string): string
export function relative(from: string, to: string): string
export function join(...paths: string[]): string
```

## 5. 依赖

### 生产依赖

- `zod` - 验证

### 开发依赖

- `typescript` - 类型检查
- `@types/bun` - Bun 类型

## 6. 使用方式

```typescript
// 直接导入
import { NamedError } from "@opencode-ai/util/error"
import { lazy } from "@opencode-ai/util/lazy"
import { retry } from "@opencode-ai/util/retry"
```

## 7. 总结

Util 包的特点：

1. **无状态**: 所有函数都是纯函数
2. **轻量**: 依赖少，只有 zod
3. **复用**: 被多个包引用
4. **类型安全**: 完整的 TypeScript 类型
