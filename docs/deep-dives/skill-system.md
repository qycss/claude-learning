# Skill System

## Overview

The Skill system provides a plugin-like extension mechanism for slash commands and prompt templates. Skills are loaded from **5 sources** with conditional activation and lazy extraction.

## 5-Source Loading (`src/skills/loadSkillsDir.ts`)

| Source | Location | Priority |
|--------|----------|----------|
| Bundled | Compiled into binary | Lowest |
| Managed | Remote-managed deployment | |
| User | `~/.claude/skills/` | |
| Project | `.claude/skills/` in project | |
| Plugin | From installed plugins | Highest |

## Skill Structure

Each skill is a markdown file with frontmatter:

```markdown
---
name: commit
description: Create a git commit
triggers:
  - /commit
  - commit changes
tools:
  - Bash
  - FileRead
conditional:
  requires: git
---

[Skill prompt content here]
```

## Conditional Activation

Skills can declare activation conditions:
- `requires: git` — Only active in git repositories
- `requires: node` — Only active in Node.js projects
- Custom condition functions

Inactive skills are hidden from the LLM's tool list, reducing context window usage.

## Budget-Aware Listing

The skill system enforces a **1% context window budget** for skill listings. When the total skill description text exceeds 1% of the current model's context window, lower-priority skills are truncated or omitted. This prevents skill bloat from consuming valuable context space.

## Lazy Extraction

Skills are lazy-extracted at first use:
1. Skill file is loaded and frontmatter parsed during startup
2. Prompt content is only extracted when the skill is triggered
3. This reduces startup memory footprint

## Slash Command Integration

Skills registered with `triggers` become available as slash commands:
- `/commit` → loads and executes the commit skill
- Skills can specify `allowedTools` to restrict tool access during execution
- Each command defines its type: 'prompt', 'action', or 'jsx'
