# 模型代号（Model Codenames）

## 概述

Claude Code 为模型和产品本身使用内部代号。这些代号贯穿整个源代码，出现在 feature flag、API 端点和配置中。

## 已知代号映射

| 代号 | 对应对象 | 依据 |
|------|----------|------|
| **Tengu**（天狗） | Claude Code（产品本身） | 所有 Datadog 事件以 `tengu_` 为前缀，GrowthBook flag 以 `tengu_` 为前缀 |
| **Capybara** | Claude Sonnet | 在模型路由代码中被引用 |
| **Fennec** | Opus 4.6 之前的模型 | 在模型历史记录中被引用 |
| **Numbat** | 下一个未发布的模型 | 在前瞻性代码中被引用 |
| **Penguin** | Fast Mode 功能 | API 端点：`/api/claude_code_penguin_mode` |

## 代号使用模式

### Tengu — 产品代号

在代码库中广泛使用：
- **Datadog 事件**：`tengu_session_start`、`tengu_query_complete` 等
- **GrowthBook flag**：`tengu_auto_mode_config`、`tengu_penguins_off`、`tengu_frond_boric`
- **API 路径**：各种内部 API 端点

### Penguin — Fast Mode

- **API 端点**：`/api/claude_code_penguin_mode`
- **紧急开关**：`tengu_penguins_off`
- **缓存键**：`penguinModeOrgEnabled`

## 代号泄露防护

归属系统（`attribution.ts`）明确防范代号泄露：

```
未知/内部模型 → 硬编码为 "Claude Opus 4.6"
```

这确保即使正在测试新的内部模型（如 Numbat），归属文本也永远不会暴露代号。只有已知的、已公开发布的模型名称才会出现在面向用户的输出中。

## Feature Flag 混淆

详见 [Feature Flag 混淆](feature-flag-obfuscation.md)，了解代号如何在 feature flag 命名中被进一步模糊化。
