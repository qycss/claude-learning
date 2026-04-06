# Feature Flag Obfuscation

## Overview

Claude Code uses **intentionally opaque** feature flag names to prevent external observers from understanding what features are being tested or rolled out. The naming convention follows the pattern `tengu_<word1>_<word2>` where the words have no semantic relationship to the feature.

## Naming Pattern

```
tengu_<word1>_<word2>
```

- `tengu` — Product codename (天狗)
- `<word1>` and `<word2>` — Random, unrelated English words

## Known Examples

| Flag Name | Actual Purpose |
|-----------|---------------|
| `tengu_penguins_off` | Kill switch for Fast/Penguin Mode |
| `tengu_frond_boric` | Analytics sink kill switch (per-channel: datadog, firstParty) |
| `tengu_auto_mode_config` | YOLO classifier configuration (enabled, twoStageClassifier, etc.) |
| `tengu_event_sampling_config` | Telemetry event sampling rates |
| `tengu_log_datadog_events` | Datadog event logging gate |
| `tengu_ant_model_override` | Internal model override config |

## Why Obfuscation?

Feature flags are often visible in:
1. **Network traffic** — GrowthBook API calls include flag names
2. **Source code** — Even in compiled/bundled code, string literals survive
3. **Error messages** — Flag names may appear in logs
4. **Client storage** — Cached feature values written to disk

Obfuscated names prevent:
- Competitors from understanding A/B test strategies
- Users from manually overriding experimental features
- Security researchers from inferring unreleased capabilities

## GrowthBook Integration

Flags are evaluated via `remoteEval` mode — the server pre-evaluates all features based on user attributes and returns results. The flag names are the keys in this response.

Override hierarchy:
```
env var (CLAUDE_INTERNAL_FC_OVERRIDES)  ← highest
  > config overrides (ant-only /config Gates)
    > remote eval values
      > disk cache                     ← lowest
```

## Contrast with Direct Flag Names

Some flags use recognizable names when obfuscation isn't needed:
- `tengu_auto_mode_config` — Semi-recognizable (auto mode is a public feature)
- `tengu_penguins_off` — Semi-obfuscated (penguins = internal codename)
- `tengu_frond_boric` — Fully obfuscated (no connection to analytics)
