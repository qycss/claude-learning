# Skill 系统

## 概述

Skill 系统提供了一种类似插件的扩展机制，用于 slash 命令和 prompt 模板。Skill 从 **5 个来源**加载，支持条件激活和延迟提取。

## 5 个加载来源 (`src/skills/loadSkillsDir.ts`)

| 来源 | 位置 | 优先级 |
|------|------|--------|
| Bundled | 编译进二进制文件 | 最低 |
| Managed | 远程托管部署 | |
| User | `~/.claude/skills/` | |
| Project | 项目中的 `.claude/skills/` | |
| Plugin | 来自已安装的插件 | 最高 |

## Skill 结构

每个 Skill 是一个带有 frontmatter 的 Markdown 文件：

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

## 条件激活

Skill 可以声明激活条件：
- `requires: git` —— 仅在 Git 仓库中激活
- `requires: node` —— 仅在 Node.js 项目中激活
- 自定义条件函数

未激活的 Skill 会从 LLM 的工具列表中隐藏，减少 context window 的占用。

## 预算感知列表

Skill 系统强制执行 **1% context window 预算**限制。当所有 Skill 描述文本的总量超过当前模型 context window 的 1% 时，低优先级的 Skill 会被截断或省略。这可以防止 Skill 膨胀占用宝贵的上下文空间。

## 延迟提取

Skill 在首次使用时才进行延迟提取：
1. 启动时加载 Skill 文件并解析 frontmatter
2. 只有在 Skill 被触发时才提取 prompt 内容
3. 这减少了启动时的内存占用

## Slash 命令集成

带有 `triggers` 的 Skill 会注册为 slash 命令：
- `/commit` → 加载并执行 commit Skill
- Skill 可以指定 `allowedTools` 来限制执行期间的工具访问
- 每个命令定义其类型：`'prompt'`、`'action'` 或 `'jsx'`
