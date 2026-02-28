# Enterprise 包分析

## 1. 包概述

**包名**: `@opencode-ai/enterprise`
**版本**: 1.2.15
**类型**: 企业功能
**用途**: 企业级特性和后端服务

## 2. 入口点

```json
{
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "build:cloudflare": "OPENCODE_DEPLOYMENT_TARGET=cloudflare vite build"
  }
}
```

## 3. 技术栈

- **框架**: @solidjs/start (SolidStart)
- **后端**: Hono + nitro
- **样式**: Tailwind CSS
- **部署**: Cloudflare / Node.js

## 4. 模块

### 4.1 前端

- SolidStart 应用
- 企业控制台

### 4.2 后端

- Hono API
- OpenAPI 集成
- AWS 集成

## 5. 依赖

### 生产依赖

- `@opencode-ai/util` - 工具库
- `@opencode-ai/ui` - UI
- `@solidjs/router` - 路由
- `@solidjs/start` - SolidStart
- `hono` - Web 框架
- `hono-openapi` - OpenAPI
- `aws4fetch` - AWS
- `nitro` - 部署

### 开发依赖

- `@cloudflare/workers-types` - Cloudflare
- `tailwindcss` - 样式
- `vite` - 构建

## 6. 运行环境

- Node.js >= 22

## 7. 静态结构

```
src/
├── routes/             # 路由
├── components/         # 组件
└── ...
```

## 8. 总结

Enterprise 包提供：

- 企业级功能
- SolidStart 架构
- 多部署目标支持
- OpenAPI 集成
