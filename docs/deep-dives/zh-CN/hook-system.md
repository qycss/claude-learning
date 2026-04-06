# Hook 系统

## 概述

Hook 系统是 Claude Code 的生命周期扩展机制，允许用户在 **27 个事件点**上执行自定义逻辑。Hook 在 settings.json 中配置，支持 matcher 模式匹配，并使用**退出码协议**（0=成功，2=阻止，其他=非阻塞错误）。

## 文件清单

| 文件路径 | 行数 | 职责 |
|-----------|-------|------|
| `src/utils/hooks.ts` | 1800+ | Hook 执行引擎核心 |
| `src/types/hooks.ts` | 291 | Hook 类型定义 + Zod Schema |
| `src/utils/hooks/hooksConfigManager.ts` | 401 | 27 种事件类型的元数据与分组 |
| `src/utils/hooks/hookEvents.ts` | 193 | Hook 事件广播系统 |
| `src/utils/hooks/hooksSettings.ts` | 100+ | Hook 配置读取/展示/比较 |
| `src/utils/hooks/AsyncHookRegistry.ts` | 80+ | 异步 Hook 注册与超时管理 |
| `src/utils/plugins/loadPluginHooks.ts` | 288 | Plugin Hook 加载与热重载 |

## 27 种 Hook 事件类型

### Tool 生命周期
- `PreToolUse` -- Tool 执行前（可阻止）
- `PostToolUse` -- Tool 成功执行后
- `PostToolUseFailure` -- Tool 执行失败后
- `PermissionDenied` -- 用户拒绝授权时
- `PermissionRequest` -- 请求授权时

### 会话生命周期
- `SessionStart`、`SessionEnd`、`Stop`、`StopFailure`

### 用户交互
- `UserPromptSubmit`、`Notification`

### 多 Agent
- `SubagentStart`、`SubagentStop`、`TeammateIdle`
- `TaskCreated`、`TaskCompleted`

### 配置/文件
- `ConfigChange`、`InstructionsLoaded`、`CwdChanged`、`FileChanged`

### Compact
- `PreCompact`、`PostCompact`

### Worktree
- `WorktreeCreate`、`WorktreeRemove`

### MCP Elicitation
- `Elicitation`、`ElicitationResult`

### 基础设施
- `Setup`

## 四种执行类型

| 类型 | 机制 |
|------|------|
| `command` | Shell 命令（自动选择 bash/powershell） |
| `prompt` | 通过 `execPromptHook` 执行 LLM Prompt |
| `agent` | 通过 `execAgentHook` 执行子 Agent |
| `http` | 通过 `execHttpHook` 调用 HTTP 端点 |

## 权限覆盖

`PreToolUse` Hook（`hooks.ts:489-610`）可返回 `permissionDecision: allow|deny|ask`，**直接覆盖**权限系统的判定。

`PermissionRequest` Hook 可返回完整决策（`types/hooks.ts:122-134`）：
- `updatedInput` -- 在执行前修改 Tool 输入
- `updatedPermissions` -- 写入新的权限规则

## 信任安全层

`shouldSkipHookDueToTrust()`（`hooks.ts:267-296`）：
- 所有 Hook 都需要工作区信任对话框的确认
- 交互模式下：不受信任的工作区 Hook 会被跳过
- 历史修复：SessionEnd Hook 此前即使用户拒绝信任也会触发
- SDK/Headless 模式下：信任检查隐式通过

## 异步 Hook 协议

Hook 可返回 `{"async": true, "asyncTimeout": 15000}`（`AsyncHookRegistry.ts:30-80`）以变为后台任务：
- 在 `pendingHooks` Map 中注册并进行超时管理
- 结果通过 `emitHookResponse` 广播
- 在 `asyncRewake` 模式下，退出码为 2 的结果会通过 `enqueuePendingNotification` 唤醒模型

## SessionEnd 超时

SessionEnd Hook 的默认超时时间为 **1.5 秒**（其他 Hook 为 10 分钟），可通过 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` 覆盖（`hooks.ts:166-181`）。

## 值得关注的发现

- **`if` 条件执行**：相同命令搭配不同 `if` 条件会被视为不同的 Hook
- **Plugin Hook 原子替换**（`loadPluginHooks.ts:137-156`）：修复 gh-29767 -- `clearRegisteredPluginHooks()` + `registerHookCallbacks()` 作为原子操作对
- **热重载**：`setupPluginHookHotReload()` 比较 4 个字段的 JSON 快照，仅在实际变更时才重新加载
- **Plugin 数据隔离**：Hook 执行时提供 `getPluginDataDir(pluginId)` 和 `CLAUDE_PLUGIN_OPTION_*` 环境变量
