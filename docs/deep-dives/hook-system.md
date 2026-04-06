# Hook System

## Overview

The Hook system is Claude Code's lifecycle extension mechanism, allowing users to execute custom logic at **27 event points**. Hooks are configured in settings.json, support matcher pattern matching, and use an **exit code protocol** (0=success, 2=block, other=non-blocking error).

## File Inventory

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/utils/hooks.ts` | 1800+ | Hook execution engine core |
| `src/types/hooks.ts` | 291 | Hook type definitions + Zod schemas |
| `src/utils/hooks/hooksConfigManager.ts` | 401 | 27 event type metadata & grouping |
| `src/utils/hooks/hookEvents.ts` | 193 | Hook event broadcasting system |
| `src/utils/hooks/hooksSettings.ts` | 100+ | Hook config read/display/comparison |
| `src/utils/hooks/AsyncHookRegistry.ts` | 80+ | Async hook registration & timeout |
| `src/utils/plugins/loadPluginHooks.ts` | 288 | Plugin hooks loading & hot reload |

## 27 Hook Event Types

### Tool Lifecycle
- `PreToolUse` — Before tool execution (can block)
- `PostToolUse` — After successful tool execution
- `PostToolUseFailure` — After tool execution failure
- `PermissionDenied` — When user denies permission
- `PermissionRequest` — When permission is requested

### Session Lifecycle
- `SessionStart`, `SessionEnd`, `Stop`, `StopFailure`

### User Interaction
- `UserPromptSubmit`, `Notification`

### Multi-Agent
- `SubagentStart`, `SubagentStop`, `TeammateIdle`
- `TaskCreated`, `TaskCompleted`

### Config/File
- `ConfigChange`, `InstructionsLoaded`, `CwdChanged`, `FileChanged`

### Compact
- `PreCompact`, `PostCompact`

### Worktree
- `WorktreeCreate`, `WorktreeRemove`

### MCP Elicitation
- `Elicitation`, `ElicitationResult`

### Infra
- `Setup`

## Four Execution Types

| Type | Mechanism |
|------|-----------|
| `command` | Shell command (bash/powershell selection) |
| `prompt` | LLM prompt via `execPromptHook` |
| `agent` | Sub-agent via `execAgentHook` |
| `http` | HTTP endpoint via `execHttpHook` |

## Permission Override

`PreToolUse` hooks (`hooks.ts:489-610`) can return `permissionDecision: allow|deny|ask`, **directly overriding** the permission system.

`PermissionRequest` hooks can return a complete decision (`types/hooks.ts:122-134`):
- `updatedInput` — modify tool input before execution
- `updatedPermissions` — write new permission rules

## Trust Security Layer

`shouldSkipHookDueToTrust()` (`hooks.ts:267-296`):
- All hooks require workspace trust dialog confirmation
- Interactive mode: untrusted workspace hooks are skipped
- Historical fix: SessionEnd hooks previously fired even when user rejected trust
- SDK/headless mode: trust check implicitly passes

## Async Hook Protocol

Hooks can return `{"async": true, "asyncTimeout": 15000}` (`AsyncHookRegistry.ts:30-80`) to become background tasks:
- Registered in `pendingHooks` Map with timeout management
- Results broadcast via `emitHookResponse`
- In `asyncRewake` mode, exit code 2 results wake the model via `enqueuePendingNotification`

## SessionEnd Timeout

SessionEnd hooks have a **1.5 second** default timeout (vs 10 minutes for other hooks), overridable via `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` (`hooks.ts:166-181`).

## Unique Findings

- **`if` conditional execution**: Same command with different `if` conditions is treated as different hooks
- **Atomic plugin hook replacement** (`loadPluginHooks.ts:137-156`): Fix for gh-29767 — `clearRegisteredPluginHooks()` + `registerHookCallbacks()` as atomic pair
- **Hot reload**: `setupPluginHookHotReload()` compares JSON snapshots of 4 fields, only reloads on actual changes
- **Plugin data isolation**: Hook execution provides `getPluginDataDir(pluginId)` and `CLAUDE_PLUGIN_OPTION_*` env vars
