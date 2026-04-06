# State Management

## Zustand Store (`src/state/AppState.tsx`)

The central state container is a Zustand store at ~1,200 lines:

```typescript
// Simplified structure
const useAppState = create<AppState>((set, get) => ({
  // Conversation state
  messages: [],
  conversationId: null,

  // Session state
  sessionId: null,
  isLoading: false,
  permissionMode: 'default',

  // Tool state
  activeTools: [],
  pendingPermissions: [],

  // UI state
  scrollPosition: 0,
  searchQuery: '',

  // Settings
  settings: mergedSettings,

  // Actions
  addMessage: (msg) => set(state => ({ messages: [...state.messages, msg] })),
  // ... ~100+ actions
}))
```

## Store Architecture (`src/state/store.ts`)

Minimal 34-line Store abstraction:
- Wraps Zustand's `create()`
- Adds selector memoization
- Provides change tracking via subscription

## Message Types

The system uses normalized message types (`src/types/`):

| Type | Description |
|------|-------------|
| `UserMessage` | User input (text + attachments) |
| `AssistantMessage` | Model response (text + tool_use blocks) |
| `SystemMessage` | System prompt sections |
| `ToolUseBlock` | Tool invocation (name + input) |
| `ToolResultBlock` | Tool execution result |
| `ProgressMessage` | Streaming progress updates |

Messages are normalized via `normalizeMessagesForAPI()` before API calls.

## Settings State

Settings are loaded via the 7-source pipeline and stored in the Zustand store:

```
Plugin base → userSettings → projectSettings → localSettings → flagSettings → policySettings
```

Settings changes trigger store updates via `changeDetector.ts` (chokidar file watching).

## Persistence

State persistence uses two mechanisms:

### 1. Session Storage (JSONL)
- `src/utils/sessionStorage.ts` (~5,000 lines)
- Append-only JSONL log
- parentUuid chain structure
- Batch write queue (100ms drain interval)
- See [Session Persistence](../deep-dives/session-persistence.md) for full analysis

### 2. Global Config
- `src/utils/config.ts` (~500 lines)
- User preferences, project config, history
- `insideGetConfig` re-entrancy guard prevents infinite recursion (`getConfig → logEvent → getGlobalConfig → getConfig`)

## Change Tracking

The store provides subscription-based change tracking:
- Selectors for efficient re-rendering
- React Compiler integration (`_c()` memo cache) reduces unnecessary updates
- `OffscreenFreeze` prevents cascading dirty flags in long conversations (2800+ messages)
