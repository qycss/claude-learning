# Undercover Mode（隐匿模式）

> **源文件**: `src/utils/undercover.ts`（90 行）

## 功能说明

当 Anthropic 员工（`USER_TYPE === 'ant'`）向**公开仓库**提交代码时，Undercover Mode 会自动隐藏所有 AI 归属信息。系统会从所有输出中去除对 Claude Code、AI 辅助、内部代号和模型名称的引用。

## 工作原理

### 激活逻辑

```
isUndercover() = true UNLESS repo is in INTERNAL_MODEL_REPOS whitelist
```

**关键设计决策**：该模式**无法强制关闭**。源码中的注释写道：

> "if we're not confident we're in an internal repo, we stay undercover"
> （如果无法确认当前处于内部仓库，则保持隐匿状态）

这意味着系统强烈偏好误报（在内部仓库中保持隐匿）而非漏报（在公开仓库中泄露 AI 归属信息）。

### 核心函数

| 函数 | 用途 |
|------|------|
| `isUndercover()` | 除非处于白名单内部仓库，否则返回 true |
| `getUndercoverInstructions()` | 返回禁止提及 AI/Claude/代号的 LLM prompt |
| `shouldShowUndercoverAutoNotice()` | 一次性的说明对话框 |

### Prompt 指令

激活后，LLM 会收到以下指令：
- 不得提及 Claude Code 或 AI 辅助
- 不得引用内部代号
- 不得包含归属标记
- commit message 和 PR 描述应如同人类编写

## 外部构建行为

所有 undercover 代码都通过 `process.env.USER_TYPE === 'ant'` 进行门控。在外部构建中，Bun 的死代码消除机制会完全移除这些代码——外部用户永远不会遇到或触发 Undercover Mode。

## 集成点

- **归属系统**（`attribution.ts`）：当 `isUndercover() === true` 时，所有归属函数返回空字符串
- **Commit message**：不添加 `Co-Authored-By: Claude` 标记
- **PR 描述**：不生成 "Generated with Claude Code" 页脚
- **System prompt**：将 undercover 指令注入上下文

## 存在意义

Anthropic 员工经常向公司以外的开源项目贡献代码。如果没有 Undercover Mode，他们的贡献会暴露 AI 工具的使用情况，可能导致：
- 引发代码所有权方面的疑问
- 泄露内部工具能力
- 影响代码审查判断（正面或负面均有可能）
