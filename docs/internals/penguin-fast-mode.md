# Penguin Mode (Fast Mode)

> **Source**: `src/utils/fastMode.ts` (533 lines)

## What It Is

**Penguin Mode** is the internal codename for Claude Code's Fast Mode. It does **not** switch to a different model — it uses the same Claude Opus 4.6 with a faster output channel.

## Key Details

| Component | Value |
|-----------|-------|
| API Endpoint | `/api/claude_code_penguin_mode` |
| GrowthBook Kill Switch | `tengu_penguins_off` |
| Model String | `'opus' + (isOpus1mMergeEnabled() ? '[1m]' : '')` |
| Persistent Cache | `penguinModeOrgEnabled` in global config |
| Min Prefetch Interval | 30 seconds (prevents API hammering) |

## State Machine

```
FastModeRuntimeState: 'active' | 'cooldown'
```

Cooldown uses the signal pattern:
```typescript
createSignal<[resetAt: number, reason: CooldownReason]>()
```

### CooldownReason
- `'rate_limit'` — Hit rate limits
- `'overloaded'` — API overloaded

## Organization-Level Control

```typescript
FastModeOrgStatus: 'pending' | 'enabled' | 'disabled'
```

### Disabled Reasons

| Reason | Description |
|--------|-------------|
| `'free'` | Free tier account |
| `'preference'` | Org preference disabled |
| `'extra_usage_disabled'` | Extra usage not enabled |
| `'network_error'` | Can't reach status endpoint |
| `'unknown'` | Fallback |

## Special Behaviors

- **Anthropic employees**: Default to `enabled` (`resolveFastModeStatusFromCache`)
- **Overage handling**: `handleFastModeOverageRejection()` handles 7+ specific overage scenarios
- **Prefetch caching**: Results cached to disk (`penguinModeOrgEnabled`), with 30-second minimum refetch interval
- **Toggle**: Users can toggle via `/fast` slash command

## API Integration

The fast mode endpoint is separate from the standard messages API:
```
Standard: POST /v1/messages
Fast:     POST /api/claude_code_penguin_mode
```

Both use the same model, but the fast endpoint routes through a different serving infrastructure optimized for lower latency with faster token output.
