# Function 包分析

## 1. 包概述

**包名**: `@opencode-ai/function`
**版本**: 1.2.15
**类型**: Cloudflare Worker / 云函数
**用途**: 后端服务函数

## 2. 入口点

通过 `package.json` 的 `exports` 字段定义（私有包）

## 3. 技术栈

- **运行时**: Cloudflare Workers
- **框架**: Hono
- **认证**: jose (JWT)
- **HTTP**: @octokit/rest

## 4. 依赖

### 生产依赖

- `@octokit/auth-app` - GitHub App 认证
- `@octokit/rest` - GitHub API
- `hono` - Web 框架
- `jose` - JWT 处理

### 开发依赖

- `@cloudflare/workers-types` - Cloudflare 类型
- `@tsconfig/node22` - TS 配置
- `typescript`

## 5. 用途

该包用于：

- 云端 API 函数
- GitHub 集成
- 认证服务

## 6. 总结

Function 包是云函数包，提供：

- Serverless API
- GitHub 集成
- JWT 认证
