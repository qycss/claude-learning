# AI Agent Engineer JD 关键词 → Claude Code 源码映射表

> 本表将 AI Agent Engineer 岗位 JD 中的常见技能要求，
> 映射到 Claude Code (205K 行 TypeScript) 中的具体实现模块。
> 用于面试准备时快速定位学习重点。
>
> 与 Agent Infra 版映射表的区别：本表侧重 **应用层**（Prompt、RAG、评估、Agent 行为设计），
> 而非底座层（Runtime 引擎、进程隔离、安全沙箱）。
>
> 源码根目录: `claude-code/src/`

---

## 主映射表

| # | JD 关键词 | 出现频率 | 源码模块 | 核心文件 | 行数 | 一句话说明 |
|---|----------|---------|---------|---------|------|----------|
| 1 | Prompt Engineering / System Prompt Design | 高 | `utils/`, `tools/*/prompt.ts` | `utils/systemPrompt.ts`, `services/compact/prompt.ts`, `utils/claudemd.ts` | 123 + 374 + 1479 | 多层优先级 system prompt 构建器：override > coordinator > agent > custom > default，36 个工具各有独立 prompt.ts |
| 2 | RAG (Retrieval Augmented Generation) | 高 | `memdir/`, `services/MagicDocs/`, `services/SessionMemory/` | `memdir/findRelevantMemories.ts`, `services/MagicDocs/magicDocs.ts` | 141 + 254 | LLM side-query 从 memory 文件中检索最相关的 5 条注入上下文（轻量级替代向量数据库）; MagicDocs 自维护知识文档 |
| 3 | Tool Use / Function Calling | 高 | `tools/` (42 个子目录), `Tool.ts` | `Tool.ts`, `tools/BashTool/`, `tools/AgentTool/` | 792 + 184 文件 | `buildTool()` 工厂 + Zod schema 输入验证 + before/after hooks，42 个工具全量 permission-gated |
| 4 | Agent Loop / Agentic Workflow | 高 | `query.ts`, `QueryEngine.ts` | `query.ts`, `QueryEngine.ts` | 1729 + 1295 | 主循环编排：用户输入→LLM 调用→工具执行→结果回传，支持 auto-compact、中断恢复、工具链跟踪 |
| 5 | LLM API Integration | 高 | `services/api/` | `services/api/claude.ts`, `services/api/withRetry.ts` | 3419 + 822 | Anthropic SDK 封装：stream/batch、cache control、Bedrock/Vertex/Foundry 多后端、指数退避重试 |
| 6 | Streaming / Real-time Response | 高 | `services/api/`, `utils/` | `services/api/claude.ts`, `utils/streamlinedTransform.ts` | 3419 + 201 | BetaRawMessageStreamEvent 流式处理，StreamEvent 类型归一化，Ink 实时渲染 |
| 7 | Context Window Management | 高 | `utils/context.ts`, `services/compact/` | `services/compact/autoCompact.ts`, `services/compact/compact.ts` | 351 + 1705 | 200k/1M 上下文检测，三粒度压缩（full/partial/micro），6 层分级回收策略 |
| 8 | Memory / Conversation History | 高 | `memdir/`, `services/SessionMemory/`, `utils/sessionStorage.ts` | `memdir/memdir.ts`, `services/SessionMemory/sessionMemory.ts`, `utils/sessionStorage.ts` | 507 + 495 + 5105 | 三层记忆：memdir 持久文件记忆 + SessionMemory 会话级摘要 + sessionStorage JSONL 全量对话 |
| 9 | Multi-turn Conversation | 高 | `query.ts`, `utils/messages.ts` | `query.ts`, `utils/messages.ts`, `utils/sessionRestore.ts` | 1729 + 5512 + 551 | 消息类型标准化（7 种），session resume 恢复历史对话，跨 compact 边界保持连续性 |
| 10 | Chain-of-Thought / Reasoning | 高 | `utils/thinking.ts` | `utils/thinking.ts`, `tools/EnterPlanModeTool/` | 162 | ThinkingConfig 三模式（adaptive/enabled/disabled），ultrathink 关键词触发深度推理 |
| 11 | Agent Evaluation / Testing | 中 | `components/Feedback.tsx` | `components/Feedback.tsx`, `utils/permissions/yoloClassifier.ts` | 591 + 1495 | Feedback 组件收集 thumbs up/down + 文字反馈；yoloClassifier 对工具调用做安全评估分类 |
| 12 | Prompt Optimization / Iteration | 中 | `services/compact/`, `services/MagicDocs/` | `services/compact/prompt.ts`, `memdir/findRelevantMemories.ts` | 374 + 141 | compact prompt 的 analysis+summary 两阶段设计；prompt 模板版本化管理 |
| 13 | User Experience / Interaction Design | 中 | `ink/`, `components/` | `components/` 全目录, `ink/ink.tsx` | ~30 组件 | React+Ink 终端 UI 框架，虚拟滚动，vim 模式，Dialog 系统，快捷键 |
| 14 | Error Handling / Graceful Degradation | 高 | `services/api/withRetry.ts`, `utils/` | `services/api/withRetry.ts`, `utils/conversationRecovery.ts` | 822 + ~200 | 10 次指数退避重试、529 过载特殊处理、fast-mode 降级、conversation recovery |
| 15 | Token Management / Cost Optimization | 高 | `utils/tokens.ts`, `utils/tokenBudget.ts` | `utils/tokens.ts`, `utils/modelCost.ts`, `utils/tokenBudget.ts` | 261 + 231 + 73 | 精确 token 计数，每模型定价表，"+500k" 自然语言预算语法，CAPPED_DEFAULT_MAX_TOKENS 优化 |
| 16 | Safety / Content Filtering | 高 | `utils/permissions/`, `hooks/toolPermission/` | `utils/permissions/yoloClassifier.ts`, `hooks/toolPermission/` | 1495 + 1386 | 全量工具 permission-gated；auto 模式 LLM 分类器判断安全性；Bash AST 防注入 |
| 17 | Multi-Agent Collaboration | 中 | `coordinator/`, `utils/swarm/` | `coordinator/coordinatorMode.ts`, `utils/swarm/inProcessRunner.ts` | 369 + 7169 | coordinator 编排，swarm 三 backend（tmux/iTerm/in-process），mailbox 消息传递 |
| 18 | Agent Persona / Behavior Design | 中 | `tools/AgentTool/loadAgentsDir.ts` | `tools/AgentTool/loadAgentsDir.ts`, `components/agents/generateAgent.ts` | ~500 | markdown 定义 persona/system prompt/allowedTools，支持 built-in 和目录加载 |
| 19 | Knowledge Base Integration | 中 | `utils/claudemd.ts`, `services/MagicDocs/` | `utils/claudemd.ts`, `services/MagicDocs/magicDocs.ts` | 1479 + 254 | claude.md 分层加载（全局/项目/本地），MagicDocs 自维护知识文件 |
| 20 | Embedding / Vector Search | 低 | 无直接实现 | `memdir/findRelevantMemories.ts` | 141 | 不使用 embedding/向量数据库，用 LLM side-query 做语义匹配（轻量替代方案） |
| 21 | Fine-tuning / Model Customization | 低 | `utils/model/` | `utils/model/model.ts`, `utils/model/modelOptions.ts` | ~500 | 不做 fine-tuning，但支持 11+ 种模型配置切换，第三方模型通过 capability override 接入 |
| 22 | Output Parsing / Structured Output | 中 | `entrypoints/sdk/`, `utils/zodToJsonSchema.ts` | `entrypoints/sdk/coreSchemas.ts`, `tools/SyntheticOutputTool/` | 1889 | SDK 支持 json_schema output format，Zod→JSON Schema 自动转换，SyntheticOutputTool 强制结构化 |
| 23 | Guardrails / Output Validation | 高 | `hooks/toolPermission/`, `utils/permissions/` | `hooks/toolPermission/`, `schemas/hooks.ts` | 1386 + ~800 | 五种权限模式，pre/post 工具执行 hook，Bash AST 安全校验，路径写保护 |
| 24 | Feedback Loop / RLHF | 低 | `components/Feedback.tsx` | `components/Feedback.tsx` | 591 | thumbs up/down + 文字反馈，提交到 API 端点；会话 transcript 可导出供分析 |
| 25 | API Design for Agent Services | 中 | `entrypoints/sdk/`, `entrypoints/mcp.ts` | `entrypoints/sdk/coreSchemas.ts`, `bridge/replBridge.ts` | 2614 + 2406 | Zod schema first 类型化 SDK；MCP 标准化工具协议；Bridge IDE 双向通信 |
| 26 | Agent Monitoring / Analytics | 中 | `services/analytics/`, `utils/telemetry/` | `services/analytics/index.ts`, `utils/telemetry/sessionTracing.ts` | 4040 + 4044 | 事件队列架构，Datadog/BigQuery 双出口，OpenTelemetry 集成，GrowthBook 特性开关 |
| 27 | Latency Optimization | 中 | `utils/apiPreconnect.ts`, `main.tsx` | `utils/apiPreconnect.ts`, `utils/fastMode.ts` | 71 + 4683 | 启动并行 prefetch（MDM/keychain/API preconnect），TCP+TLS 握手提前 100-200ms |
| 28 | Caching Strategy | 中 | `utils/statsCache.ts`, `services/api/claude.ts` | `utils/statsCache.ts`, `services/api/claude.ts` | 434 + 3419 | prompt cache ephemeral breakpoints，lodash memoize，tool/file/settings cache |
| 29 | Plugin / Extension System | 中 | `utils/plugins/`, `services/mcp/` | `utils/plugins/pluginLoader.ts`, `services/mcp/client.ts` | 20452 + 12310 | marketplace 发现/安装/版本管理；MCP 协议扩展工具集 |
| 30 | Agent SDK / Developer Experience | 中 | `entrypoints/sdk/` | `entrypoints/sdk/coreSchemas.ts`, `entrypoints/sdk/controlSchemas.ts` | 2614 | Zod schema first SDK，流式事件推送，structured output，programmatic agent 编排 |

---

## 按技能域分组详解

### 域 1：Prompt Engineering & Knowledge（#1, #2, #12, #19, #18）

**涉及关键词**: Prompt Engineering, RAG, Prompt Optimization, Knowledge Base Integration, Agent Persona

**核心文件**:
- `src/utils/systemPrompt.ts` — 多层 system prompt 优先级构建
- `src/memdir/findRelevantMemories.ts` — LLM-based 语义记忆检索
- `src/services/MagicDocs/magicDocs.ts` — 自维护知识文档
- `src/services/compact/prompt.ts` — analysis+summary 两阶段压缩 prompt
- `src/utils/claudemd.ts` — 分层项目指令加载

**面试话术**:

> "在 Prompt Engineering 方面，我设计了多层优先级的 System Prompt 构建体系（override > coordinator > agent > custom > default），确保不同运行模式下 prompt 的正确组合。RAG 方面采用了轻量级 LLM side-query 方案：扫描 memory 文件的 frontmatter 元数据，用 Sonnet 做语义匹配，选出最相关的 5 条注入上下文。这避免了维护向量索引的复杂度。Prompt 优化方面，compact prompt 采用 analysis+summary 两阶段设计，先让模型在 scratchpad 中完整推理，再产出结构化摘要，显著提升了上下文压缩的信息保留率。"

---

### 域 2：Agent Runtime & Workflow（#3, #4, #5, #6, #9）

**涉及关键词**: Tool Use, Agent Loop, LLM API Integration, Streaming, Multi-turn Conversation

**核心文件**:
- `src/Tool.ts` — buildTool() 工厂 + Zod 输入验证
- `src/query.ts` — 主 agent loop 编排
- `src/QueryEngine.ts` — LLM API 调用引擎
- `src/services/api/claude.ts` — Anthropic SDK 封装
- `src/utils/messages.ts` — 消息类型归一化

**面试话术**:

> "Agent 主循环由 query.ts 编排：用户输入→LLM 调用→流式解析→工具执行→结果回传的循环，直到模型决定 stop。工具系统采用 buildTool() 工厂模式，每个工具用 Zod schema 定义输入，自带 prompt 描述和权限规则，目前管理了 42 个工具。LLM API 层封装了 Anthropic SDK 的 streaming 调用，支持 Bedrock/Vertex/Foundry 三种云后端切换。多轮对话通过 normalizeMessagesForAPI() 统一 7 种消息类型，session restore 可跨进程恢复完整对话上下文。"

---

### 域 3：Context & Memory Management（#7, #8, #10, #15, #28）

**涉及关键词**: Context Window Management, Memory/History, Chain-of-Thought, Token Management, Caching

**核心文件**:
- `src/services/compact/autoCompact.ts` — 自动压缩触发
- `src/services/compact/compact.ts` — 压缩执行
- `src/utils/tokens.ts` + `utils/tokenBudget.ts` — token 计数与预算
- `src/utils/modelCost.ts` — 成本计算
- `src/utils/thinking.ts` — Chain-of-Thought 配置

**面试话术**:

> "Context Window 管理是 Agent 工程的核心挑战。我设计了三层方案：精确感知——通过 API response 的 usage 字段实时追踪上下文占用；自动压缩——当 token 接近 effective context window 时触发 compact，用 forked agent 生成结构化摘要替换历史；成本优化——通过 CAPPED_DEFAULT_MAX_TOKENS 把输出上限从 32k 降到 8k（覆盖 99% 请求的 p99=4.9k），节约 8-16 倍 slot 预留。Prompt cache 的 ephemeral breakpoints 让重复上下文读取成本降到写入的 1/12。"

---

### 域 4：Safety, Guardrails & Evaluation（#11, #16, #23, #24, #14）

**涉及关键词**: Agent Evaluation, Safety, Guardrails, Feedback Loop, Error Handling

**核心文件**:
- `src/utils/permissions/yoloClassifier.ts` — LLM 安全分类器
- `src/hooks/toolPermission/` — 权限门控系统
- `src/services/api/withRetry.ts` — 重试与降级
- `src/components/Feedback.tsx` — 用户反馈收集

**面试话术**:

> "Agent 安全采用多层 guardrails：permission gating——每个工具调用必过权限检查，支持 default/plan/auto/bypass 四种模式；auto 模式的 LLM 分类器用独立请求分析命令安全性，结合 AST 解析检测危险 pattern；pre/post 执行 hooks 支持自定义校验。错误处理实现了 10 次指数退避重试，529 过载有 3 次上限，fast-mode 超额自动降级。反馈循环通过 Feedback 组件收集 thumbs up/down 和文字描述，支持离线分析。"

---

### 域 5：Multi-Agent & Platform（#17, #25, #26, #27, #30）

**涉及关键词**: Multi-Agent, API Design, Monitoring, Latency Optimization, SDK

**核心文件**:
- `src/coordinator/coordinatorMode.ts` — 多 agent 协调器
- `src/utils/swarm/` (7169 行) — 并行 agent 执行框架
- `src/entrypoints/sdk/coreSchemas.ts` — SDK 类型定义
- `src/services/analytics/` (4040 行) — 监控分析

**面试话术**:

> "Multi-agent 系统的 Coordinator 模式让主 agent 编排多个 teammate。底层 swarm 框架支持 tmux 进程隔离、iTerm 可视化面板、in-process 轻量执行三种 backend，根据环境自动选择。SDK 采用 Zod schema first 策略，所有数据类型都是 schema 约束的，保证 API 契约的类型安全。延迟方面，启动时并行执行 MDM 配置拉取、keychain 预读、API TCP+TLS 预连接，节约 100-200ms 冷启动。"

---

### 域 6：Extensibility & UX（#20, #21, #22, #29, #13）

**涉及关键词**: Embedding/Vector Search, Model Customization, Structured Output, Plugin System, UX

**核心文件**:
- `src/utils/plugins/pluginLoader.ts` (3302 行) — 插件加载器
- `src/services/mcp/client.ts` (3348 行) — MCP 协议客户端
- `src/tools/SyntheticOutputTool/` — 结构化输出
- `src/ink/` + `src/components/` — 终端 UI 框架

**面试话术**:

> "平台可扩展性方面，设计了完整的插件系统：marketplace 发现与安装、git 仓库引入两种来源，插件可注入 commands/agents/hooks。MCP 集成让第三方工具无需改源码即可接入。结构化输出通过 json_schema output format 约束返回格式，Zod→JSON Schema 自动转换确保类型一致。模型适配支持 11+ 种模型配置切换，第三方模型通过 capability override 声明功能子集。"

---

## 附：关键技术亮点速查

| 技术亮点 | 源码位置 | 可量化数据 |
|---------|---------|-----------|
| 42 个工具的 permission-gated 执行系统 | `src/tools/` (42 目录) | 36 个 prompt.ts，5 种权限模式 |
| 三层记忆架构 | `memdir/ + SessionMemory/ + sessionStorage.ts` | 1736 + 819 + 5105 行 |
| 自动 compact 上下文压缩 | `services/compact/` | 3960 行，full/partial/micro 三粒度 |
| Swarm 多 agent 并行框架 | `utils/swarm/` | 7169 行，3 种 backend |
| 完整插件生态系统 | `utils/plugins/` | 20452 行，marketplace + git + npm |
| MCP 协议标准化工具扩展 | `services/mcp/` | 12310 行，stdio/SSE 双 transport |
| Agent SDK 类型安全接口 | `entrypoints/sdk/` | 2614 行，Zod schema first |
| 事件驱动监控体系 | `services/analytics/ + utils/telemetry/` | 8084 行，Datadog + BigQuery + OTel |
