# 归属系统（Attribution System）

> **源文件**: `src/utils/attribution.ts`（394 行）

## 概述

归属系统为 commit 和 pull request 生成署名/归属文本，并提供"增强"格式，包含 AI 贡献的详细统计信息。

## 增强版 PR 归属

格式示例：
```
🤖 Generated with Claude Code (93% 3-shotted by claude-opus-4-5, 2 memories recalled)
```

### 组成部分

| 组件 | 来源 | 说明 |
|------|------|------|
| `claudePercent` | 计算得出 | AI 贡献百分比 |
| `promptCount` | `getTranscriptStats()` | 用户 prompt 数量（compact boundary 之后） |
| `memoryAccessCount` | `getTranscriptStats()` | Memory 系统访问次数 |
| 模型名称 | 经过清洗处理 | 未知模型回退为硬编码的 "Claude Opus 4.6" |

### N-Shot 术语

"N-shotted" 标签统计用户 prompt 次数：
- 0-shot：AI 在零额外 prompt 的情况下完成
- 1-shot：一次用户 prompt
- N-shot：需要 N 次用户 prompt

这提供了关于所需人工引导程度的透明度。

## 模型名称清洗

未知或内部模型名称**绝不会**出现在归属信息中：
- 已知模型：使用其公开名称（如 "claude-opus-4-6"）
- 未知模型：回退为硬编码的 "Claude Opus 4.6"

这可以防止内部代号（Capybara、Fennec、Numbat）通过 commit message 或 PR 描述泄露。

## Undercover 集成

当 `isUndercover()` 返回 `true` 时：
- 所有归属函数返回**空字符串**
- 不添加 commit 标记
- 不生成 PR 页脚
- 完全抑制 AI 归属信息

## 对话统计

`getTranscriptStats()` 分析对话内容：
- 统计 **compact boundary 之后**的用户 prompt 数量（并非总数）
- 统计 memory 访问事件
- 为增强版归属计算提供数据

## Commit 归属

标准格式：
```
Co-Authored-By: Claude <noreply@anthropic.com>
```

当归属功能启用且 Undercover Mode 未激活时，此标记会作为 git trailer 添加到 commit message 中。
