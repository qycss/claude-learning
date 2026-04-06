# 遥测与隐私（Telemetry & Privacy）

## 概述

Claude Code 实现了**4 通道遥测架构**和 **3 级隐私控制**。系统通过通道分离、PII 路由和多重紧急开关，在可观测性需求与用户隐私之间取得平衡。

## 四大遥测通道

| 通道 | 传输方式 | 间隔 | 用途 |
|------|----------|------|------|
| **Datadog** | HTTP intake | 15 秒批量，每批最多 100 条 | 运维监控（约 40 个白名单事件） |
| **1P（First Party）** | OTel LoggerProvider → protobuf | 批量 | 完整事件日志，含 PII 标记列 |
| **BigQuery Metrics** | OTel MeterProvider → `/api/claude_code/metrics` | 1 分钟 | 聚合指标 |
| **OTel Traces** | OTLP/console + Perfetto | 实时 | 会话追踪（交互、工具调用、LLM 调用的 span） |

## 三级隐私控制

| 级别 | 触发条件 | 效果 |
|------|----------|------|
| `default` | 正常运行 | 所有遥测正常工作 |
| `no-telemetry` | `DISABLE_TELEMETRY=1` | 分析功能禁用，其他流量正常 |
| `essential-traffic` | `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` | 仅保留必要 API 调用（禁用分析、自动更新、发布说明、模型能力查询） |

### 自动禁用
第三方云服务商（Bedrock、Vertex、Foundry）**自动禁用**所有分析功能。例外：反馈调查不受第三方标志影响（它们是本地 UI 组件，不包含对话数据）。

## PII 保护

### `_PROTO_*` 字段路由

敏感数据的双通道设计：

```
事件负载:
  ├── regular_field: "sanitized value"        → Datadog（可访问）
  └── _PROTO_plugin_name: "raw plugin name"   → 仅 1P（受权限控制）
```

- `stripProtoFields()` 为 Datadog 通道移除 `_PROTO_*` 键
- 1P exporter 将其提升为 protobuf 顶层字段（受访问控制）
- 零分配优化：无 `_PROTO_` 键时直接返回原始对象引用

### MCP Tool 名称清洗
`sanitizeToolNameForAnalytics()` 将所有 `mcp__*` 工具名称替换为通用的 `'mcp_tool'`。工具名称可能暴露用户配置信息（例如使用了哪些 MCP server）。

### 类型级别安全
`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 是一个 `never` 类型标记，强制开发者显式转换，以确认值中不包含代码或文件路径。

## 紧急开关

### 远程紧急开关
`tengu_frond_boric` GrowthBook 配置提供按通道控制：
```json
{
  "datadog": false,      // 关闭 Datadog 通道
  "firstParty": true     // 保留 1P 通道
}
```

### 事件采样
`tengu_event_sampling_config` 控制按事件的采样率。被采样的事件在 metadata 中包含 `sample_rate` 以用于统计修正。

## 1P 事件恢复机制

`firstPartyEventLoggingExporter.ts`（806 行）实现了：

1. **JSONL 持久化**：失败事件追加写入 `~/.claude/telemetry/1p_failed_events.<batch_uuid>.jsonl`
2. **跨会话恢复**：启动时重试上一会话的失败批次
3. **指数退避**：基础 500ms，最大 30s，8 次尝试
4. **认证降级**：401 响应后尝试无认证重试
5. **内联紧急开关**：每次 POST 请求都检查紧急开关以实现即时生效

## 会话追踪（`sessionTracing.ts`）

- `AsyncLocalStorage` 维护交互/工具调用的 span 上下文
- `activeSpans` 使用 `WeakRef` 存储
- 非 ALS span（LLM 请求、等待用户阻塞）通过 `strongSpans` Map 保持存活
- 30 分钟的 `SPAN_TTL` 清理因中断流程而产生的孤立 span

## Perfetto 追踪（仅限 ant 用户）

`perfettoTracing.ts`（1,120 行）生成 Chrome Trace Event 格式：
- Process ID = agent ID（可视化 agent 层级结构）
- 追踪 TTFT（Time to First Token，首 token 时间）和 TTLT（Time to Last Token，末 token 时间）
- 写入 `~/.claude/traces/trace-<session-id>.json`
- 可在 `ui.perfetto.dev` 查看
- 通过 `CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S` 控制增量写入间隔

## 重要发现

1. **Tengu 代号**：所有 Datadog 事件以 `tengu_` 为前缀。紧急开关使用混淆名称 `tengu_frond_boric`。
2. **GrowthBook SDK 变通方案**（`growthbook.ts:330-393`）：API 返回 `value` 而非 `defaultValue`，SDK 的 `setForcedFeatures` 不可靠——因此完全绕过 SDK，直接缓存服务器返回结果。
3. **BigQuery 组织级别退出**：唯一由组织 API 响应控制的遥测通道。其他通道忽略该设置。
4. **Perfetto + OTel 并行**：同一工具调用通过统一的 `startToolSpan`/`endToolSpan` API 同时生成 OTel span 和 Perfetto 事件。
5. **事件队列解耦**（`index.ts:80-123`）：`logEvent()` 在 sink 初始化前将事件加入队列。`attachAnalyticsSink()` 通过 `queueMicrotask` 消费队列——不阻塞启动流程。
