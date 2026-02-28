# Turbo 构建工具使用分析

## 1. 配置概览

项目使用 **Turbo 2.5.6** 作为构建工具，采用 monorepo 方式进行项目管理。

### 根目录 package.json workspaces 配置

```json
{
  "workspaces": {
    "packages": ["packages/*", "packages/console/*", "packages/sdk/js", "packages/slack"]
  }
}
```

## 2. Monorepo 结构

项目包含以下主要包：

| 包                    | 说明           |
| --------------------- | -------------- |
| `packages/opencode`   | CLI 核心工具   |
| `packages/app`        | 独立应用程序   |
| `packages/ui`         | 共享 UI 组件库 |
| `packages/web`        | 文档网站       |
| `packages/console`    | 后端管理控制台 |
| `packages/sdk`        | SDK 生成       |
| `packages/plugin`     | 插件系统       |
| `packages/desktop`    | 桌面应用       |
| `packages/enterprise` | 企业功能       |

## 3. 构建命令

项目使用 Bun 作为运行时，但通过 Turbo 进行任务编排：

```bash
# 开发命令
bun dev                    # 开发模式
bun dev <directory>        # 在指定目录运行

# 构建命令
bun run build              # 构建所有包
bun run --cwd <package> build  # 构建特定包

# 测试命令
bun test                   # 运行测试
```

## 4. 包管理

项目使用 **npm workspaces** 进行包管理，所有包都在 `packages/` 目录下组织。

### 依赖共享

- 内部包通过 `workspace:*` 协议引用
- 例如：`"@opencode-ai/sdk": "workspace:*"`

## 5. 最佳实践

1. **Monorepo 架构**：通过 workspaces 共享代码
2. **集中管理**：根目录统一管理依赖版本
3. **独立构建**：各包可以独立构建和测试
4. **Bun 优先**：使用 Bun 作为运行时和包管理器
