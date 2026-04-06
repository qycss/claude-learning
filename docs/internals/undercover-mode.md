# Undercover Mode

> **Source**: `src/utils/undercover.ts` (90 lines)

## What It Does

When Anthropic employees (`USER_TYPE === 'ant'`) contribute code to **public repositories**, Undercover Mode automatically hides all AI attribution. The system strips mentions of Claude Code, AI assistance, internal codenames, and model names from all outputs.

## How It Works

### Activation Logic

```
isUndercover() = true UNLESS repo is in INTERNAL_MODEL_REPOS whitelist
```

**Critical design decision**: There is **no force-OFF**. The comment in source reads:

> "if we're not confident we're in an internal repo, we stay undercover"

This means false positives (staying undercover in an internal repo) are strongly preferred over false negatives (leaking AI attribution in a public repo).

### Key Functions

| Function | Purpose |
|----------|---------|
| `isUndercover()` | Returns true unless in allowlisted internal repo |
| `getUndercoverInstructions()` | Returns LLM prompt forbidding AI/Claude/codename mentions |
| `shouldShowUndercoverAutoNotice()` | One-time explainer dialog |

### Prompt Instructions

When active, the LLM receives instructions to:
- Never mention Claude Code or AI assistance
- Never reference internal codenames
- Never include attribution markers
- Write commit messages and PR descriptions as if a human wrote them

## External Build Behavior

All undercover code is gated on `process.env.USER_TYPE === 'ant'`. In external builds, Bun's dead-code elimination removes this code entirely — external users never encounter or can trigger undercover mode.

## Integration Points

- **Attribution system** (`attribution.ts`): All attribution functions return empty strings when `isUndercover() === true`
- **Commit messages**: No `Co-Authored-By: Claude` trailer
- **PR descriptions**: No "Generated with Claude Code" footer
- **System prompt**: Undercover instructions injected into context

## Why This Exists

Anthropic employees regularly contribute to open-source projects outside the company. Without undercover mode, their contributions would reveal AI tool usage, potentially:
- Creating questions about code ownership
- Revealing internal tool capabilities
- Biasing code review (positively or negatively)
