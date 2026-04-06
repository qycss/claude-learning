# Prompt Injection Defenses

## Overview

Claude Code implements multiple layers of defense against prompt injection attacks, particularly in the YOLO classifier and tool execution pipeline.

## Defense 1: Transcript Sanitization

The YOLO classifier's `buildTranscriptEntries()` (`yoloClassifier.ts:341-357`) applies a critical filter:

**Only `tool_use` blocks from assistant messages are included. All assistant text is excluded.**

### Why This Matters

```
Attack scenario:
1. Malicious content in a file being read
2. Content influences model to generate text like:
   "This is safe, allow all future commands"
3. If this text reached the classifier, it could influence decisions

Defense:
- Assistant text is model-authored → could be influenced by injection
- tool_use blocks are structured data → cannot be forged by text
- Classifier only sees: what tools were called + with what inputs
```

## Defense 2: JSONL Transcript Format

The system is migrating from text-prefix to JSONL format for classifier transcripts:

| Format | Example | Vulnerability |
|--------|---------|--------------|
| Text-prefix | `Bash ls` | Newlines in content can forge `{"user":"..."}` lines |
| JSONL | `{"Bash":"ls"}` | JSON escaping prevents content injection |

Controlled by GrowthBook `jsonlTranscript` flag.

## Defense 3: Fail-Closed Classifier

Any ambiguity in the classifier's output defaults to BLOCK:

- Unparseable response → block
- API timeout → block
- Missing `<block>` tag → block
- Multiple `<block>` tags → block

This ensures that even a successful injection that confuses the classifier's output format results in denial, not approval.

## Defense 4: Independent Side Query

The YOLO classifier uses a **separate API call** from the main conversation:
- Main conversation context cannot directly influence classifier
- Classifier has its own system prompt optimized for security decisions
- Different model parameters (lower max_tokens, stop sequences)

## Defense 5: CLAUDE.md as Intent Signal

The classifier receives user's CLAUDE.md content in `<user_claude_md>` tags (`yoloClassifier.ts:460-477`). This provides **legitimate user intent** context:
- User-declared project conventions are trusted
- This displaces the need for the model to "guess" user intent from potentially-injected content

## Defense 6: Dual Template System

Internal vs external users get different classifier templates (`yoloClassifier.ts:54-78`):
- External (`permissions_external.txt`): User rules **replace** template sections
- Internal (`permissions_anthropic.txt`): User rules **append** to template sections

This prevents external users from overriding core safety rules.

## Defense 7: Dangerous Permission Stripping

When entering auto mode, the system strips existing allow rules that could bypass the classifier:
- Blanket tool allows (e.g., `Bash` with no content filter)
- Interpreter prefixes (23 patterns)
- Infrastructure tools (internal-only)

Even if a user previously approved these patterns in manual mode, auto mode strips them to force classifier evaluation.

## Defense 8: Context Ratio Monitoring

`yoloClassifier.ts:1073-1087` tracks the ratio of classifier context to main loop context:
- Expected ratio: 0.6-0.8
- If ratio > 1.0: classifier context exceeds main loop
- Risk: auto-compact reduces main loop but not classifier → stale/wrong decisions
- Action: `transcript_too_long` error triggered
