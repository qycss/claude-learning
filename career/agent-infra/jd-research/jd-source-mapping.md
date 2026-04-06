# Agent Infra JD Keywords → Claude Code Source Mapping Table

> This table maps common skill requirements from AI Agent Infrastructure Engineer JDs
> to specific implementation modules in Claude Code (205K lines of TypeScript).
> Use it to quickly locate study priorities during interview preparation.
>
> Source root directory: `claude-code/src/`

| # | JD Keyword | Frequency | Source Module | Core Files | Lines | One-Line Description |
|---|-----------|-----------|--------------|------------|-------|---------------------|
| 1 | Agent Orchestration | High | `coordinator/` | `coordinatorMode.ts` | 369 | Supervisor-Worker orchestration, feature flag gating, coordinator/normal mode switching |
| 2 | Multi-Agent System | High | `tools/shared/` | `spawnMultiAgent.ts` | 1093 | Multi-Agent concurrent spawning, worktree isolation, task distribution and result collection |
| 3 | Sub-agent Spawning | High | `tools/AgentTool/` | `AgentTool.tsx`, `runAgent.ts` | 2370 | Sub-Agent lifecycle management, tool set authorization, fork/resume mechanism |
| 4 | LLM API Integration | High | `services/api/` | `claude.ts` | 3419 | Anthropic API client wrapper, streaming, usage accumulation, multi-provider support |
| 5 | Streaming & SSE | High | `QueryEngine.ts` | `QueryEngine.ts` | 1295 | LLM streaming response handling, tool-call loops, thinking mode, token counting |
| 6 | Retry & Error Handling | High | `services/api/` | `withRetry.ts` | 822 | Exponential backoff retry, 429/529 rate limiting, OAuth 401 refresh, fast-mode cooldown |
| 7 | Tool System / Function Calling | High | `Tool.ts`, `tools.ts` | `Tool.ts` | 792 | `buildTool()` factory + Zod schema input validation + before/after hooks |
| 8 | Permission & Access Control | High | `utils/permissions/` | `permissions.ts`, `permissionSetup.ts` | 9409 | Multi-modal permissions (default/plan/auto/bypass), classifier decisions, rule parsing |
| 9 | Sandboxed Execution | High | `utils/sandbox/` + `tools/BashTool/` | `sandbox-adapter.ts`, `shouldUseSandbox.ts` | 1138 | Command sandbox isolation, dangerous pattern detection, read-only mode validation |
| 10 | Security (Command Injection) | High | `tools/BashTool/` | `bashSecurity.ts` | 2592 | Shell command security validation, injection detection, path validation, destructive command warnings |
| 11 | State Management | High | `state/` | `AppStateStore.ts`, `AppState.tsx` | 768 | Zustand store, selector pattern, state change tracking |
| 12 | Context Window Management | High | `services/compact/` | `compact.ts`, `autoCompact.ts` | 3960 | Conversation context compression, auto-compact triggering, session memory persistence |
| 13 | MCP (Model Context Protocol) | High | `services/mcp/` | `client.ts`, `config.ts`, `auth.ts` | 7391 | MCP server connection management, channel permissions, OAuth elicitation, registry |
| 14 | Token Counting & Cost Tracking | Medium | `cost-tracker.ts`, `services/` | `tokenEstimation.ts`, `cost-tracker.ts` | 818 | Token estimation, API usage tracking, model pricing calculation |
| 15 | Prompt Engineering | Medium | `utils/` | `systemPrompt.ts`, `messages.ts` | 5635 | System prompt construction, message normalization (`normalizeMessagesForAPI`), XML tags |
| 16 | Feature Flags & A/B Testing | Medium | `services/analytics/` | `growthbook.ts` | 1155 | GrowthBook integration, Statsig gate caching, `feature('FLAG')` compile-time gating |
| 17 | OAuth 2.0 & Authentication | Medium | `services/oauth/` + `utils/` | `oauth/client.ts`, `utils/auth.ts` | 3053 | OAuth 2.0 PKCE flow, token refresh, keychain storage, multi-provider authentication |
| 18 | IDE/Extension Integration | Medium | `bridge/` | `bridgeMain.ts`, `bridgeMessaging.ts` | 3460 | JWT bidirectional communication, VS Code/JetBrains bridging, message protocol, session management |
| 19 | Plugin Architecture | Medium | `services/plugins/` | `pluginOperations.ts` | 1616 | Plugin discovery/installation/loading, CLI command registration, dependency management |
| 20 | Task Queue & Background Jobs | Medium | `tasks/` + `tools/TaskCreateTool/` | `Task.ts`, `types.ts`, `stopTask.ts` | 310 | Multiple task types (Local/Remote/Dream), task lifecycle management, background execution |
| 21 | Configuration Management | Medium | `utils/` + `services/remoteManagedSettings/` | `config.ts`, `remoteManagedSettings/index.ts` | 2455 | Multi-layer config merging (CLI/project/user/MDM), remote settings sync, caching |
| 22 | Observability & Telemetry | Medium | `services/analytics/` | `datadog.ts`, `firstPartyEventLogger.ts` | 4040 | Datadog integration, event log export, metadata collection, diagnostic tracing |
| 23 | LSP (Language Server Protocol) | Low | `services/lsp/` | `LSPClient.ts`, `LSPServerManager.ts` | 2460 | LSP server lifecycle management, diagnostics registration, passive feedback |
| 24 | Rate Limiting | Medium | `services/` | `rateLimitMessages.ts`, `api/errors.ts` | 1551 | 429/529 error classification, rate limit message display, mock rate limit testing |
| 25 | Graceful Shutdown & Cleanup | Low | `utils/` | `cleanup.ts`, `cleanupRegistry.ts` | 627 | Process cleanup registry, concurrent session management, resource release |
| 26 | Schema Validation (Zod) | High | `schemas/` + `entrypoints/sdk/` | `hooks.ts`, `coreSchemas.ts` | 2111 | Zod v4 schema definitions, SDK control/core schemas, hook configuration validation |
| 27 | Prompt Caching | Low | `services/api/` | `promptCacheBreakDetection.ts` | 727 | Cache break detection, cache hit rate optimization |
| 28 | CLI Framework | Medium | `main.tsx` + `commands.ts` | `main.tsx` | 4683 | Commander.js parsing, Ink renderer initialization, parallel prefetch |
| 29 | Hooks / Lifecycle Events | Medium | `utils/hooks/` + `schemas/` | `hookHelpers.ts`, `sessionHooks.ts` | 3721 | Async hook registry, agent/HTTP/prompt hooks, SSRF guard |
| 30 | Model Routing & Selection | Medium | `utils/model/` | `model.ts`, `modelOptions.ts`, `providers.ts` | 2710 | Model aliasing, Bedrock/Vertex adaptation, capability detection, deprecation management |

---

## Grouped by JD Skill Domain

### 1. Orchestration & Scheduling

**Keywords**: Agent Orchestration, Multi-Agent System, Sub-agent Spawning, Task Queue

- **coordinator/coordinatorMode.ts** (369 lines): Implements Supervisor-Worker orchestration pattern. Dual-gated via `feature('COORDINATOR_MODE')` compile-time flag and `CLAUDE_CODE_COORDINATOR_MODE` runtime env var. Defines coordinator and worker tool whitelists (INTERNAL_WORKER_TOOLS), supports session mode matching and recovery. Interview talking point: "I've studied production coordinator mode implementation — it uses feature flag + env var dual gating for rollout control, and supports coordinator/normal mode session resume."

- **tools/shared/spawnMultiAgent.ts** (1093 lines): Core logic for multi-Agent concurrent spawning. Handles worktree isolation (each agent gets an independent git worktree), task distribution, result collection and merging. Interview talking point: "In multi-Agent scenarios, each worker gets an independent workspace via git worktree; the primary agent handles task splitting and result aggregation — a classic fan-out/fan-in pattern."

- **tools/AgentTool/** (2580 lines): Complete sub-Agent lifecycle management. `AgentTool.tsx` defines agent creation, `runAgent.ts` manages execution loops, `forkSubagent.ts` handles agent forking. Supports built-in agents, memory snapshots, color management. Interview talking point: "Sub-agents are created via fork mechanism with independent tool set authorization and memory space, supporting resume checkpoint recovery."

- **tasks/** (310+ lines): Multiple task type abstractions — `LocalAgentTask` (local agent), `InProcessTeammateTask` (in-process collaboration), `RemoteAgentTask` (remote agent), `DreamTask` (background reasoning). Interview talking point: "The task system supports local/remote/background execution modes through polymorphic abstraction, unifying lifecycle management interfaces."

### 2. Fault Tolerance & Reliability

**Keywords**: Retry & Error Handling, Rate Limiting, Graceful Shutdown, Prompt Caching

- **services/api/withRetry.ts** (822 lines): Production-grade retry engine. Implements exponential backoff, jitter, max retry count control. Distinguishes retryable/non-retryable errors (429 rate limiting vs 400 bad request). Handles OAuth 401 token refresh, fast-mode overage degradation, AWS credentials refresh. Interview talking point: "The retry strategy distinguishes transient errors (network timeout/rate limiting) from permanent errors (auth failure/malformed request), implementing circuit-breaker-style fast-mode cooldown."

- **services/api/errors.ts** (1207 lines): Error classification and handling. `categorizeRetryableAPIError()` normalizes API errors into actionable categories. Includes dedicated handling for repeated 529s and user-friendly messages. Interview talking point: "Error handling uses a classification strategy pattern — each error type has corresponding recovery strategies and user prompts."

- **services/api/promptCacheBreakDetection.ts** (727 lines): Detects prompt cache breaks, optimizes cache hit rates. Interview talking point: "LLM prompt caching needs cache-break event detection and strategy adjustment — directly impacting latency and cost."

- **services/compact/** (3960 lines): Conversation context compression. Auto-triggers compact when message history exceeds token budget, extracting key information to session memory. Interview talking point: "Context management is the core challenge for long-running Agents — the compact mechanism uses LLM-as-summarizer to maintain context window utilization."

- **utils/cleanup.ts** + **cleanupRegistry.ts** (627 lines): Process lifecycle management. Registers cleanup callbacks, handles SIGINT/SIGTERM, manages concurrent sessions. Interview talking point: "Graceful shutdown uses a cleanup registry pattern ensuring all resources (child processes, temp files, network connections) are properly released."

### 3. Security & Permissions

**Keywords**: Permission & Access Control, Sandboxed Execution, Security, OAuth 2.0

- **utils/permissions/** (9409 lines): Complete multi-layer permission system. Supports 5 permission modes (default/plan/auto/bypass/acceptEdits). `yoloClassifier.ts` implements LLM-based auto-approval classifier. `permissionRuleParser.ts` parses user/project-level permission rules. `dangerousPatterns.ts` defines dangerous operation pattern matching. Interview talking point: "The permission system splits into rule layer (static matching) and classification layer (LLM dynamic judgment), supporting config merging from user/project/org/CLI multiple levels."

- **tools/BashTool/bashSecurity.ts** (2592 lines): Shell command security validation. Detects command injection, dangerous command patterns, path traversal. Interview talking point: "Every shell command executed by the Agent undergoes multi-dimensional security checks: syntax analysis, pattern matching, path validation — preventing command injection from prompt injection."

- **utils/sandbox/sandbox-adapter.ts** (985 lines): Sandbox adapter providing isolated environments for command execution. Interview talking point: "The sandbox isolates agent filesystem and network access permissions via namespace/container technology."

- **services/oauth/** (1051 lines): Complete OAuth 2.0 PKCE implementation. Authorization code listener, token management, profile retrieval. Interview talking point: "The auth system supports OAuth 2.0 PKCE flow, securely stores credentials via keychain, supports multi-provider switching."

- **bridge/jwtUtils.ts** (256 lines): JWT token generation and verification for IDE bridge communication authentication. Interview talking point: "IDE integration uses JWT-based bidirectional authentication ensuring only authorized IDE extensions can communicate with the agent."

- **hooks/toolPermission/** (626 lines): Tool-level permission context and logging. Every tool invocation passes through a permission gateway. Interview talking point: "Tool permissions are declarative — each tool defines its required permission level, with a unified permission gate intercepting and approving at runtime."

### 4. Observability

**Keywords**: Observability & Telemetry, Feature Flags, Token Counting, Logging

- **services/analytics/** (4040 lines): Complete telemetry system. `datadog.ts` integrates Datadog APM. `growthbook.ts` integrates GrowthBook for feature flags and A/B testing. `firstPartyEventLogger.ts` + `firstPartyEventLoggingExporter.ts` implement custom event logging pipeline. `metadata.ts` collects rich runtime metadata. Interview talking point: "The observability system uses multi-sink architecture — Datadog for APM, custom exporter for business event analysis, GrowthBook for experiment management, with killswitch support for emergency shutdown."

- **cost-tracker.ts** (323 lines) + **services/tokenEstimation.ts** (495 lines): Token usage and cost tracking. `getModelUsage()` / `getTotalCost()` / `getTotalAPIDuration()` provide real-time metrics. Interview talking point: "Cost tracking granularity reaches per-API-call level, supporting model/session dimension aggregation — an essential operational monitoring capability for LLM applications."

- **services/api/logging.ts** (788 lines): API call logging, including structured usage data, performance metrics. Interview talking point: "API layer structured logs include token usage, latency, cache hit and other key metrics, supporting downstream analysis and alerting."

- **services/diagnosticTracking.ts** (397 lines): Diagnostic event tracking for troubleshooting and performance analysis. Interview talking point: "The diagnostic system is independent from business telemetry, focusing on debug scenarios — error stacks, slow queries, abnormal state transitions."

### 5. Protocols & Communication

**Keywords**: MCP, LSP, IDE Integration, Streaming, SDK

- **services/mcp/** (7391+ lines): Complete Model Context Protocol implementation. `client.ts` (3348 lines) manages long connections to MCP servers. `auth.ts` (2465 lines) handles MCP OAuth elicitation. `config.ts` (1578 lines) manages server configuration. `channelPermissions.ts` controls channel-level permissions. Interview talking point: "MCP is the standardized protocol for Agent-Tool communication, implementing the complete lifecycle management of server discovery/connection/authentication/permissions — analogous to a microservice service mesh architecture."

- **bridge/** (3460+ lines): IDE bidirectional communication bridge. `bridgeMain.ts` (2999 lines) is the core — managing connection pool, message routing, session binding. `bridgeMessaging.ts` (461 lines) defines the message protocol. Supports VS Code and JetBrains. Interview talking point: "Bridge uses JWT-authenticated long connections with bidirectional message push, implementing session runner pattern for managing multiple concurrent IDE connections."

- **services/lsp/** (2460 lines): LSP client management. `LSPServerManager.ts` manages multiple LSP server instances. `LSPDiagnosticRegistry.ts` collects diagnostics as agent context input. Interview talking point: "The Agent obtains code diagnostic information (compile errors, lint warnings) via LSP, injecting them into the prompt as context for IDE-level code understanding."

- **entrypoints/sdk/** (2614 lines): Agent SDK integration layer. `coreSchemas.ts` (1889 lines) defines input/output Zod schemas. `controlSchemas.ts` (663 lines) defines control message schemas. Interview talking point: "The SDK layer defines agent input/output contracts via strict Zod schemas, ensuring type-safe cross-process communication."

- **QueryEngine.ts** (1295 lines): Core streaming engine. Manages LLM API call loop — sending requests, processing stream tokens, executing tool calls, handling thinking blocks. Interview talking point: "QueryEngine implements the complete agentic loop — streaming + tool-call + thinking nested loops, supporting abort, timeout, and token budget control."

### 6. State Management & Persistence

**Keywords**: State Management, Context Window, Configuration, Session

- **state/** (768 lines): Zustand-based global state container. `AppStateStore.ts` (569 lines) defines core store. `selectors.ts` (76 lines) provides derived state. `onChangeAppState.ts` implements change listening. Interview talking point: "State management uses Zustand (lightweight Redux alternative), avoiding unnecessary re-renders via selector pattern, supporting state change tracking."

- **services/compact/** (3960 lines): Conversation context persistence and compression. `compact.ts` (1705 lines) implements core compression algorithm — identifying key messages, generating summaries via LLM, preserving critical information from tool outputs. `sessionMemoryCompact.ts` (630 lines) manages cross-session memory persistence. `autoCompact.ts` (351 lines) implements token-usage-based auto-trigger. Interview talking point: "Context management is one of the biggest engineering challenges in Agent systems. The compact mechanism uses LLM-as-summarizer to compress historical context while preserving key information."

- **utils/config.ts** (1817 lines): Multi-layer configuration system. Merges CLI args / project settings / user settings / MDM remote config, supports hot updates. Interview talking point: "The config system implements the classic overlay pattern — configs from local to remote layers merge by priority, supporting MDM enterprise governance."

- **services/remoteManagedSettings/** (877 lines): Remote settings sync. Pulls org-level config from MDM endpoint. `syncCache.ts` for local caching and incremental sync. Interview talking point: "Enterprise scenarios require centralized config distribution — pulling policies via MDM API, local cache fallback, periodic sync updates."

- **history.ts** (464 lines): Session history management. Persists conversation records, supports history search and resume. Interview talking point: "Session persistence uses filesystem storage, supporting session resume — the foundation for long-running agents and checkpoint recovery."

- **utils/fileHistory.ts** (1115 lines) + **fileStateCache.ts** (142 lines): File change history and state cache. Provides undo capability for the agent — recording pre/post snapshots of each file modification. Interview talking point: "File change tracking via snapshot mechanism — saving state before each write/edit, supporting per-turn granularity rollback."

---

## Appendix: Quick Reference Index

### Source Module → JD Keyword Reverse Lookup

| Source Module | Covered JD Keywords |
|--------------|-------------------|
| `coordinator/` | Agent Orchestration, Multi-Agent |
| `tools/AgentTool/` | Sub-agent, Agent Lifecycle |
| `tools/shared/spawnMultiAgent.ts` | Multi-Agent, Fan-out/Fan-in |
| `services/api/` | LLM API, Retry, Error Handling, Streaming, Rate Limiting |
| `QueryEngine.ts` | Agentic Loop, Streaming, Tool Calling |
| `services/mcp/` | MCP Protocol, Tool Integration, Service Mesh |
| `services/compact/` | Context Window, Token Management, Memory |
| `bridge/` | IDE Integration, IPC, JWT Auth |
| `utils/permissions/` | RBAC, Access Control, Security |
| `tools/BashTool/` | Sandboxing, Security, Command Validation |
| `state/` | State Management, Zustand |
| `services/analytics/` | Observability, Feature Flags, A/B Testing |
| `services/lsp/` | LSP Protocol, Code Intelligence |
| `services/plugins/` | Plugin Architecture, Extensibility |
| `entrypoints/sdk/` | SDK Design, Schema Validation |
| `utils/hooks/` | Lifecycle Hooks, Event System |
| `utils/model/` | Model Routing, Multi-Provider |
| `tasks/` | Background Jobs, Task Queue |

### Interview High-Frequency Q&A Quick Reference

1. **"Describe an Agent orchestration system you've designed"** → See `coordinator/` + `tools/shared/spawnMultiAgent.ts`
2. **"How do you handle LLM API reliability?"** → See `services/api/withRetry.ts` + `errors.ts`
3. **"How do you approach Agent security?"** → See `utils/permissions/` + `tools/BashTool/bashSecurity.ts`
4. **"How do you manage context windows?"** → See `services/compact/compact.ts` + `autoCompact.ts`
5. **"How do you integrate external tools?"** → See `services/mcp/client.ts` + `Tool.ts`
6. **"What's your state management approach?"** → See `state/AppStateStore.ts` + `utils/config.ts`
7. **"How do you implement observability?"** → See `services/analytics/` + `cost-tracker.ts`
8. **"How do multiple Agents communicate?"** → See `bridge/bridgeMain.ts` + `tools/SendMessageTool/`
