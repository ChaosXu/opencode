# Console 包分析

## 1. 包概述

Console 是 OpenCode 的后端管理控制台，包含多个子包。

## 2. 子包结构

### 2.1 console/core

**版本**: 1.2.15
**类型**: 核心库 + 数据库

**技术栈**:

- Drizzle ORM (MySQL/PlanetScale)
- Hono

**依赖**:

- `hono` - Web 框架
- `drizzle-orm` - ORM

### 2.2 console/app

**版本**: 1.2.15
**类型**: Web 应用

**技术栈**:

- SolidJS
- SolidStart

### 2.3 console/function

**版本**: 1.2.15
**类型**: Serverless 函数

**技术栈**:

- Hono
- Cloudflare Workers

### 2.4 console/resource

**版本**: 1.2.15
**类型**: 资源定义

### 2.5 console/mail

**版本**: 1.2.15
**类型**: 邮件模板

## 3. 功能

- Zen API 路由
- Workspace 管理
- Stripe 集成 (支付)
- 邮件功能

## 4. 数据库

Console 使用 MySQL + PlanetScale：

- Schema 定义: `src/**/*.sql.ts`
- 迁移: Drizzle Kit

## 5. 总结

Console 是完整的后端管理系统，包含：

- Web 应用 (SolidStart)
- API 服务 (Hono)
- Serverless 函数
- 数据库 (MySQL)
- 支付集成 (Stripe)
