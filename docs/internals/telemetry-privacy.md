# Telemetry & Privacy

## Overview

Claude Code implements a **4-channel telemetry architecture** with **3-level privacy controls**. The system balances observability needs with user privacy through channel separation, PII routing, and multiple kill switches.

## Four Telemetry Channels

| Channel | Transport | Interval | Purpose |
|---------|-----------|----------|---------|
| **Datadog** | HTTP intake | 15s batch, max 100/batch | Operational monitoring (~40 whitelisted events) |
| **1P (First Party)** | OTel LoggerProvider → protobuf | Batch | Full event logging with PII-tagged columns |
| **BigQuery Metrics** | OTel MeterProvider → `/api/claude_code/metrics` | 1 minute | Aggregated metrics |
| **OTel Traces** | OTLP/console + Perfetto | Real-time | Session tracing (spans for interactions, tools, LLM calls) |

## Three Privacy Levels

| Level | Trigger | Effect |
|-------|---------|--------|
| `default` | Normal operation | All telemetry active |
| `no-telemetry` | `DISABLE_TELEMETRY=1` | Analytics disabled, other traffic allowed |
| `essential-traffic` | `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` | Only essential API calls (no analytics, auto-update, release notes, model capabilities) |

### Automatic Disabling
Third-party cloud providers (Bedrock, Vertex, Foundry) **automatically disable** all analytics. Exception: feedback surveys are NOT disabled by 3P flag (they're local UI with no transcript data).

## PII Protection

### `_PROTO_*` Field Routing

A dual-channel design for sensitive data:

```
Event payload:
  ├── regular_field: "sanitized value"        → Datadog (accessible)
  └── _PROTO_plugin_name: "raw plugin name"   → 1P only (privileged access)
```

- `stripProtoFields()` removes `_PROTO_*` keys for Datadog channel
- 1P exporter promotes them to protobuf top-level fields (access-controlled)
- Zero-allocation optimization: no `_PROTO_` keys → return original object reference

### MCP Tool Name Sanitization
`sanitizeToolNameForAnalytics()` replaces all `mcp__*` tool names with generic `'mcp_tool'`. Tool names could reveal user configuration (which MCP servers they use).

### Type-Level Safety
`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` is a `never` type marker that forces developers to explicitly cast, confirming the value doesn't contain code or file paths.

## Kill Switches

### Remote Kill Switch
`tengu_frond_boric` GrowthBook config provides per-sink control:
```json
{
  "datadog": false,      // kill Datadog channel
  "firstParty": true     // keep 1P channel
}
```

### Event Sampling
`tengu_event_sampling_config` controls per-event sampling rates. Sampled events include `sample_rate` in metadata for statistical correction.

## 1P Event Resilience

`firstPartyEventLoggingExporter.ts` (806 lines) implements:

1. **JSONL persistence**: Failed events appended to `~/.claude/telemetry/1p_failed_events.<batch_uuid>.jsonl`
2. **Cross-session recovery**: Startup retries previous session's failed batches
3. **Exponential backoff**: Base 500ms, max 30s, 8 attempts
4. **Auth degradation**: 401 → retry without auth
5. **Inline kill switch**: Every POST checks kill switch for immediate effect

## Session Tracing (`sessionTracing.ts`)

- `AsyncLocalStorage` maintains interaction/tool span context
- `activeSpans` uses `WeakRef` storage
- Non-ALS spans (LLM request, blocked-on-user) kept alive via `strongSpans` Map
- 30-minute `SPAN_TTL` cleans orphan spans from interrupted flows

## Perfetto Tracing (ant-only)

`perfettoTracing.ts` (1,120 lines) generates Chrome Trace Event format:
- Process ID = agent ID (visualizes agent hierarchy)
- Tracks TTFT (Time to First Token) and TTLT (Time to Last Token)
- Writes to `~/.claude/traces/trace-<session-id>.json`
- Viewable at `ui.perfetto.dev`
- Incremental writes via `CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S`

## Unique Findings

1. **Tengu codename**: All Datadog events prefixed `tengu_`. Kill switch uses obfuscated name `tengu_frond_boric`.
2. **GrowthBook SDK workaround** (`growthbook.ts:330-393`): API returns `value` instead of `defaultValue`, SDK's `setForcedFeatures` unreliable → bypasses SDK entirely, caches server results directly.
3. **BigQuery org-level opt-out**: Only telemetry channel controlled by org API response. Others ignore it.
4. **Perfetto + OTel parallel**: Same tool call generates both OTel span and Perfetto event via unified `startToolSpan`/`endToolSpan` API.
5. **Event queue decoupling** (`index.ts:80-123`): `logEvent()` queues events before sink initialization. `attachAnalyticsSink()` drains via `queueMicrotask` — no startup blocking.
