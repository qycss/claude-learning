# AI Agent Engineer JD Keywords → Claude Code Source Mapping Table

> This table maps common skill requirements from AI Agent Engineer JDs
> to specific implementation modules in Claude Code (205K lines of TypeScript).
> Use it to quickly locate study priorities during interview preparation.
>
> Difference from the Agent Infra mapping table: This table focuses on the **application layer** (Prompt, RAG, evaluation, Agent behavior design),
> rather than the infrastructure layer (Runtime engine, process isolation, security sandbox).
>
> Source root directory: `claude-code/src/`

---

## Main Mapping Table

| # | JD Keyword | Frequency | Source Module | Core Files | Lines | One-Line Description |
|---|-----------|-----------|--------------|------------|-------|---------------------|
| 1 | Prompt Engineering / System Prompt Design | High | `utils/`, `tools/*/prompt.ts` | `utils/systemPrompt.ts`, `services/compact/prompt.ts`, `utils/claudemd.ts` | 123 + 374 + 1479 | Multi-priority system prompt builder: override > coordinator > agent > custom > default; 36 tools each with independent prompt.ts |
| 2 | RAG (Retrieval Augmented Generation) | High | `memdir/`, `services/MagicDocs/`, `services/SessionMemory/` | `memdir/findRelevantMemories.ts`, `services/MagicDocs/magicDocs.ts` | 141 + 254 | LLM side-query retrieves top 5 most relevant memories from memory files for context injection (lightweight vector DB alternative); MagicDocs self-maintained knowledge documents |
| 3 | Tool Use / Function Calling | High | `tools/` (42 subdirectories), `Tool.ts` | `Tool.ts`, `tools/BashTool/`, `tools/AgentTool/` | 792 + 184 files | `buildTool()` factory + Zod schema input validation + before/after hooks; all 42 tools permission-gated |
| 4 | Agent Loop / Agentic Workflow | High | `query.ts`, `QueryEngine.ts` | `query.ts`, `QueryEngine.ts` | 1729 + 1295 | Main loop orchestration: user input → LLM call → tool execution → result feedback; supports auto-compact, interrupt recovery, tool chain tracking |
| 5 | LLM API Integration | High | `services/api/` | `services/api/claude.ts`, `services/api/withRetry.ts` | 3419 + 822 | Anthropic SDK wrapper: stream/batch, cache control, Bedrock/Vertex/Foundry multi-backend, exponential backoff retry |
| 6 | Streaming / Real-time Response | High | `services/api/`, `utils/` | `services/api/claude.ts`, `utils/streamlinedTransform.ts` | 3419 + 201 | BetaRawMessageStreamEvent streaming processing, StreamEvent type normalization, Ink real-time rendering |
| 7 | Context Window Management | High | `utils/context.ts`, `services/compact/` | `services/compact/autoCompact.ts`, `services/compact/compact.ts` | 351 + 1705 | 200k/1M context detection, three-granularity compression (full/partial/micro), 6-tier graduated reclamation strategy |
| 8 | Memory / Conversation History | High | `memdir/`, `services/SessionMemory/`, `utils/sessionStorage.ts` | `memdir/memdir.ts`, `services/SessionMemory/sessionMemory.ts`, `utils/sessionStorage.ts` | 507 + 495 + 5105 | Three-layer memory: memdir persistent file memory + SessionMemory session-level summaries + sessionStorage JSONL full conversation |
| 9 | Multi-turn Conversation | High | `query.ts`, `utils/messages.ts` | `query.ts`, `utils/messages.ts`, `utils/sessionRestore.ts` | 1729 + 5512 + 551 | Message type normalization (7 types), session resume for history restoration, cross-compact boundary continuity |
| 10 | Chain-of-Thought / Reasoning | High | `utils/thinking.ts` | `utils/thinking.ts`, `tools/EnterPlanModeTool/` | 162 | ThinkingConfig three modes (adaptive/enabled/disabled), ultrathink keyword triggers deep reasoning |
| 11 | Agent Evaluation / Testing | Medium | `components/Feedback.tsx` | `components/Feedback.tsx`, `utils/permissions/yoloClassifier.ts` | 591 + 1495 | Feedback component collects thumbs up/down + text feedback; yoloClassifier performs safety assessment classification on tool calls |
| 12 | Prompt Optimization / Iteration | Medium | `services/compact/`, `services/MagicDocs/` | `services/compact/prompt.ts`, `memdir/findRelevantMemories.ts` | 374 + 141 | Compact prompt analysis+summary two-stage design; prompt template version management |
| 13 | User Experience / Interaction Design | Medium | `ink/`, `components/` | `components/` full directory, `ink/ink.tsx` | ~30 components | React+Ink terminal UI framework, virtual scrolling, vim mode, Dialog system, keyboard shortcuts |
| 14 | Error Handling / Graceful Degradation | High | `services/api/withRetry.ts`, `utils/` | `services/api/withRetry.ts`, `utils/conversationRecovery.ts` | 822 + ~200 | 10x exponential backoff retry, 529 overload special handling, fast-mode degradation, conversation recovery |
| 15 | Token Management / Cost Optimization | High | `utils/tokens.ts`, `utils/tokenBudget.ts` | `utils/tokens.ts`, `utils/modelCost.ts`, `utils/tokenBudget.ts` | 261 + 231 + 73 | Precise token counting, per-model pricing table, "+500k" natural language budget syntax, CAPPED_DEFAULT_MAX_TOKENS optimization |
| 16 | Safety / Content Filtering | High | `utils/permissions/`, `hooks/toolPermission/` | `utils/permissions/yoloClassifier.ts`, `hooks/toolPermission/` | 1495 + 1386 | All tools permission-gated; auto mode LLM classifier for safety judgment; Bash AST anti-injection |
| 17 | Multi-Agent Collaboration | Medium | `coordinator/`, `utils/swarm/` | `coordinator/coordinatorMode.ts`, `utils/swarm/inProcessRunner.ts` | 369 + 7169 | Coordinator orchestration, swarm three backends (tmux/iTerm/in-process), mailbox message passing |
| 18 | Agent Persona / Behavior Design | Medium | `tools/AgentTool/loadAgentsDir.ts` | `tools/AgentTool/loadAgentsDir.ts`, `components/agents/generateAgent.ts` | ~500 | Markdown-defined persona/system prompt/allowedTools, supports built-in and directory loading |
| 19 | Knowledge Base Integration | Medium | `utils/claudemd.ts`, `services/MagicDocs/` | `utils/claudemd.ts`, `services/MagicDocs/magicDocs.ts` | 1479 + 254 | claude.md hierarchical loading (global/project/local), MagicDocs self-maintained knowledge files |
| 20 | Embedding / Vector Search | Low | No direct implementation | `memdir/findRelevantMemories.ts` | 141 | No embedding/vector DB used; LLM side-query for semantic matching (lightweight alternative) |
| 21 | Fine-tuning / Model Customization | Low | `utils/model/` | `utils/model/model.ts`, `utils/model/modelOptions.ts` | ~500 | No fine-tuning, but supports 11+ model configuration switches; third-party models via capability override |
| 22 | Output Parsing / Structured Output | Medium | `entrypoints/sdk/`, `utils/zodToJsonSchema.ts` | `entrypoints/sdk/coreSchemas.ts`, `tools/SyntheticOutputTool/` | 1889 | SDK supports json_schema output format, Zod→JSON Schema auto-conversion, SyntheticOutputTool forces structured output |
| 23 | Guardrails / Output Validation | High | `hooks/toolPermission/`, `utils/permissions/` | `hooks/toolPermission/`, `schemas/hooks.ts` | 1386 + ~800 | Five permission modes, pre/post tool execution hooks, Bash AST security validation, path write protection |
| 24 | Feedback Loop / RLHF | Low | `components/Feedback.tsx` | `components/Feedback.tsx` | 591 | Thumbs up/down + text feedback, submitted to API endpoint; session transcripts exportable for analysis |
| 25 | API Design for Agent Services | Medium | `entrypoints/sdk/`, `entrypoints/mcp.ts` | `entrypoints/sdk/coreSchemas.ts`, `bridge/replBridge.ts` | 2614 + 2406 | Zod schema-first typed SDK; MCP standardized tool protocol; Bridge IDE bidirectional communication |
| 26 | Agent Monitoring / Analytics | Medium | `services/analytics/`, `utils/telemetry/` | `services/analytics/index.ts`, `utils/telemetry/sessionTracing.ts` | 4040 + 4044 | Event queue architecture, Datadog/BigQuery dual export, OpenTelemetry integration, GrowthBook feature flags |
| 27 | Latency Optimization | Medium | `utils/apiPreconnect.ts`, `main.tsx` | `utils/apiPreconnect.ts`, `utils/fastMode.ts` | 71 + 4683 | Startup parallel prefetch (MDM/keychain/API preconnect), TCP+TLS handshake 100-200ms early |
| 28 | Caching Strategy | Medium | `utils/statsCache.ts`, `services/api/claude.ts` | `utils/statsCache.ts`, `services/api/claude.ts` | 434 + 3419 | Prompt cache ephemeral breakpoints, lodash memoize, tool/file/settings cache |
| 29 | Plugin / Extension System | Medium | `utils/plugins/`, `services/mcp/` | `utils/plugins/pluginLoader.ts`, `services/mcp/client.ts` | 20452 + 12310 | Marketplace discovery/install/version management; MCP protocol extends tool set |
| 30 | Agent SDK / Developer Experience | Medium | `entrypoints/sdk/` | `entrypoints/sdk/coreSchemas.ts`, `entrypoints/sdk/controlSchemas.ts` | 2614 | Zod schema-first SDK, streaming event push, structured output, programmatic agent orchestration |

---

## Skill Domain Grouping Details

### Domain 1: Prompt Engineering & Knowledge (#1, #2, #12, #19, #18)

**Keywords**: Prompt Engineering, RAG, Prompt Optimization, Knowledge Base Integration, Agent Persona

**Core Files**:
- `src/utils/systemPrompt.ts` — Multi-layer system prompt priority builder
- `src/memdir/findRelevantMemories.ts` — LLM-based semantic memory retrieval
- `src/services/MagicDocs/magicDocs.ts` — Self-maintained knowledge documents
- `src/services/compact/prompt.ts` — Analysis+summary two-stage compact prompt
- `src/utils/claudemd.ts` — Hierarchical project instruction loading

**Interview Script**:

> "In Prompt Engineering, I designed a multi-priority System Prompt construction system (override > coordinator > agent > custom > default) ensuring correct prompt composition across different execution modes. For RAG, I adopted a lightweight LLM side-query approach: scanning memory file frontmatter metadata, using Sonnet for semantic matching, selecting the top 5 most relevant entries for context injection. This avoids the complexity of maintaining vector indices. For prompt optimization, the compact prompt uses an analysis+summary two-stage design — first letting the model fully reason in a scratchpad, then producing a structured summary, significantly improving information retention in context compression."

---

### Domain 2: Agent Runtime & Workflow (#3, #4, #5, #6, #9)

**Keywords**: Tool Use, Agent Loop, LLM API Integration, Streaming, Multi-turn Conversation

**Core Files**:
- `src/Tool.ts` — buildTool() factory + Zod input validation
- `src/query.ts` — Main agent loop orchestration
- `src/QueryEngine.ts` — LLM API call engine
- `src/services/api/claude.ts` — Anthropic SDK wrapper
- `src/utils/messages.ts` — Message type normalization

**Interview Script**:

> "The Agent main loop is orchestrated by query.ts: user input → LLM call → streaming parse → tool execution → result feedback loop, until the model decides to stop. The tool system uses a buildTool() factory pattern where each tool defines inputs via Zod schema, comes with prompt descriptions and permission rules, currently managing 42 tools. The LLM API layer wraps the Anthropic SDK's streaming calls, supporting Bedrock/Vertex/Foundry three cloud backend switches. Multi-turn conversation unifies 7 message types through normalizeMessagesForAPI(), and session restore can recover full conversation context across processes."

---

### Domain 3: Context & Memory Management (#7, #8, #10, #15, #28)

**Keywords**: Context Window Management, Memory/History, Chain-of-Thought, Token Management, Caching

**Core Files**:
- `src/services/compact/autoCompact.ts` — Auto-compact trigger
- `src/services/compact/compact.ts` — Compact execution
- `src/utils/tokens.ts` + `utils/tokenBudget.ts` — Token counting & budget
- `src/utils/modelCost.ts` — Cost calculation
- `src/utils/thinking.ts` — Chain-of-Thought configuration

**Interview Script**:

> "Context Window management is the core challenge in Agent engineering. I designed a three-layer solution: precise awareness — real-time tracking of context usage via API response usage fields; automatic compression — triggering compact when tokens approach the effective context window, using a forked agent to generate structured summaries replacing history; cost optimization — reducing output cap from 32k to 8k via CAPPED_DEFAULT_MAX_TOKENS (covering p99=4.9k of 99% requests), saving 8-16x slot reservation. Prompt cache ephemeral breakpoints reduce repeated context read costs to 1/12 of write costs."

---

### Domain 4: Safety, Guardrails & Evaluation (#11, #16, #23, #24, #14)

**Keywords**: Agent Evaluation, Safety, Guardrails, Feedback Loop, Error Handling

**Core Files**:
- `src/utils/permissions/yoloClassifier.ts` — LLM safety classifier
- `src/hooks/toolPermission/` — Permission gate system
- `src/services/api/withRetry.ts` — Retry & degradation
- `src/components/Feedback.tsx` — User feedback collection

**Interview Script**:

> "Agent safety uses multi-layer guardrails: permission gating — every tool call must pass permission checks, supporting default/plan/auto/bypass four modes; auto mode's LLM classifier uses independent requests to analyze command safety, combined with AST parsing to detect dangerous patterns; pre/post execution hooks support custom validation. Error handling implements 10x exponential backoff retry, 529 overload has a 3-attempt cap, fast-mode overage auto-degrades. Feedback loop collects thumbs up/down and text descriptions via the Feedback component, supporting offline analysis."

---

### Domain 5: Multi-Agent & Platform (#17, #25, #26, #27, #30)

**Keywords**: Multi-Agent, API Design, Monitoring, Latency Optimization, SDK

**Core Files**:
- `src/coordinator/coordinatorMode.ts` — Multi-agent coordinator
- `src/utils/swarm/` (7169 lines) — Parallel agent execution framework
- `src/entrypoints/sdk/coreSchemas.ts` — SDK type definitions
- `src/services/analytics/` (4040 lines) — Monitoring analytics

**Interview Script**:

> "The multi-agent Coordinator mode lets the primary agent orchestrate multiple teammates. The underlying swarm framework supports tmux process isolation, iTerm visual panels, and in-process lightweight execution — three backends with automatic selection based on environment. The SDK adopts a Zod schema-first strategy where all data types are schema-constrained, ensuring API contract type safety. For latency, startup runs parallel MDM config fetch, keychain pre-read, and API TCP+TLS preconnect, saving 100-200ms cold start."

---

### Domain 6: Extensibility & UX (#20, #21, #22, #29, #13)

**Keywords**: Embedding/Vector Search, Model Customization, Structured Output, Plugin System, UX

**Core Files**:
- `src/utils/plugins/pluginLoader.ts` (3302 lines) — Plugin loader
- `src/services/mcp/client.ts` (3348 lines) — MCP protocol client
- `src/tools/SyntheticOutputTool/` — Structured output
- `src/ink/` + `src/components/` — Terminal UI framework

**Interview Script**:

> "For platform extensibility, I designed a complete plugin system: marketplace discovery and installation, git repository import — two sources; plugins can inject commands/agents/hooks. MCP integration lets third-party tools plug in without source code changes. Structured output constrains return format via json_schema output format, with Zod→JSON Schema auto-conversion ensuring type consistency. Model adaptation supports 11+ model configuration switches; third-party models declare feature subsets via capability override."

---

## Appendix: Key Technical Highlights Quick Reference

| Technical Highlight | Source Location | Quantifiable Data |
|--------------------|----------------|-------------------|
| 42-tool permission-gated execution system | `src/tools/` (42 directories) | 36 prompt.ts files, 5 permission modes |
| Three-layer memory architecture | `memdir/ + SessionMemory/ + sessionStorage.ts` | 1736 + 819 + 5105 lines |
| Auto-compact context compression | `services/compact/` | 3960 lines, full/partial/micro three granularities |
| Swarm multi-agent parallel framework | `utils/swarm/` | 7169 lines, 3 backends |
| Complete plugin ecosystem | `utils/plugins/` | 20452 lines, marketplace + git + npm |
| MCP protocol standardized tool extension | `services/mcp/` | 12310 lines, stdio/SSE dual transport |
| Agent SDK type-safe interface | `entrypoints/sdk/` | 2614 lines, Zod schema-first |
| Event-driven monitoring system | `services/analytics/ + utils/telemetry/` | 8084 lines, Datadog + BigQuery + OTel |
