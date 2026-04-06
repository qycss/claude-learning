# Session Persistence

## Overview

Claude Code's session persistence is built on **JSONL append-only logs** with **parentUuid chain** structure. Each session maps to `~/.claude/projects/{sanitized_project_path}/{sessionId}.jsonl`. Messages form a linked list via UUID parent pointers, supporting branches (sidechains for sub-agents) and merges.

## File Inventory

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/utils/sessionStorage.ts` | 5000+ | Core persistence engine — read/write/chain rebuild |
| `src/utils/sessionStoragePortable.ts` | 280 | Portable layer (shared with VS Code extension) |
| `src/utils/conversationRecovery.ts` | 600 | Deserialization + session recovery |
| `src/utils/sessionRestore.ts` | 550 | State restoration orchestration |
| `src/utils/messages.ts` | 2200+ | Message normalization (`normalizeMessagesForAPI`) |
| `src/types/logs.ts` | 331 | 21 Entry types (discriminated union) |

## Message Format

Each `TranscriptMessage` (`types/logs.ts:221-231`):

```typescript
type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null        // Chain pointer to previous message
  logicalParentUuid?: UUID       // Logical parent preserved after compact
  isSidechain: boolean           // Sub-agent branch marker
  agentId?: string               // Sub-agent identifier
  promptId?: string              // OTel correlation
}
```

## 21 Entry Types

The `Entry` type is a discriminated union covering: `transcript`, `summary`, `custom-title`, `tag`, `agent-name`, `pr-link`, `file-history-snapshot`, `attribution-snapshot`, `content-replacement`, `context-collapse-commit`, and 11 more.

## Write Path — Batch Queue

`sessionStorage.ts:532-686` implements a `Project` class with batched writes:

1. `enqueueWrite(filePath, entry)` — Push to per-file queue
2. `scheduleDrain()` — Triggers every 100ms
3. `drainWriteQueue()` — Serialize to JSONL, append in 100MB chunks
4. `appendToFile()` — Auto `mkdir` fallback for NFS non-standard error codes

## Chain Reconstruction

`buildConversationChain()` (`sessionStorage.ts:2069-2093`):

```
1. Find latest leaf message
2. Walk parentUuid chain back to root
3. Reverse to get chronological order
4. Run recoverOrphanedParallelToolResults() post-pass
```

The `recoverOrphanedParallelToolResults` pass (`sessionStorage.ts:2096-2206`) handles DAG structures from parallel tool execution. When N parallel tool_use blocks generate N assistant messages (same message.id, different uuid), single-chain traversal only keeps one branch. The post-pass recovers orphans via message.id grouping + parentUuid reverse indexing.

## 64KB Recovery Boundary

`LITE_READ_BUF_SIZE = 65536` (`sessionStoragePortable.ts:17`). `readHeadAndTail()` reads first and last 64KB for lite metadata (title, tag, agentName), avoiding full file reads.

`reAppendSessionMetadata` (`sessionStorage.ts:450-465`) guarantees metadata always stays within the tail 64KB window by re-appending after writes.

## Interruption Detection

`detectTurnInterruption()` (`conversationRecovery.ts:164-333`):

| Type | Condition | Action |
|------|-----------|--------|
| `none` | Last message is assistant | Normal |
| `interrupted_prompt` | User input submitted, no response | Resume |
| `interrupted_turn` | Tool execution interrupted | Auto-inject "Continue from where you left off." |

## Performance Optimizations

- **50MB OOM protection** (`sessionStorage.ts:229`): `MAX_TRANSCRIPT_READ_BYTES = 50MB`. Tombstone rewrite operations above this threshold are rejected (fixes inc-3930).
- **Pre-compact skip** (`sessionStorage.ts:3536-3579`): Files above `SKIP_PRECOMPACT_THRESHOLD` get stream-read, skipping messages before compact boundary. Reduces memory from file size to output size (e.g., 151MB file → ~32MB allocation).
- **walkChainBeforeParse** (`sessionStorage.ts:3572-3579`): For >5MB post-boundary buffers, byte-level chain traversal pre-prunes dead branches before JSON parse.

## Unique Findings

1. **progressBridge legacy compat** (`sessionStorage.ts:3623-3641`): PR #24099 removed progress messages from chains. Loading builds `progressBridge: Map<UUID, UUID|null>` to skip progress nodes without modifying disk files.
2. **Concurrent write safety**: Writes use file locks (`lockfile.lock()`). `getHistory()` prioritizes current session entries to prevent cross-session history interleaving.
3. **Stale-while-error undone history** (`history.ts:442-464`): Two-level strategy — fast path pops from `pendingEntries`; if already flushed to disk (race lost), adds timestamp to `skipSet` for later filtering.
