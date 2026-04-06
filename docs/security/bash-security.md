# Bash Security

## Overview

`src/tools/BashTool/bashSecurity.ts` is a **2,592-line** security validator that analyzes every shell command before execution. It's the most complex single security file in the codebase.

## Key File

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/tools/BashTool/bashSecurity.ts` | 2592 | Command analysis, pattern matching, risk classification |

## Analysis Approach

The validator performs multi-pass analysis on each command:

### 1. Command Parsing
- Tokenizes the command into components
- Handles pipes, redirects, subshells, command substitution
- Resolves aliases and shell builtins

### 2. Pattern Matching

Commands are matched against categorized patterns:

| Category | Examples | Risk Level |
|----------|----------|------------|
| Destructive | `rm -rf`, `mkfs`, `dd` | Critical |
| Network | `curl \| sh`, `wget -O- \| bash` | Critical |
| Persistence | `crontab`, `systemctl enable` | High |
| Privilege escalation | `sudo`, `su`, `chmod +s` | High |
| Information disclosure | `env`, `printenv`, `cat /etc/shadow` | Medium |
| Package management | `npm install -g`, `pip install` | Medium |

### 3. Compound Command Analysis

Handles complex command structures:
- Pipes: `cat file | sh` (code execution via pipe)
- Redirects: `echo "malicious" > /etc/hosts`
- Command substitution: `` `curl evil.com` ``
- Background execution: `nohup malicious &`

### 4. Interpreter Detection

23 cross-platform interpreter patterns detected:
```
python, python3, node, ruby, perl, php, lua,
bash, sh, zsh, fish, dash, csh, tcsh, ksh,
powershell, pwsh, osascript, wscript, cscript,
deno, bun, tsx
```

Commands starting with interpreter prefixes are flagged because they can execute arbitrary code from strings or URLs.

## Risk Classification

Each analyzed command receives a risk classification:

| Level | Action |
|-------|--------|
| `safe` | Auto-allow |
| `needs_review` | Send to YOLO classifier (auto mode) or prompt user |
| `dangerous` | Always prompt user, even in auto mode |
| `blocked` | Deny regardless of mode |

## Integration with Permission System

```
User runs Bash command
  → bashSecurity.ts analyzes
    → Returns risk level
      → Permission system applies rules
        → YOLO classifier (if auto mode and needs_review)
          → Execute or deny
```

## Internal-Only Patterns

Anthropic-internal builds (ant-only) include additional patterns for infrastructure tools:
- `gh api` — GitHub API calls
- `curl` — arbitrary HTTP requests
- `kubectl` — Kubernetes cluster access
- `aws` — AWS CLI
- 8 more infrastructure patterns

These are stripped from permission rules when entering auto mode (`dangerousPatterns.ts`).
