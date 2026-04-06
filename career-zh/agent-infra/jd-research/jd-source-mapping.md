# Agent Infra JD 关键词 → Claude Code 源码映射表

> 本表将 AI Agent Infrastructure Engineer 岗位 JD 中的常见技能要求，
> 映射到 Claude Code (205K 行 TypeScript) 中的具体实现模块。
> 用于面试准备时快速定位学习重点。
>
> 源码根目录: `claude-code/src/`

| # | JD 关键词 | 出现频率 | 源码模块 | 核心文件 | 行数 | 一句话说明 |
|---|----------|---------|---------|---------|------|----------|
| 1 | Agent Orchestration | 高 | `coordinator/` | `coordinatorMode.ts` | 369 | Supervisor-Worker 编排，feature flag 门控，coordinator/normal 模式切换 |
| 2 | Multi-Agent System | 高 | `tools/shared/` | `spawnMultiAgent.ts` | 1093 | 多 Agent 并发生成、worktree 隔离、任务分发与结果收集 |
| 3 | Sub-agent Spawning | 高 | `tools/AgentTool/` | `AgentTool.tsx`, `runAgent.ts` | 2370 | 子 Agent 生命周期管理、工具集授权、fork/resume 机制 |
| 4 | LLM API Integration | 高 | `services/api/` | `claude.ts` | 3419 | Anthropic API 客户端封装、streaming、usage 累计、多 provider 支持 |
| 5 | Streaming & SSE | 高 | `QueryEngine.ts` | `QueryEngine.ts` | 1295 | LLM 流式响应处理、tool-call 循环、thinking mode、token 计数 |
| 6 | Retry & Error Handling | 高 | `services/api/` | `withRetry.ts` | 822 | 指数退避重试、429/529 限流处理、OAuth 401 刷新、fast-mode cooldown |
| 7 | Tool System / Function Calling | 高 | `Tool.ts`, `tools.ts` | `Tool.ts` | 792 | `buildTool()` 工厂 + Zod schema 输入验证 + before/after hooks |
| 8 | Permission & Access Control | 高 | `utils/permissions/` | `permissions.ts`, `permissionSetup.ts` | 9409 | 多模态权限 (default/plan/auto/bypass)、classifier 决策、规则解析 |
| 9 | Sandboxed Execution | 高 | `utils/sandbox/` + `tools/BashTool/` | `sandbox-adapter.ts`, `shouldUseSandbox.ts` | 1138 | 命令沙箱隔离、dangerous pattern 检测、只读模式验证 |
| 10 | Security (Command Injection) | 高 | `tools/BashTool/` | `bashSecurity.ts` | 2592 | Shell 命令安全验证、注入检测、路径验证、破坏性命令警告 |
| 11 | State Management | 高 | `state/` | `AppStateStore.ts`, `AppState.tsx` | 768 | Zustand store、selector 模式、状态变更追踪 |
| 12 | Context Window Management | 高 | `services/compact/` | `compact.ts`, `autoCompact.ts` | 3960 | 对话上下文压缩、自动 compact 触发、session memory 持久化 |
| 13 | MCP (Model Context Protocol) | 高 | `services/mcp/` | `client.ts`, `config.ts`, `auth.ts` | 7391 | MCP server 连接管理、channel 权限、OAuth elicitation、registry |
| 14 | Token Counting & Cost Tracking | 中 | `cost-tracker.ts`, `services/` | `tokenEstimation.ts`, `cost-tracker.ts` | 818 | Token 预估、API 用量跟踪、模型定价计算 |
| 15 | Prompt Engineering | 中 | `utils/` | `systemPrompt.ts`, `messages.ts` | 5635 | 系统 prompt 构建、消息标准化 (`normalizeMessagesForAPI`)、XML 标签 |
| 16 | Feature Flags & A/B Testing | 中 | `services/analytics/` | `growthbook.ts` | 1155 | GrowthBook 集成、Statsig gate 缓存、`feature('FLAG')` 编译时门控 |
| 17 | OAuth 2.0 & Authentication | 中 | `services/oauth/` + `utils/` | `oauth/client.ts`, `utils/auth.ts` | 3053 | OAuth 2.0 PKCE 流程、token 刷新、keychain 存储、多 provider 认证 |
| 18 | IDE/Extension Integration | 中 | `bridge/` | `bridgeMain.ts`, `bridgeMessaging.ts` | 3460 | JWT 双向通信、VS Code/JetBrains 桥接、消息协议、session 管理 |
| 19 | Plugin Architecture | 中 | `services/plugins/` | `pluginOperations.ts` | 1616 | 插件发现/安装/加载、CLI 命令注册、依赖管理 |
| 20 | Task Queue & Background Jobs | 中 | `tasks/` + `tools/TaskCreateTool/` | `Task.ts`, `types.ts`, `stopTask.ts` | 310 | 多任务类型(Local/Remote/Dream)、task 生命周期管理、后台执行 |
| 21 | Configuration Management | 中 | `utils/` + `services/remoteManagedSettings/` | `config.ts`, `remoteManagedSettings/index.ts` | 2455 | 多层配置合并(CLI/project/user/MDM)、远程设置同步、缓存 |
| 22 | Observability & Telemetry | 中 | `services/analytics/` | `datadog.ts`, `firstPartyEventLogger.ts` | 4040 | Datadog 集成、事件日志导出、metadata 收集、诊断跟踪 |
| 23 | LSP (Language Server Protocol) | 低 | `services/lsp/` | `LSPClient.ts`, `LSPServerManager.ts` | 2460 | LSP server 生命周期管理、diagnostics 注册、被动反馈 |
| 24 | Rate Limiting | 中 | `services/` | `rateLimitMessages.ts`, `api/errors.ts` | 1551 | 429/529 错误分类、限流消息展示、mock 限流测试 |
| 25 | Graceful Shutdown & Cleanup | 低 | `utils/` | `cleanup.ts`, `cleanupRegistry.ts` | 627 | 进程清理注册表、并发 session 管理、资源释放 |
| 26 | Schema Validation (Zod) | 高 | `schemas/` + `entrypoints/sdk/` | `hooks.ts`, `coreSchemas.ts` | 2111 | Zod v4 schema 定义、SDK 控制/核心 schema、hook 配置验证 |
| 27 | Prompt Caching | 低 | `services/api/` | `promptCacheBreakDetection.ts` | 727 | 缓存断裂检测、cache 命中率优化 |
| 28 | CLI Framework | 中 | `main.tsx` + `commands.ts` | `main.tsx` | 4683 | Commander.js 解析、Ink 渲染器初始化、并行 prefetch |
| 29 | Hooks / Lifecycle Events | 中 | `utils/hooks/` + `schemas/` | `hookHelpers.ts`, `sessionHooks.ts` | 3721 | 异步 hook 注册表、agent/HTTP/prompt hook、SSRF guard |
| 30 | Model Routing & Selection | 中 | `utils/model/` | `model.ts`, `modelOptions.ts`, `providers.ts` | 2710 | 模型别名、Bedrock/Vertex 适配、能力检测、deprecation 管理 |

---

## 按 JD 技能域分组详解

### 1. 编排与调度 (Orchestration & Scheduling)

**涉及关键词**: Agent Orchestration, Multi-Agent System, Sub-agent Spawning, Task Queue

- **coordinator/coordinatorMode.ts** (369 行): 实现 Supervisor-Worker 编排模式。通过 `feature('COORDINATOR_MODE')` 编译时门控和运行时环境变量 `CLAUDE_CODE_COORDINATOR_MODE` 双重控制。定义了 coordinator 和 worker 的工具白名单（INTERNAL_WORKER_TOOLS），支持 session 模式匹配和恢复。面试时可引用："我研究过生产级 coordinator 模式的实现，它通过 feature flag + env var 双重门控来控制 rollout，并支持 coordinator/normal 模式的 session resume。"

- **tools/shared/spawnMultiAgent.ts** (1093 行): 多 Agent 并发生成的核心逻辑。处理 worktree 隔离（每个 agent 独立 git worktree）、任务分发、结果收集与合并。面试时可引用："在多 Agent 场景中，每个 worker 通过 git worktree 获得独立工作空间，主 agent 负责任务拆分和结果聚合，这是典型的 fan-out/fan-in 模式。"

- **tools/AgentTool/** (2580 行): 子 Agent 完整生命周期管理。`AgentTool.tsx` 定义 agent 创建、`runAgent.ts` 管理执行循环、`forkSubagent.ts` 处理 agent 分叉。支持 built-in agents、内存快照、颜色管理。面试时可引用："子 agent 通过 fork 机制创建，拥有独立的工具集授权和内存空间，支持 resume 断点恢复。"

- **tasks/** (310+ 行): 多种任务类型抽象——`LocalAgentTask`(本地 agent)、`InProcessTeammateTask`(进程内协作)、`RemoteAgentTask`(远程 agent)、`DreamTask`(后台推理)。面试时可引用："任务系统通过多态抽象支持本地/远程/后台等不同执行模式，统一了生命周期管理接口。"

### 2. 容错与可靠性 (Fault Tolerance & Reliability)

**涉及关键词**: Retry & Error Handling, Rate Limiting, Graceful Shutdown, Prompt Caching

- **services/api/withRetry.ts** (822 行): 生产级重试引擎。实现指数退避、jitter、最大重试次数控制。区分 retryable/non-retryable 错误（429 限流 vs 400 请求错误）。处理 OAuth 401 token 刷新、fast-mode 超额降级、AWS credentials refresh。面试时可引用："重试策略区分了暂时性错误（网络超时/限流）和永久性错误（认证失败/请求畸形），并实现了 circuit-breaker 式的 fast-mode cooldown 机制。"

- **services/api/errors.ts** (1207 行): 错误分类与处理。`categorizeRetryableAPIError()` 将 API 错误标准化为可操作的分类。包含 repeated 529 的专用处理逻辑和用户友好消息。面试时可引用："错误处理采用分类策略模式，每种错误类型有对应的恢复策略和用户提示。"

- **services/api/promptCacheBreakDetection.ts** (727 行): 检测 prompt cache 断裂、优化缓存命中率。面试时可引用："LLM prompt caching 需要检测 cache-break 事件并调整策略，这直接影响延迟和成本。"

- **services/compact/** (3960 行): 对话上下文压缩。当消息历史超过 token 预算时自动触发 compact，将关键信息提取到 session memory。面试时可引用："上下文管理是 Agent 长时运行的核心挑战，compact 机制通过 LLM 自身做摘要来保持上下文窗口利用率。"

- **utils/cleanup.ts** + **cleanupRegistry.ts** (627 行): 进程生命周期管理。注册清理回调、处理 SIGINT/SIGTERM、管理并发 session。面试时可引用："优雅关闭通过 cleanup registry 模式确保所有资源（子进程、临时文件、网络连接）被正确释放。"

### 3. 安全与权限 (Security & Permissions)

**涉及关键词**: Permission & Access Control, Sandboxed Execution, Security, OAuth 2.0

- **utils/permissions/** (9409 行): 完整的多层权限系统。支持 5 种权限模式（default/plan/auto/bypass/acceptEdits）。`yoloClassifier.ts` 实现基于 LLM 的自动审批分类器。`permissionRuleParser.ts` 解析用户/项目级权限规则。`dangerousPatterns.ts` 定义危险操作模式匹配。面试时可引用："权限系统分为规则层（静态匹配）和分类层（LLM 动态判断），支持从 user/project/org/CLI 多个层级合并权限配置。"

- **tools/BashTool/bashSecurity.ts** (2592 行): Shell 命令安全验证。检测命令注入、危险命令模式、路径遍历。面试时可引用："对 Agent 执行的每条 shell 命令进行多维度安全检查：语法分析、模式匹配、路径验证，防止 prompt injection 导致的命令注入。"

- **utils/sandbox/sandbox-adapter.ts** (985 行): 沙箱适配器，为命令执行提供隔离环境。面试时可引用："沙箱通过 namespace/容器技术隔离 agent 的文件系统和网络访问权限。"

- **services/oauth/** (1051 行): OAuth 2.0 PKCE 完整实现。authorization code listener、token 管理、profile 获取。面试时可引用："认证系统支持 OAuth 2.0 PKCE 流程，通过 keychain 安全存储 credentials，支持多 provider 切换。"

- **bridge/jwtUtils.ts** (256 行): JWT token 生成与验证，用于 IDE 桥接通信的身份认证。面试时可引用："IDE 集成使用 JWT-based 双向认证，确保只有授权的 IDE 扩展可以与 agent 通信。"

- **hooks/toolPermission/** (626 行): 工具级权限上下文与日志记录。每次工具调用都经过权限网关。面试时可引用："工具权限是声明式的——每个 tool 定义自己需要的权限级别，运行时由统一的 permission gate 做拦截和审批。"

### 4. 可观测性 (Observability & Monitoring)

**涉及关键词**: Observability & Telemetry, Feature Flags, Token Counting, Logging

- **services/analytics/** (4040 行): 完整的遥测系统。`datadog.ts` 集成 Datadog APM。`growthbook.ts` 集成 GrowthBook 做 feature flag 和 A/B 测试。`firstPartyEventLogger.ts` + `firstPartyEventLoggingExporter.ts` 实现自研事件日志管道。`metadata.ts` 采集丰富的运行时元数据。面试时可引用："可观测性系统采用多 sink 架构——Datadog 做 APM、自研 exporter 做业务事件分析、GrowthBook 做实验管理，通过 killswitch 支持紧急关闭。"

- **cost-tracker.ts** (323 行) + **services/tokenEstimation.ts** (495 行): Token 用量与成本跟踪。`getModelUsage()` / `getTotalCost()` / `getTotalAPIDuration()` 提供实时度量。面试时可引用："成本追踪粒度到每次 API 调用，支持按 model/session 维度聚合，是 LLM 应用必备的运营监控能力。"

- **services/api/logging.ts** (788 行): API 调用日志记录，包括 usage 结构化数据、性能指标。面试时可引用："API layer 的结构化日志包含 token usage、latency、cache hit 等关键指标，支持下游分析和报警。"

- **services/diagnosticTracking.ts** (397 行): 诊断事件追踪，用于问题排查和性能分析。面试时可引用："诊断系统独立于业务遥测，专注于 debug 场景——错误堆栈、慢查询、异常状态转换。"

### 5. 协议与通信 (Protocols & Communication)

**涉及关键词**: MCP, LSP, IDE Integration, Streaming, SDK

- **services/mcp/** (7391+ 行): Model Context Protocol 完整实现。`client.ts` (3348 行) 管理与 MCP server 的长连接。`auth.ts` (2465 行) 处理 MCP OAuth elicitation。`config.ts` (1578 行) 管理 server 配置。`channelPermissions.ts` 控制 channel 级权限。面试时可引用："MCP 是 Agent-Tool 通信的标准化协议，实现了 server 发现/连接/认证/权限的完整生命周期管理，类似微服务的 service mesh 架构。"

- **bridge/** (3460+ 行): IDE 双向通信桥接。`bridgeMain.ts` (2999 行) 是核心——管理连接池、消息路由、session 绑定。`bridgeMessaging.ts` (461 行) 定义消息协议。支持 VS Code 和 JetBrains。面试时可引用："Bridge 采用 JWT-authenticated 长连接，支持双向消息推送，实现了 session runner 模式来管理多个并发 IDE 连接。"

- **services/lsp/** (2460 行): LSP 客户端管理。`LSPServerManager.ts` 管理多个 LSP server 实例。`LSPDiagnosticRegistry.ts` 收集 diagnostics 作为 agent 的上下文输入。面试时可引用："Agent 通过 LSP 获取代码诊断信息（编译错误、lint 警告），将其注入 prompt 作为上下文，实现 IDE-level 的代码理解。"

- **entrypoints/sdk/** (2614 行): Agent SDK 集成层。`coreSchemas.ts` (1889 行) 定义输入/输出的 Zod schema。`controlSchemas.ts` (663 行) 定义控制消息 schema。面试时可引用："SDK 层通过严格的 Zod schema 定义 agent 的输入/输出契约，确保类型安全的跨进程通信。"

- **QueryEngine.ts** (1295 行): 核心流式处理引擎。管理 LLM API 调用循环——发送请求、处理 stream token、执行 tool calls、处理 thinking blocks。面试时可引用："QueryEngine 实现了完整的 agentic loop——streaming + tool-call + thinking 的嵌套循环，支持 abort、timeout、token budget 控制。"

### 6. 状态管理与持久化 (State Management & Persistence)

**涉及关键词**: State Management, Context Window, Configuration, Session

- **state/** (768 行): Zustand-based 全局状态容器。`AppStateStore.ts` (569 行) 定义核心 store。`selectors.ts` (76 行) 提供派生状态。`onChangeAppState.ts` 实现变更监听。面试时可引用："状态管理采用 Zustand（轻量级 Redux 替代），通过 selector 模式避免不必要的 re-render，支持状态变更追踪。"

- **services/compact/** (3960 行): 对话上下文持久化与压缩。`compact.ts` (1705 行) 实现核心压缩算法——识别关键消息、通过 LLM 生成摘要、保留 tool 输出中的关键信息。`sessionMemoryCompact.ts` (630 行) 管理跨 session 的记忆持久化。`autoCompact.ts` (351 行) 实现基于 token 用量的自动触发。面试时可引用："上下文管理是 Agent 系统最大的工程挑战之一。compact 机制用 LLM-as-summarizer 方式在保留关键信息的同时压缩历史上下文。"

- **utils/config.ts** (1817 行): 多层配置系统。合并 CLI args / project settings / user settings / MDM remote config，支持热更新。面试时可引用："配置系统实现了经典的 overlay 模式——从本地到远程多个层级的配置按优先级合并，支持 MDM 企业管控。"

- **services/remoteManagedSettings/** (877 行): 远程设置同步。从 MDM endpoint 拉取组织级配置。`syncCache.ts` 本地缓存与增量同步。面试时可引用："企业场景需要中心化配置下发——通过 MDM API 拉取策略、本地缓存降级、定时同步更新。"

- **history.ts** (464 行): 会话历史管理。持久化对话记录、支持历史搜索和 resume。面试时可引用："Session 持久化采用文件系统存储，支持会话恢复（resume），是 agent 长时运行和断点续传的基础。"

- **utils/fileHistory.ts** (1115 行) + **fileStateCache.ts** (142 行): 文件变更历史与状态缓存。为 agent 提供 undo 能力——记录每个文件修改前后的快照。面试时可引用："文件变更追踪通过 snapshot 机制实现，每次 write/edit 前保存状态，支持按 turn 粒度回滚。"

---

## 附录: 快速参考索引

### 按源码模块反查 JD 关键词

| 源码模块 | 覆盖的 JD 关键词 |
|---------|----------------|
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

### 面试高频问答速查

1. **"请描述你设计过的 Agent 编排系统"** → 看 `coordinator/` + `tools/shared/spawnMultiAgent.ts`
2. **"你如何处理 LLM API 的可靠性问题"** → 看 `services/api/withRetry.ts` + `errors.ts`
3. **"Agent 安全怎么做"** → 看 `utils/permissions/` + `tools/BashTool/bashSecurity.ts`
4. **"上下文窗口怎么管理"** → 看 `services/compact/compact.ts` + `autoCompact.ts`
5. **"如何集成外部工具"** → 看 `services/mcp/client.ts` + `Tool.ts`
6. **"状态管理方案"** → 看 `state/AppStateStore.ts` + `utils/config.ts`
7. **"可观测性怎么做"** → 看 `services/analytics/` + `cost-tracker.ts`
8. **"多 Agent 如何通信"** → 看 `bridge/bridgeMain.ts` + `tools/SendMessageTool/`
