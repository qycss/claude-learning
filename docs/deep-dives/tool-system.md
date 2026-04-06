# Tool System

## Overview

Claude Code's tool system is built around a `buildTool()` factory that creates standardized tool objects from Zod schemas. Each of the ~40 tools lives in its own subdirectory under `src/tools/` with a consistent structure.

## Tool Directory Structure

```
src/tools/
├── BashTool/
│   ├── BashTool.ts        # Implementation
│   ├── UI.tsx             # Ink-based progress/output rendering
│   ├── prompt.ts          # Tool description for LLM
│   └── bashSecurity.ts    # 2592-line security validator
├── FileReadTool/
│   ├── FileReadTool.ts
│   ├── UI.tsx
│   └── prompt.ts
├── AgentTool/
│   ├── AgentTool.ts
│   ├── UI.tsx
│   ├── prompt.ts
│   └── forkSubagent.ts    # Fork cache sharing
├── ... (~40 total)
```

## buildTool() Factory (`src/Tool.ts`)

A remarkably concise 4-line factory:

```typescript
export function buildTool<T extends z.ZodType>(config: ToolConfig<T>): Tool<T> {
  return { ...config }
}
```

Each tool config includes:
- `name` — Tool identifier
- `description` — LLM-facing description (from `prompt.ts`)
- `inputSchema` — Zod schema for input validation
- `call(input, context)` — Execution function
- `isAllowed?(input, context)` — Per-tool permission check
- `beforeHook?(input, context)` — Pre-execution hook
- `afterHook?(result, context)` — Post-execution hook
- `UI` — React/Ink component for rendering

## Key Tools

### BashTool
- Shell command execution with security validation
- `bashSecurity.ts` — 2,592 lines of command analysis
- Pattern matching against known dangerous commands
- See [Bash Security](../security/bash-security.md)

### FileReadTool / FileWriteTool / FileEditTool
- File system operations with path safety
- CWD containment check
- FileEdit uses exact string matching for replacements

### AgentTool
- Sub-agent spawning for complex tasks
- `forkSubagent.ts` — Fork with LLM cache sharing (Copy-on-Write)
- See [Multi-Agent](multi-agent.md)

### GrepTool
- Wrapper around ripgrep (`rg`)
- Supports regex, glob filtering, context lines

### GlobTool
- File pattern matching
- Returns results sorted by modification time

### MCPTool
- Proxies tool calls to MCP servers
- Dynamic tool registration from MCP server capabilities

### LSPTool
- Language Server Protocol integration
- Code intelligence (hover, definitions, references)

## StreamingToolExecutor

`src/services/tools/StreamingToolExecutor.ts` manages concurrent tool execution during model streaming:

```
Read tools (Grep, Glob, FileRead):     Execute concurrently
Write tools (FileWrite, FileEdit, Bash): Serialize execution
```

Read-write lock semantics prevent concurrent file modifications while allowing parallel reads.

## Permission Integration

Every tool invocation passes through the permission system:
1. Check against permission rules (allow/deny/ask)
2. Check per-tool `isAllowed()` method
3. In auto mode, send to YOLO classifier
4. In default mode, prompt user

Tools in `SAFE_YOLO_ALLOWLISTED_TOOLS` (25+ tools) skip the classifier in auto mode.

## Tool Registration

Tools are registered during `setup.ts` initialization:
- All 40 tools are registered with their Zod schemas
- MCP tools are dynamically added from connected servers
- Plugin tools are loaded from enabled plugins
- Tool availability is checked each turn (some tools are context-dependent)
