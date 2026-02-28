# TypeScript 使用分析

## 1. 配置概览

项目采用**多包 Monorepo 结构**，共 22 个 tsconfig.json 文件。

### 根级配置

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {}
}
```

### opencode 核心包配置

```json
{
  "extends": "@tsconfig/bun/tsconfig.json",
  "compilerOptions": {
    "jsx": "preserve",
    "jsxImportSource": "@opentui/solid",
    "lib": ["ESNext", "DOM", "DOM.Iterable"],
    "types": [],
    "noUncheckedIndexedAccess": false,
    "customConditions": ["browser"],
    "paths": {
      "@/*": ["./src/*"],
      "@tui/*": ["./src/cli/cmd/tui/*"]
    }
  }
}
```

### app 前端包配置

```json
{
  "compilerOptions": {
    "composite": true,
    "target": "ESNext",
    "module": "ESNext",
    "strict": true,
    "jsx": "preserve",
    "jsxImportSource": "solid-js",
    "isolatedModules": true,
    "paths": { "@/*": ["./src/*"] }
  }
}
```

## 2. 类型定义最佳实践

### Zod 作为核心验证库

项目广泛使用 **Zod** 进行运行时类型验证和类型推断：

```typescript
// 使用 z.infer 从 Zod schema 推断 TypeScript 类型
export const TextPart = PartBase.extend({
  type: z.literal("text"),
  text: z.string(),
}).meta({ ref: "TextPart" })

export type TextPart = z.infer<typeof TextPart>
```

### interface vs type 使用模式

- **优先使用 `type`**：用于类型别名、联合类型、Zod 推断类型
- **使用 `interface`**：仅在需要扩展或需要明确声明结构时使用

```typescript
// 类型别名
type PartData = Omit<MessageV2.Part, "id" | "sessionID" | "messageID">
type InfoData = Omit<MessageV2.Info, "id" | "sessionID">
```

### Drizzle Schema 定义

- 表名和列名使用 `snake_case`
- 索引命名为 `<table>_<column>_idx`
- 使用 `$type<T>()` 定义 JSON 字段 TypeScript 类型

```typescript
export const SessionTable = sqliteTable(
  "session",
  {
    id: text().primaryKey(),
    project_id: text()
      .notNull()
      .references(() => ProjectTable.id, { onDelete: "cascade" }),
    summary_diffs: text({ mode: "json" }).$type<Snapshot.FileDiff[]>(),
  },
  (table) => [index("session_project_idx").on(table.project_id)],
)
```

## 3. 编码风格

来自 `AGENTS.md` 的官方风格指南：

### 核心原则

```typescript
// 避免 any - 严格类型化
// Good
const value: string = "hello"

// 依赖类型推断 - 避免显式类型注解
// Good
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// 使用单字变量名
// Good
const foo = 1
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// 避免不必要的解构
// Good
obj.a
obj.b
// Bad
const { a, b } = obj

// 避免 else - 使用早期返回
// Good
function foo() {
  if (condition) return 1
  return 2
}
```

### 函数式方法

- 优先使用 `flatMap`、`filter`、`map` 等函数式数组方法
- 使用类型守卫维持类型推断

## 4. 泛型使用模式

### 泛型函数

```typescript
// 工具类型定义
export function fn<T extends z.ZodType, Result>(schema: T, cb: (input: z.infer<T>) => Result) {
  // ...
}
```

### Zod 类型推断

```typescript
// 从 Zod schema 推断类型
export type APIError = z.infer<typeof APIError.Schema>

// 泛型约束
export interface Info<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
  // ...
}
```

### as const 使用

```typescript
export const UserRole = ["admin", "member"] as const
```

## 5. 工具类型

项目使用 TypeScript 内置工具类型：

- `Omit`、`Pick`、`Partial`
- 条件类型和映射类型

## 6. 总结

| 特性         | 使用情况                   |
| ------------ | -------------------------- |
| 运行时验证   | Zod（核心）                |
| ORM          | Drizzle                    |
| 类型定义倾向 | type > interface           |
| 泛型使用     | 广泛使用                   |
| 编码风格     | 函数式、避免嵌套、依赖推断 |
| 包管理       | Bun                        |
| 前端框架     | SolidJS                    |
