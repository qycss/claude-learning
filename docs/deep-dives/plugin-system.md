# Plugin System

## Overview

Claude Code's Plugin system is a Marketplace-based distribution architecture, internally codenamed **Tengu**. It uses a **settings-first** design — declare intent by writing to settings, then materialize later. Each Plugin is a directory containing a `.claude-plugin/plugin.json` manifest, providing **7 extension types**: MCP servers, slash commands, agents, skills, hooks, LSP servers, and output styles.

## File Inventory

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/services/plugins/PluginInstallationManager.ts` | 185 | Background Marketplace installation, reconcile with AppState |
| `src/services/plugins/pluginOperations.ts` | 1089 | Core CRUD: install/uninstall/enable/disable/update |
| `src/services/plugins/pluginCliCommands.ts` | 345 | CLI wrapper, telemetry + process.exit management |
| `src/utils/plugins/schemas.ts` | 1682 | All Zod schemas (manifest/marketplace/install/source) |
| `src/utils/plugins/pluginLoader.ts` | 2000+ | Discovery/load/validate/cache engine |
| `src/utils/plugins/validatePlugin.ts` | 904 | Validation engine (manifest/marketplace/component) |
| `src/utils/plugins/pluginBlocklist.ts` | 128 | Delisting detection & auto-uninstall |
| `src/utils/plugins/mcpPluginIntegration.ts` | 400+ | MCP server loading from plugins (MCPB/DXT formats) |
| `src/utils/plugins/loadPluginHooks.ts` | 288 | Plugin hooks loading & hot reload |

## Three-Phase Loading

### Phase 1 — Declaration (`pluginOperations.ts:321-418`)

`installPluginOp()` searches materialized marketplaces, writes to settings (`enabledPlugins`), caches version info. This is the atomic intent declaration — execution can be deferred.

### Phase 2 — Materialization (`PluginInstallationManager.ts:60-184`)

`performBackgroundPluginInstallations()` calls `reconcileMarketplaces()` in background, using `diffMarketplaces()` to compute missing/sourceChanged state, driving git clone/npm install.

### Phase 3 — Loading (`pluginLoader.ts:0-150`)

`loadAllPlugins()` (memoized) reads cache/seed directories, parses `plugin.json` manifests, builds `LoadedPlugin` objects.

## Plugin Source Types

7 source types (`schemas.ts:1062-1161`):

| Type | Example |
|------|---------|
| Relative path | `./my-plugin` |
| npm | `@scope/plugin` (custom registry support) |
| pip | Python packages |
| URL | Git clone URLs |
| GitHub | `owner/repo` shorthand |
| git-subdir | Monorepo sparse clone |
| Local directory | Absolute path |

## Anti-Impersonation Security

Three-layer defense (`schemas.ts:9-101`):

1. **Reserved name whitelist** — `ALLOWED_OFFICIAL_MARKETPLACE_NAMES`
2. **Regex blocking** — `BLOCKED_OFFICIAL_NAME_PATTERN` detects "official" + "anthropic/claude" combinations
3. **Unicode homoglyph detection** — `NON_ASCII_PATTERN`
4. `validateOfficialNameSource()` requires reserved names to come from `anthropics` GitHub org

## Auto-Delisting

When `forceRemoveDeletedPlugins: true` (`pluginBlocklist.ts:64-127`), the system automatically detects installed plugins no longer listed in the marketplace and calls `uninstallPluginOp()`.

Dependency safety: `uninstallPluginOp()` calls `findReverseDependents()` (line 546-547) but **does not block** — only warns. This avoids tombstone problems where a delisted plugin prevents dependency graph teardown.

## Scope Isolation

Four scopes (`schemas.ts:1494-1509`):
```
managed > user > project > local
```

- `project` scope writes to `.claude/settings.json` (team-shared)
- `local` scope writes to `.claude/settings.local.json` (personal override)
- `disablePlugin --scope local` provides escape hatch without modifying team config

## PII Isolation

Plugin names are routed through `_PROTO_plugin_name` to PII-tagged BigQuery columns (`pluginCliCommands.ts:72-73`). Standard analytics metadata does not retain raw plugin names.
