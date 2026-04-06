# Query Engine

## Two-Layer Architecture

The query system uses a two-layer AsyncGenerator coroutine design:

```
QueryEngine (session-scoped)
  └── query() (turn-scoped)
        └── StreamingToolExecutor (concurrent tools)
```

### Layer 1: QueryEngine (`src/QueryEngine.ts`)

Session-scoped AsyncGenerator that:
- Maintains conversation history across turns
- Manages system prompt (static + dynamic sections)
- Handles session-level abort via AbortController cascade
- Tracks token usage and budget
- Coordinates retry logic via `withRetry.ts`

### Layer 2: query() (`src/query.ts`)

Turn-scoped loop that handles a single user prompt → assistant response cycle:

```
while (true) {
  response = await streamAPICall(messages)

  if (response has tool_use blocks) {
    results = await executeTools(response.tool_use)
    messages.push(assistant_message, tool_results)
    continue  // loop back to API
  }

  break  // no more tool calls, turn complete
}
```

## Streaming Architecture

The API call uses Server-Sent Events (SSE) streaming:

```
API Stream → chunks → StreamingToolExecutor
                          ├── Text chunks → yield ProgressMessage
                          ├── Tool use start → begin tool execution
                          ├── Tool use end → collect result
                          └── Stop reason → end turn
```

### StreamingToolExecutor (`src/services/tools/StreamingToolExecutor.ts`)

Concurrent tool execution during model streaming with read-write lock semantics:
- **Read tools** (Grep, Glob, FileRead): Execute concurrently
- **Write tools** (FileWrite, FileEdit, Bash): Serialize execution
- Lock protects against concurrent file modifications

## Retry Engine (`src/services/api/withRetry.ts`)

629-line retry engine wrapping all API calls:

| Feature | Detail |
|---------|--------|
| Strategy | Exponential backoff with jitter |
| Max retries | Configurable (default 3) |
| Clock skew detection | Compares server `Date` header with local time |
| Rate limit handling | Respects `Retry-After` header |
| Overload detection | 529 status code → extended backoff |
| Abort integration | Checks AbortController between retries |

## AbortController Cascade

Three-layer abort hierarchy:

```
Session AbortController
  └── Sibling AbortController (per-turn)
        └── Per-Tool AbortController (per-tool invocation)
```

- Session abort → cancels all turns and tools
- Sibling abort → cancels current turn but not session
- Per-tool abort → cancels only one tool execution
- WeakRef used for GC safety on per-tool controllers

## Token Budget

The query loop enforces token budget:
- `maxTokens` per turn (configurable)
- `contextWindow` tracking (model-specific)
- Auto-compact triggers when context approaches limit
- Budget-aware tool execution (skip expensive tools when near limit)

## Message Normalization

`normalizeMessagesForAPI()` (`src/utils/messages.ts:1989-2138`) prepares messages:
- Filters progress and synthetic error messages
- Merges consecutive user messages (Bedrock requirement)
- Re-positions attachment messages
- Removes unavailable tool references
- Strips problematic PDF/image content blocks
