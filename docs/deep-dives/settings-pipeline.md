# Settings Pipeline

## Overview

The Settings system is a **7-source layered merge pipeline**: Plugin base → userSettings → projectSettings → localSettings → flagSettings → policySettings. All sources validated via Zod v4, merged via lodash `mergeWith` (array dedup concat, `undefined` = delete). Enterprise policy delivered through MDM profiles and remote management server.

## File Inventory

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/utils/settings/settings.ts` | 1015 | 7-source load, merge, update |
| `src/utils/settings/types.ts` | 1148 | SettingsSchema (Zod v4), 700+ lines |
| `src/utils/settings/changeDetector.ts` | 488 | chokidar file watch + MDM polling |
| `src/utils/settings/settingsCache.ts` | 80 | Three-level cache |
| `src/utils/settings/validation.ts` | 265 | Zod error formatting + rule validation |
| `src/utils/settings/mdm/settings.ts` | 316 | MDM: macOS plist / Windows HKLM/HKCU |
| `src/utils/settings/mdm/rawRead.ts` | 130 | Zero-dependency subprocess raw read |
| `src/services/analytics/growthbook.ts` | 1155 | GrowthBook remoteEval + disk cache |

## 7-Source Merge

```
Plugin base (lowest priority)
  → userSettings (~/.claude/settings.json)
    → projectSettings (.claude/settings.json)
      → localSettings (.claude/settings.local.json)
        → flagSettings (SDK/CLI flags)
          → policySettings (highest priority)
```

### Policy "First Source Wins"

policySettings has its own internal precedence (`settings.ts:677-738`):
```
remote > HKLM/plist > managed-settings.json + drop-in dir > HKCU
```
First source with content wins — no merging between policy sources, avoiding conflicts.

## MDM Integration

### Parallel Loading
`startMdmSettingsLoad()` fires at earliest CLI stage (`cli.tsx`). `rawRead.ts` has zero heavy imports — can run during module evaluation:

```
main.tsx evaluates
  ├── rawRead.ts spawns subprocess (macOS: defaults read, Windows: reg query)
  ├── heavy modules loading...
  └── MDM result ready by the time settings.ts needs it
```

### Drop-in Directory
`managed-settings.d/` uses systemd/sudoers convention:
- Multiple teams deploy policy fragments independently
- Files sorted by name determine override priority (`10-otel.json`, `20-security.json`)

## Change Detection (`changeDetector.ts`)

- **chokidar** watches all settings file paths
- `awaitWriteFinish`: stabilityThreshold=1000ms, pollInterval=500ms
- Deletion events: `DELETION_GRACE_MS` grace period (handles delete-and-recreate pattern)
- MDM: independent 30-minute polling (registry/plist have no filesystem events)
- Internal writes: `markInternalWrite` + 5s `INTERNAL_WRITE_WINDOW_MS` suppression

## GrowthBook Feature Flags

### Remote Eval Mode
`remoteEval: true` — server pre-evaluates all features. Client caches to Map + disk.

### Three-Layer Override
```
env var (CLAUDE_INTERNAL_FC_OVERRIDES)
  > config overrides (ant-only /config Gates)
    > remote eval values
      > disk cache (offline fallback)
```

### Disk Cache
`cachedGrowthBookFeatures` written to global config, guaranteeing offline availability with stale-while-revalidate semantics.

## Zod Schema (`types.ts`)

700+ lines covering:
- Permission rules (allow/deny/ask)
- Hooks (12+ hook types with matchers)
- Sandbox config (network/filesystem)
- MCP server whitelist/blacklist
- Model overrides
- Environment variables
- Attribution settings

Uses `lazySchema()` for deferred construction avoiding circular references. `z.passthrough()` preserves unknown fields for backward compatibility.

## Unique Findings

1. **Import cycle leaf module** (`syncCacheState.ts`): Pure state leaf extracted from `syncCache.ts` to break `settings → syncCache → auth → settings` cycle. Comment explains SCC problem and timing constraints.
2. **`insideGetConfig` anti-recursion** (`config.ts:51`): Prevents `getConfig → logEvent → getGlobalConfig → getConfig` infinite loop.
3. **`CUSTOMIZATION_SURFACES`** (`types.ts:248-253`): `strictPluginOnlyCustomization` locks skills/agents/hooks/mcp surfaces to plugin-only modification — enterprise standardization tool.
4. **Three-level cache** (`settingsCache.ts`): parseFile → perSource → session merged, plus `pluginSettingsBase` for plugin defaults.
