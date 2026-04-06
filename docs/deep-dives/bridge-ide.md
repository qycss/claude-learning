# Bridge & IDE Integration

## Overview

The Bridge system provides bidirectional communication between Claude Code and IDE extensions (VS Code, JetBrains) and the claude.ai web interface. It has two protocol generations and an "env-less" variant.

## File Inventory

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/bridge/types.ts` | 263 | Type definitions (BridgeConfig, SessionHandle, SpawnMode) |
| `src/bridge/bridgeMain.ts` | 700+ | Standalone bridge (multi-session mode) |
| `src/bridge/replBridge.ts` | 800+ | REPL-embedded bridge |
| `src/bridge/remoteBridgeCore.ts` | 1009 | Env-less V2 bridge |
| `src/bridge/replBridgeTransport.ts` | 371 | Transport abstraction (V1/V2 adapters) |
| `src/bridge/jwtUtils.ts` | 257 | JWT decode + refresh scheduler |
| `src/bridge/flushGate.ts` | — | Queue during history flush |

## Protocol Comparison

| Aspect | V1 | V2 |
|--------|----|----|
| Read | WebSocket (HybridTransport) | SSE (断点续传 via `from_sequence_num`) |
| Write | HTTP POST to Session-Ingress | CCRClient (SerialBatchEventUploader) |
| Sequence | None (returns 0) | Full sequence tracking |
| Delivery | None | `received` + `processed` ACK |
| State | None | `reportState()`: idle/running/requires_action |
| Metadata | None | `reportMetadata()` |

## Env-less V2 — Three-Step Connection

The simplest path, bypassing the Environments API lifecycle:

```
1. POST /v1/code/sessions (OAuth) → session.id
2. POST /v1/code/sessions/{id}/bridge (OAuth) → worker_jwt + epoch
3. createV2ReplTransport(worker_jwt, epoch) → SSE + CCRClient
```

Each `/bridge` call increments epoch — it **is** the registration. No register/poll/ack/deregister lifecycle.

## Echo Dedup — Three-Level

Written messages echo back through SSE. Three dedup layers:

| Layer | Implementation | Purpose |
|-------|---------------|---------|
| `recentPostedUUIDs` | BoundedUUIDSet (2000 cap ring buffer) | Real-time messages |
| `initialMessageUUIDs` | Unbounded Set | Initial history (may exceed 2000) |
| `recentInboundUUIDs` | BoundedUUIDSet | Prevent duplicate receives |

## FlushGate

Ensures ordered transmission of `[history..., realtime...]`:

```
flushGate.start()  → begin queuing
  writeMessages()  → flushGate.enqueue(msgs) → buffer
flushGate.end()    → return buffered messages, send immediately
```

## JWT Refresh

`jwtUtils.ts:72-256` creates per-session proactive refresh timers:
- Schedule at `expiresIn - refreshBuffer` (minimum 30s)
- Generation counter prevents stale callbacks after cancel/reschedule
- Max `MAX_REFRESH_FAILURES = 3` consecutive failures before giving up
- Successful refresh schedules follow-up (fallback 30 min)

## Crash Recovery

**Standalone bridge** (`bridgeMain.ts:141+`):
- `runBridgeLoop` maintains `activeSessions` Map with timeout watchdog
- 401 JWT expiry → `reconnectSession` re-queues session
- Sleep/wake detection: if poll interval exceeds `connCapMs * 2`, assume system woke up, reset backoff

**REPL bridge** (`remoteBridgeCore.ts:530-590`):
- `recoverFromAuthFailure()`: refresh OAuth → get new /bridge credentials → `rebuildTransport()` (new epoch + JWT + SSE seq-num continuation)
- Proactive refresh and 401 recovery are mutually exclusive via `authRecoveryInFlight` flag

## Unique Findings

1. **Processed immediate ACK** (`replBridgeTransport.ts:249-252`): Sends both `received` and `processed` simultaneously to prevent daemon restart accumulating 21→25 phantom prompts.
2. **Per-instance auth closure** (`remoteBridgeCore.ts:233-234`): Worker JWT passed via closure, not `process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN`, because `mcp/client.ts` unconditionally reads that env var — leaking JWT to user-configured MCP servers.
3. **Capacity Wake** (`bridgeMain.ts:194`): When session slots are full, poll loop sleeps until `capacityWake` signal fires (session completion), avoiding pointless API polling.
4. **SpawnMode** (`types.ts:69`): `single-session | worktree | same-dir`. Worktree mode creates git worktree isolation per session.
