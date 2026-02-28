# Web 包分析

## 1. 包概述

**包名**: `@opencode-ai/web`
**版本**: 1.2.15
**类型**: 文档网站
**用途**: OpenCode 公开文档

## 2. 入口点

```json
{
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview"
  }
}
```

## 3. 技术栈

- **框架**: Astro 5.x
- **文档**: Starlight
- **前端**: SolidJS
- **部署**: Cloudflare

## 4. 功能

- 公开文档 (https://opencode.ai)
- Landing page
- 分享功能 (s/[id] 路由)
- 多语言支持 (i18n)

## 5. 依赖

### 生产依赖

- `astro` - 框架
- `@astrojs/starlight` - 文档
- `@astrojs/solid-js` - SolidJS 集成
- `@astrojs/cloudflare` - Cloudflare 适配

### 开发依赖

- `opencode` - 本地引用
- `typescript`

## 6. 静态结构

```
src/
├── content/
│   └── docs/           # 文档
├── assets/             # 静态资源
└── pages/             # 页面
```

## 7. 总结

Web 包提供：

- 公开文档网站
- Starlight 文档框架
- Cloudflare Pages 部署
