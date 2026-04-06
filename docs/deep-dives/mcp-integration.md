# MCP Integration

## Overview

Claude Code implements comprehensive MCP (Model Context Protocol) integration with **8 transport types** and a **7-scope configuration** system with enterprise exclusivity.

## Architecture

```
User/Project Config → MCP Config Resolution (7 scopes) → Transport Selection → Connection
                                                              ↓
                                                    Tool/Resource/Prompt Registration
                                                              ↓
                                                    MCPTool proxies calls to servers
```

## 8 Transport Types

| Transport | Use Case |
|-----------|----------|
| `stdio` | Local process (most common) |
| `sse` | Server-Sent Events (legacy) |
| `streamable-http` | Modern HTTP streaming |
| `node` | Node.js child process |
| `npx` | npm package execution |
| `uvx` | Python uv tool execution |
| `docker` | Container-based servers |
| `url` | Direct URL connection |

## 7-Scope Config (`src/services/mcp/config.ts`)

Configuration sources with precedence:

```
enterprise (highest)
  → managed
    → user
      → project
        → local
          → plugin
            → flag (lowest)
```

Enterprise scope supports **exclusivity** — when set, lower scopes are ignored for the specified servers. This ensures corporate-mandated MCP servers cannot be overridden by individual users.

## Authentication

### OAuth 2.0
- Full OAuth flow for MCP servers requiring authentication
- Token storage in keychain (shared with main Claude Code OAuth)

### XAA (Cross-Agent Authentication)
- Server-to-server authentication protocol
- Used when MCP servers need to authenticate with each other

## Connection Management

- Lazy connection — servers connect on first tool use
- Connection pooling for frequently-used servers
- Automatic reconnection with backoff
- Health check heartbeats

## Tool Registration

MCP servers expose tools that get dynamically registered:
```
Server connects → Server sends tool list → Tools registered as MCPTool instances
                                                     ↓
                                         Available in LLM tool_use
```

Tool names are prefixed with `mcp__` to distinguish from built-in tools.

## Security

- MCP tool names sanitized in analytics (`mcp__*` → `mcp_tool`) to prevent PII leakage
- Worker JWT passed via closure, not `process.env`, to prevent MCP servers from reading session tokens
- Enterprise exclusivity prevents shadow MCP server configurations
