# Plugin 系统

## 概述

Claude Code 的 Plugin 系统是一个基于 Marketplace 的分发架构，内部代号为 **Tengu**。它采用 **settings-first** 设计 -- 先在 settings 中声明意图，随后再进行实体化。每个 Plugin 是一个包含 `.claude-plugin/plugin.json` 清单文件的目录，提供 **7 种扩展类型**：MCP 服务器、斜杠命令、Agent、Skill、Hook、LSP 服务器和输出样式。

## 文件清单

| 文件路径 | 行数 | 职责 |
|-----------|-------|------|
| `src/services/plugins/PluginInstallationManager.ts` | 185 | 后台 Marketplace 安装，与 AppState 对账 |
| `src/services/plugins/pluginOperations.ts` | 1089 | 核心增删改查：安装/卸载/启用/禁用/更新 |
| `src/services/plugins/pluginCliCommands.ts` | 345 | CLI 封装，遥测 + process.exit 管理 |
| `src/utils/plugins/schemas.ts` | 1682 | 全部 Zod Schema（清单/Marketplace/安装/来源） |
| `src/utils/plugins/pluginLoader.ts` | 2000+ | 发现/加载/校验/缓存引擎 |
| `src/utils/plugins/validatePlugin.ts` | 904 | 校验引擎（清单/Marketplace/组件） |
| `src/utils/plugins/pluginBlocklist.ts` | 128 | 下架检测与自动卸载 |
| `src/utils/plugins/mcpPluginIntegration.ts` | 400+ | 从 Plugin 加载 MCP 服务器（MCPB/DXT 格式） |
| `src/utils/plugins/loadPluginHooks.ts` | 288 | Plugin Hook 加载与热重载 |

## 三阶段加载流程

### 阶段一 -- 声明（`pluginOperations.ts:321-418`）

`installPluginOp()` 搜索已实体化的 Marketplace，写入 settings（`enabledPlugins`），缓存版本信息。这是原子化的意图声明 -- 执行可以延迟进行。

### 阶段二 -- 实体化（`PluginInstallationManager.ts:60-184`）

`performBackgroundPluginInstallations()` 在后台调用 `reconcileMarketplaces()`，通过 `diffMarketplaces()` 计算缺失/源变更状态，驱动 git clone/npm install 操作。

### 阶段三 -- 加载（`pluginLoader.ts:0-150`）

`loadAllPlugins()`（已 memoize）读取缓存/种子目录，解析 `plugin.json` 清单，构建 `LoadedPlugin` 对象。

## Plugin 来源类型

7 种来源类型（`schemas.ts:1062-1161`）：

| 类型 | 示例 |
|------|------|
| 相对路径 | `./my-plugin` |
| npm | `@scope/plugin`（支持自定义 registry） |
| pip | Python 包 |
| URL | Git clone 地址 |
| GitHub | `owner/repo` 简写 |
| git-subdir | Monorepo 稀疏 clone |
| 本地目录 | 绝对路径 |

## 反冒名安全机制

三层防线（`schemas.ts:9-101`）：

1. **保留名称白名单** -- `ALLOWED_OFFICIAL_MARKETPLACE_NAMES`
2. **正则拦截** -- `BLOCKED_OFFICIAL_NAME_PATTERN` 检测 "official" + "anthropic/claude" 的组合
3. **Unicode 同形字检测** -- `NON_ASCII_PATTERN`
4. `validateOfficialNameSource()` 要求保留名称必须来自 `anthropics` GitHub 组织

## 自动下架

当 `forceRemoveDeletedPlugins: true` 时（`pluginBlocklist.ts:64-127`），系统会自动检测已安装但不再出现在 Marketplace 中的 Plugin，并调用 `uninstallPluginOp()` 进行卸载。

依赖安全性：`uninstallPluginOp()` 会调用 `findReverseDependents()`（第 546-547 行），但**不会阻塞** -- 仅发出警告。这避免了墓碑问题，即已下架的 Plugin 阻止依赖图的拆除。

## 作用域隔离

四种作用域（`schemas.ts:1494-1509`）：
```
managed > user > project > local
```

- `project` 作用域写入 `.claude/settings.json`（团队共享）
- `local` 作用域写入 `.claude/settings.local.json`（个人覆盖）
- `disablePlugin --scope local` 提供了一个不修改团队配置的逃生通道

## PII 隔离

Plugin 名称通过 `_PROTO_plugin_name` 路由到 BigQuery 中带 PII 标记的列（`pluginCliCommands.ts:72-73`）。标准分析元数据不会保留原始 Plugin 名称。
