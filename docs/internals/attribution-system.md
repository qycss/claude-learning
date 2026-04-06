# Attribution System

> **Source**: `src/utils/attribution.ts` (394 lines)

## Overview

The Attribution system generates credit/attribution text for commits and pull requests, with an "enhanced" format that includes detailed statistics about AI contribution.

## Enhanced PR Attribution

Format example:
```
🤖 Generated with Claude Code (93% 3-shotted by claude-opus-4-5, 2 memories recalled)
```

### Components

| Component | Source | Description |
|-----------|--------|-------------|
| `claudePercent` | Calculated | AI contribution percentage |
| `promptCount` | `getTranscriptStats()` | User prompts (post-compact-boundary) |
| `memoryAccessCount` | `getTranscriptStats()` | Memory system accesses |
| Model name | Sanitized | Unknown models → hardcoded "Claude Opus 4.6" |

### N-Shot Terminology

The "N-shotted" label counts user prompts:
- 0-shot: AI completed with zero additional prompts
- 1-shot: One user prompt
- N-shot: N user prompts needed

This provides transparency about how much human guidance was required.

## Model Name Sanitization

Unknown or internal model names are **never exposed** in attribution:
- Known models: use their public name (e.g., "claude-opus-4-6")
- Unknown models: fallback to hardcoded "Claude Opus 4.6"

This prevents internal codenames (Capybara, Fennec, Numbat) from leaking through commit messages or PR descriptions.

## Undercover Integration

When `isUndercover()` returns `true`:
- All attribution functions return **empty strings**
- No commit trailers added
- No PR footer generated
- Complete AI attribution suppression

## Transcript Statistics

`getTranscriptStats()` analyzes the conversation:
- Counts user prompts **after compact boundary** (not total)
- Counts memory access events
- Provides data for enhanced attribution calculation

## Commit Attribution

Standard format:
```
Co-Authored-By: Claude <noreply@anthropic.com>
```

This is added as a git trailer to commit messages when attribution is enabled and undercover mode is not active.
