# Contributing to Claude Learning

Thank you for your interest in contributing to this research project!

## How to Contribute

### Adding New Analysis

1. Fork the repository
2. Create a feature branch: `git checkout -b analysis/your-topic`
3. Add your analysis to the appropriate directory:
   - `docs/architecture/` - System-level architectural analysis
   - `docs/deep-dives/` - Subsystem deep dives
   - `docs/security/` - Security mechanism analysis
   - `docs/internals/` - Internal/undocumented features
4. Follow the existing document format (see template below)
5. Submit a Pull Request

### Document Template

```markdown
# [Subsystem Name]

## Overview
Brief description of what this subsystem does and why it matters.

## File Inventory
| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/...` | ~N | Description |

## Architecture
Key architectural decisions and patterns.

## Implementation Details
Source-level analysis with `file:line` references.

## Unique Findings
Discoveries not obvious from documentation or surface-level reading.
```

### Improving Diagrams

- Diagrams use [Mermaid](https://mermaid.js.org/) syntax
- Source files are in `docs/diagrams/`
- GitHub renders Mermaid natively in markdown

### Translation

- English docs live at their standard path
- Chinese translations use `.zh-CN.md` suffix
- Keep translations in sync with the source

## Code of Conduct

- This is an educational research project
- Analysis should be factual and evidence-based
- Always cite source file paths and line numbers
- Do not include or redistribute original source code

## Questions?

Open an issue for discussion.
