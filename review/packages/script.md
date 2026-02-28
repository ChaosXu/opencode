# Script 包分析

## 1. 包概述

**包名**: `@opencode-ai/script`
**版本**: 1.2.15
**类型**: 构建脚本库
**用途**: 供其他包使用的脚本工具

## 2. 入口点

```json
{
  "exports": {
    ".": "./src/index.ts"
  }
}
```

## 3. 导出内容

```typescript
// src/index.ts
// 脚本工具函数
```

## 4. 依赖

### 开发依赖

- `@types/bun` - Bun 类型

## 5. 总结

Script 包是轻量级的脚本工具库，用于：

- 构建脚本
- 发布脚本
- 代码生成
