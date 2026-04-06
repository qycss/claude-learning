# YOLO Classifier — Two-Stage Security Gatekeeper

## Overview

The YOLO Classifier is the security gatekeeper for auto/bypassPermissions modes. It uses a **two-stage classifier** (Fast + Thinking) with independent LLM API side queries to evaluate tool call safety. Core principle: **fail-closed** — any parse failure, API error, or timeout defaults to BLOCK.

## File Inventory

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/utils/permissions/yoloClassifier.ts` | 1496 | Core engine (prompt, transcript, two-stage XML/tool_use) |
| `src/utils/permissions/classifierDecision.ts` | 99 | Safe whitelist tools |
| `src/utils/permissions/classifierShared.ts` | 40 | Shared parsing utilities |
| `src/utils/permissions/denialTracking.ts` | 46 | Denial tracking & circuit breaker |
| `src/utils/permissions/autoModeState.ts` | 40 | Auto mode global state |
| `src/utils/permissions/bypassPermissionsKillswitch.ts` | 156 | Remote kill switch for bypass mode |
| `src/utils/permissions/dangerousPatterns.ts` | 81 | Dangerous bash command patterns |

## Two-Stage Architecture

### Stage 1 — Fast (`yoloClassifier.ts:769-858`)

```
max_tokens: 64
stop_sequences: ['</block>']
suffix: "Err on the side of blocking. <block> immediately."
```

Binary decision: `<block>yes</block>` or `<block>no</block>`
- allow → **immediate return** (fast path)
- block or unparseable → escalate to Stage 2

### Stage 2 — Thinking (`yoloClassifier.ts:860-996`)

```
max_tokens: 4096
suffix: "Review the classification process... Use <thinking> before responding with <block>."
```

Chain-of-thought reasoning via `<thinking>` tags reduces false positives. Both stages share the **same prompt prefix**, leveraging `cache_control` so Stage 2 gets a cache hit from Stage 1.

### Three Running Modes

| Mode | Behavior | Control |
|------|----------|---------|
| `'both'` | Default two-stage | — |
| `'fast'` | Stage 1 only (max_tokens=256) | GrowthBook |
| `'thinking'` | Stage 2 only | GrowthBook |

Controlled by `tengu_auto_mode_config.twoStageClassifier`.

## Transcript Security

`buildTranscriptEntries()` (`yoloClassifier.ts:302-360`):

**Critical decision at line 341-357**: Only `tool_use` blocks are included from assistant messages. All assistant text is excluded.

**Why**: Assistant text is model-authored and could be crafted via prompt injection to influence the classifier's decision. Tool use blocks are structured data that cannot be forged by the model's text output. This eliminates an entire attack surface.

## Fail-Closed Design

| Scenario | Result |
|----------|--------|
| API error | `shouldBlock: true` + `unavailable: true` |
| Parse failure | `shouldBlock: true` |
| Abort signal | `shouldBlock: true` |
| Empty input | `shouldBlock: false` (special: no safety-relevant input) |

## Denial Circuit Breaker (`denialTracking.ts`)

```
maxConsecutive: 3   // 3 consecutive denials
maxTotal: 20        // 20 total denials
```

`shouldFallbackToPrompting()` falls back to interactive prompting when thresholds are reached, preventing the classifier from permanently blocking the agent.

## Dangerous Permission Stripping (`permissionSetup.ts:83-119`)

When entering auto mode, existing allow rules are checked and dangerous patterns stripped:

| Pattern | Risk |
|---------|------|
| Tool-level allow (`Bash` with no ruleContent) | Allows all commands |
| Interpreter prefixes (`python:*`, `node:*`, 23 patterns) | Arbitrary code execution |
| Internal-only patterns (`gh api`, `curl`, `kubectl`, `aws`) | Infrastructure access |

## Remote Kill Switches

- `autoModeCircuitBroken`: GrowthBook `tengu_auto_mode_config.enabled === 'disabled'` blocks SDK/CLI from entering auto mode
- `checkAndDisableBypassPermissionsIfNeeded()`: Runtime disable of bypassPermissions
- `verifyAutoModeGateAccess()`: Startup + model switch gate, supports per-model auto mode checks

## Safe Whitelist

`SAFE_YOLO_ALLOWLISTED_TOOLS` (25+ tools) skip the classifier entirely. Notable:
- **FileWrite/FileEdit NOT whitelisted** — they have separate `acceptEdits` fast path (CWD: allow, outside CWD: classifier)
- **TeamCreateTool IS whitelisted** — "teammates have their own permission checks"

## Unique Findings

1. **Dual template system** (`yoloClassifier.ts:54-78`): Internal uses `permissions_anthropic.txt` (rules appended), external uses `permissions_external.txt` (rules replaced via `<user_*_to_replace>` tags)
2. **JSONL transcript migration**: Moving from text-prefix (`Bash ls`) to JSONL (`{"Bash":"ls"}`) — JSON escaping prevents content from forging `{"user":"..."}` lines
3. **alwaysOnThinking adaptation** (`yoloClassifier.ts:682-693`): Models with server-side adaptive thinking get 2048 extra tokens to prevent thinking from exhausting budget → `<block>` tag never output → false block
4. **CLAUDE.md injection** (`yoloClassifier.ts:460-477`): Classifier sees user's CLAUDE.md as `<user_claude_md>` context — "user intent" signal
5. **PowerShell mappings**: `iex (iwr ...)` → "Code from External", `Remove-Item -Recurse -Force` → "Irreversible Local Destruction"
6. **Context ratio tracking** (`yoloClassifier.ts:1073-1087`): Monitors `classifierTokensEst / mainLoopTokens` ratio (expected 0.6-0.8). If >1.0, auto-compact can't protect classifier → `transcript_too_long` error
