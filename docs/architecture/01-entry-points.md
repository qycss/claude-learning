# Entry Points & Startup

## Four Entry Points

Claude Code has four distinct entry points, each serving a different integration surface:

### 1. CLI Entry (`src/main.tsx`)

The primary entry point. Three-stage parallel prefetch architecture:

```
Stage 1 (immediate, before heavy imports):
  ├── MDM settings prefetch (rawRead.ts — zero heavy imports)
  ├── Keychain prefetch (keychainPrefetch.ts — isolated imports)
  └── API preconnect (TCP connection warming)

Stage 2 (Commander.js definition):
  ├── Parse CLI args
  ├── Register subcommands
  └── Validate flags

Stage 3 (Ink renderer):
  ├── Initialize React + Ink
  ├── Mount App component
  └── Start REPL loop
```

Key insight: `rawRead.ts` and `keychainPrefetch.ts` are deliberately designed with zero heavy imports. Bun's `__esm` wrapper evaluates the entire module graph on first symbol access — importing execa's `human-signals/cross-spawn` chain costs ~58ms synchronous initialization, which would negate prefetch parallelism.

### 2. Leader/Coordinator CLI (`src/entrypoints/cli.tsx`)

Orchestration layer that sits above `main.tsx`:
- Manages leader election for daemon mode
- Coordinates multi-session bridge connections
- Handles CLI flag normalization before main entry

### 3. MCP Server (`src/entrypoints/mcp.ts`)

Runs Claude Code as an MCP (Model Context Protocol) server:
- Exposes tools as MCP tool definitions
- Accepts MCP client connections
- Bridges MCP protocol to internal tool system

### 4. Agent SDK (`src/entrypoints/sdk/`)

Programmatic integration for the Agent SDK:
- Headless mode (no terminal UI)
- Direct message passing API
- Used by external applications embedding Claude Code

## Startup Sequence

```
1. main.tsx evaluates
   ├── Trigger parallel prefetch (MDM, keychain, preconnect)
   ├── Import Commander.js (lightweight)
   └── Define CLI program

2. Commander.js parses args
   ├── Determine mode (interactive / MCP / bridge / resume)
   └── Extract flags (model, permission mode, etc.)

3. setup.ts runs
   ├── Initialize session storage
   ├── Load settings (7-source merge)
   ├── Resolve API credentials
   ├── Initialize telemetry
   └── Build system prompt (context.ts)

4. Ink renderer mounts
   ├── App.tsx (providers: FpsMetrics, Stats, AppState)
   └── REPL.tsx (5005 lines — main interaction loop)

5. First query loop
   ├── QueryEngine.ts (session-scoped)
   └── query.ts (turn-scoped)
```

## Parallel Prefetch Detail

The prefetch architecture avoids a sequential startup waterfall:

| Component | Strategy | Why |
|-----------|----------|-----|
| MDM settings | `rawRead.ts` subprocess | Zero imports, can run during module evaluation |
| Keychain | `keychainPrefetch.ts` async | Isolated from execa to avoid 58ms sync init |
| API preconnect | TCP connect | Warm connection pool before first query |
| GrowthBook | Disk cache + async refresh | Offline-safe with stale-while-revalidate |

## Session Initialization (`src/setup.ts`)

The `setup()` function orchestrates:
1. Session ID generation (UUID v4)
2. Settings resolution (7-source pipeline)
3. API client initialization (with retry engine)
4. System prompt assembly (static + dynamic sections)
5. Tool registration (40 tools with Zod schemas)
6. Hook system activation (27 event types)
7. Telemetry initialization (4 channels)
