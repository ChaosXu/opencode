# Drizzle ORM 使用分析

## 1. 数据库配置

项目使用**两个不同的数据库后端**：

### opencode 包 (SQLite + Bun)

```typescript
// drizzle.config.ts
export default defineConfig({
  dialect: "sqlite",
  schema: "./src/**/*.sql.ts",
  out: "./migration",
  dbCredentials: { url: "/home/thdxr/.local/share/opencode/opencode.db" },
})
```

使用 `drizzle-orm/bun-sqlite` 驱动，本地存储在 `~/.local/share/opencode/opencode.db`

### console/core 包 (MySQL + PlanetScale)

```typescript
// drizzle.config.ts
export default defineConfig({
  dialect: "mysql",
  schema: ["./src/**/*.sql.ts"],
  out: "./migrations/",
  dbCredentials: { host, user, password, port, ssl: { rejectUnauthorized: false } },
})
```

使用 `drizzle-orm/planetscale-serverless` 连接 PlanetScale MySQL

## 2. 表定义模式

### SQLite 表定义示例

```typescript
// project.sql.ts
import { sqliteTable, text, integer } from "drizzle-orm/sqlite-core"
import { Timestamps } from "@/storage/schema.sql"

export const ProjectTable = sqliteTable("project", {
  id: text().primaryKey(),
  worktree: text().notNull(),
  vcs: text(),
  name: text(),
  icon_url: text(),
  icon_color: text(),
  ...Timestamps,
  time_initialized: integer(),
  sandboxes: text({ mode: "json" }).notNull().$type<string[]>(),
  commands: text({ mode: "json" }).$type<{ start?: string }>(),
})
```

### MySQL 表定义示例

```typescript
// user.sql.ts
import { mysqlTable, varchar, int, mysqlEnum, index, bigint } from "drizzle-orm/mysql-core"

export const UserTable = mysqlTable(
  "user",
  {
    ...workspaceColumns,
    ...timestamps,
    accountID: ulid("account_id"),
    email: varchar("email", { length: 255 }),
    name: varchar("name", { length: 255 }).notNull(),
    role: mysqlEnum("role", ["admin", "member"]).notNull(),
    monthlyLimit: int("monthly_limit"),
    monthlyUsage: bigint("monthly_usage", { mode: "number" }),
  },
  (table) => [
    uniqueIndex("user_account_id").on(table.workspaceID, table.accountID),
    index("global_account_id").on(table.accountID),
  ],
)
```

### 关键模式特点

- **命名约定**: 表名和列名使用 snake_case
- **索引命名**: `<table>_<column>_idx` (如 `session_project_idx`)
- **JSON 字段**: 使用 `$type<T>()` 定义 TypeScript 类型
- **外键**: 使用 `references(() => Table.column, { onDelete: "cascade" })`
- **复合主键**: 通过 `primaryKey({ columns: [...] })` 定义

## 3. 共享模式定义

### 公共时间戳

```typescript
// schema.sql.ts
export const Timestamps = {
  time_created: integer()
    .notNull()
    .$default(() => Date.now()),
  time_updated: integer()
    .notNull()
    .$onUpdate(() => Date.now()),
}
```

### MySQL 共享类型

```typescript
// drizzle/types.ts
export const ulid = (name: string) => varchar(name, { length: 30 })
export const workspaceColumns = {
  get id() {
    return ulid("id").notNull()
  },
  get workspaceID() {
    return ulid("workspace_id").notNull()
  },
}
export const timestamps = {
  timeCreated: utc("time_created").notNull().defaultNow(),
  timeUpdated: utc("time_updated")
    .notNull()
    .default(sql`CURRENT_TIMESTAMP(3) ON UPDATE CURRENT_TIMESTAMP(3)`),
  timeDeleted: utc("time_deleted"),
}
```

## 4. 查询构建使用方式

### 基础查询操作

```typescript
import { Database, eq, and, or, desc, like, inArray, isNull, gte, lt } from "../storage/db"

// SELECT - 单条查询
const row = Database.use((db) => db.select().from(ProjectTable).where(eq(ProjectTable.id, id)).get())

// SELECT - 多条查询
const sessions = Database.use((db) =>
  db.select().from(SessionTable).where(eq(SessionTable.project_id, projectID)).all(),
)

// INSERT
Database.use((db) => db.insert(ProjectTable).values(insert).run())

// UPDATE + RETURNING
const result = Database.use((db) =>
  db
    .update(ProjectTable)
    .set({ name: input.name, time_updated: Date.now() })
    .where(eq(ProjectTable.id, input.projectID))
    .returning()
    .get(),
)

// UPSERT (onConflictDoUpdate)
Database.use((db) =>
  db.insert(ProjectTable).values(insert).onConflictDoUpdate({ target: ProjectTable.id, set: updateSet }).run(),
)

// DELETE
db.delete(SessionTable).where(eq(SessionTable.id, sessionID)).run()
```

### 复杂查询条件

```typescript
// AND 条件
const conditions = [eq(SessionTable.project_id, project.id)]
if (input.directory) conditions.push(eq(SessionTable.directory, input.directory))
db.select()
  .from(SessionTable)
  .where(and(...conditions))

// OR 条件
or(
  and(eq(AuthTable.provider, response.provider), eq(AuthTable.subject, subject)),
  and(eq(AuthTable.provider, "email"), eq(AuthTable.subject, email)),
)
  // 排序 + 分页
  .query.orderBy(desc(SessionTable.time_updated), desc(SessionTable.id))
  .limit(limit)
  .all()

  // JOIN 查询
  .innerJoin(WorkspaceTable, eq(WorkspaceTable.id, KeyTable.workspaceID))
  .leftJoin(ModelTable, and(eq(ModelTable.workspaceID, KeyTable.workspaceID), isNull(ModelTable.timeDeleted)))
```

## 5. 数据库交互模式

### 数据库连接封装

```typescript
// db.ts
export namespace Database {
  // 懒加载客户端
  export const Client = lazy(() => {
    const sqlite = new BunDatabase(path, { create: true })
    sqlite.run("PRAGMA journal_mode = WAL")
    sqlite.run("PRAGMA foreign_keys = ON")
    const db = drizzle({ client: sqlite, schema })

    // 应用迁移
    migrate(db, entries)
    return db
  })

  // 事务支持
  export function use<T>(callback: (trx: TxOrDb) => T): T {
    try {
      return callback(ctx.use().tx)
    } catch {
      // 自动事务
      return ctx.provide({ tx: Client() }, () => callback(Client()))
    }
  }

  // 显式事务
  export function transaction<T>(callback: (tx: TxOrDb) => T): T {
    return Client().transaction((tx) => callback(tx))
  }
}
```

## 6. 迁移策略

### Drizzle Kit 迁移 (console 包)

```bash
bun run db generate --name <slug>
# 生成 migration/<timestamp>_<slug>/migration.sql
```

### 内嵌迁移 (opencode 包)

- 迁移脚本通过 `OPENCODE_MIGRATIONS` 内置
- 开发时从 `./migration` 目录读取 SQL 文件
- 迁移按时间戳排序后自动应用

## 7. 表结构总览

| 包           | 表文件                        | 数据库                                   |
| ------------ | ----------------------------- | ---------------------------------------- |
| opencode     | `src/storage/schema.sql.ts`   | 公共 Timestamps                          |
| opencode     | `src/project/project.sql.ts`  | Project 表                               |
| opencode     | `src/session/session.sql.ts`  | Session, Message, Part, Todo, Permission |
| opencode     | `src/share/share.sql.ts`      | Share 表                                 |
| opencode     | `src/control/control.sql.ts`  | Control 表                               |
| console/core | `src/schema/user.sql.ts`      | User 表                                  |
| console/core | `src/schema/workspace.sql.ts` | Workspace 表                             |
| console/core | `src/schema/key.sql.ts`       | Key 表                                   |
| console/core | `src/schema/billing.sql.ts`   | Billing, Payment, Subscription           |
| console/core | `src/schema/model.sql.ts`     | Model 表                                 |
| console/core | `src/schema/provider.sql.ts`  | Provider 表                              |
