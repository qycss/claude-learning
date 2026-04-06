# Model Codenames

## Overview

Claude Code uses internal codenames for models and the product itself. These codenames appear throughout the source code in feature flags, API endpoints, and configuration.

## Known Codename Mapping

| Codename | Maps To | Evidence |
|----------|---------|----------|
| **Tengu** (天狗) | Claude Code (the product) | All Datadog events prefixed `tengu_`, GrowthBook flags prefixed `tengu_` |
| **Capybara** | Claude Sonnet | Referenced in model routing code |
| **Fennec** | Pre-Opus 4.6 model | Referenced in model history |
| **Numbat** | Next unreleased model | Referenced in forward-looking code |
| **Penguin** | Fast Mode feature | API endpoint: `/api/claude_code_penguin_mode` |

## Codename Usage Patterns

### Tengu — Product Codename

Pervasive across the codebase:
- **Datadog events**: `tengu_session_start`, `tengu_query_complete`, etc.
- **GrowthBook flags**: `tengu_auto_mode_config`, `tengu_penguins_off`, `tengu_frond_boric`
- **API paths**: Various internal API endpoints

### Penguin — Fast Mode

- **API endpoint**: `/api/claude_code_penguin_mode`
- **Kill switch**: `tengu_penguins_off`
- **Cache key**: `penguinModeOrgEnabled`

## Codename Leak Prevention

The attribution system (`attribution.ts`) explicitly guards against codename leaks:

```
Unknown/internal model → hardcoded "Claude Opus 4.6"
```

This ensures that even if a new internal model (e.g., Numbat) is being tested, attribution text never exposes the codename. Only known, publicly-released model names appear in user-facing output.

## Feature Flag Obfuscation

See [Feature Flag Obfuscation](feature-flag-obfuscation.md) for how codenames are further obscured in feature flag naming.
