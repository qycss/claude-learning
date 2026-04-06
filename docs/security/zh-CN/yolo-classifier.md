# YOLO Classifier -- 两阶段安全守门机制

## 概述

YOLO Classifier 是 auto/bypassPermissions 模式下的安全守门组件。它采用 **两阶段分类器**（Fast + Thinking），通过独立的 LLM API 侧查询来评估工具调用的安全性。核心原则：**fail-closed** -- 任何解析失败、API 错误或超时均默认执行 BLOCK。

## 文件清单

| 文件路径 | 行数 | 职责 |
|-----------|-------|------|
| `src/utils/permissions/yoloClassifier.ts` | 1496 | 核心引擎（prompt、transcript、两阶段 XML/tool_use） |
| `src/utils/permissions/classifierDecision.ts` | 99 | 安全白名单工具 |
| `src/utils/permissions/classifierShared.ts` | 40 | 共享解析工具函数 |
| `src/utils/permissions/denialTracking.ts` | 46 | 拒绝跟踪与熔断器 |
| `src/utils/permissions/autoModeState.ts` | 40 | Auto mode 全局状态 |
| `src/utils/permissions/bypassPermissionsKillswitch.ts` | 156 | Bypass mode 远程终止开关 |
| `src/utils/permissions/dangerousPatterns.ts` | 81 | 危险 bash 命令模式匹配 |

## 两阶段架构

### 阶段 1 -- Fast（`yoloClassifier.ts:769-858`）

```
max_tokens: 64
stop_sequences: ['</block>']
suffix: "Err on the side of blocking. <block> immediately."
```

二元判定：`<block>yes</block>` 或 `<block>no</block>`
- allow --> **立即返回**（快速路径）
- block 或无法解析 --> 升级到阶段 2

### 阶段 2 -- Thinking（`yoloClassifier.ts:860-996`）

```
max_tokens: 4096
suffix: "Review the classification process... Use <thinking> before responding with <block>."
```

通过 `<thinking>` 标签进行思维链推理，降低误报率。两个阶段共享 **相同的 prompt 前缀**，利用 `cache_control` 使阶段 2 能命中阶段 1 的缓存。

### 三种运行模式

| 模式 | 行为 | 控制方式 |
|------|------|----------|
| `'both'` | 默认两阶段 | -- |
| `'fast'` | 仅阶段 1（max_tokens=256） | GrowthBook |
| `'thinking'` | 仅阶段 2 | GrowthBook |

由 `tengu_auto_mode_config.twoStageClassifier` 控制。

## Transcript 安全性

`buildTranscriptEntries()`（`yoloClassifier.ts:302-360`）：

**关键决策（第 341-357 行）**：assistant 消息中仅包含 `tool_use` blocks，所有 assistant 文本均被排除。

**原因**：assistant 文本由模型生成，可能通过 Prompt Injection 被操纵，从而影响分类器的决策。而 tool use blocks 是结构化数据，无法被模型的文本输出伪造。这一设计消除了整个攻击面。

## Fail-Closed 设计

| 场景 | 结果 |
|------|------|
| API 错误 | `shouldBlock: true` + `unavailable: true` |
| 解析失败 | `shouldBlock: true` |
| Abort 信号 | `shouldBlock: true` |
| 空输入 | `shouldBlock: false`（特殊情况：无安全相关输入） |

## 拒绝熔断器（`denialTracking.ts`）

```
maxConsecutive: 3   // 连续 3 次拒绝
maxTotal: 20        // 累计 20 次拒绝
```

`shouldFallbackToPrompting()` 在达到阈值时回退到交互式提示模式，防止分类器永久阻塞 agent。

## 危险权限剥离（`permissionSetup.ts:83-119`）

进入 auto mode 时，系统会检查现有的 allow 规则并剥离危险模式：

| 模式 | 风险 |
|------|------|
| 工具级 allow（`Bash` 无 ruleContent） | 允许所有命令 |
| 解释器前缀（`python:*`、`node:*`，共 23 种模式） | 任意代码执行 |
| 仅限内部的模式（`gh api`、`curl`、`kubectl`、`aws`） | 基础设施访问 |

## 远程终止开关

- `autoModeCircuitBroken`：GrowthBook `tengu_auto_mode_config.enabled === 'disabled'` 阻止 SDK/CLI 进入 auto mode
- `checkAndDisableBypassPermissionsIfNeeded()`：运行时禁用 bypassPermissions
- `verifyAutoModeGateAccess()`：启动时和模型切换时的准入检查，支持按模型的 auto mode 检查

## 安全白名单

`SAFE_YOLO_ALLOWLISTED_TOOLS`（25+ 工具）完全跳过分类器。值得注意的是：
- **FileWrite/FileEdit 不在白名单中** -- 它们有单独的 `acceptEdits` 快速路径（CWD 内：允许，CWD 外：走分类器）
- **TeamCreateTool 在白名单中** -- "队友有自己的权限检查"

## 独特发现

1. **双模板系统**（`yoloClassifier.ts:54-78`）：内部用户使用 `permissions_anthropic.txt`（规则追加），外部用户使用 `permissions_external.txt`（规则通过 `<user_*_to_replace>` 标签替换）
2. **JSONL transcript 迁移**：从文本前缀格式（`Bash ls`）迁移到 JSONL 格式（`{"Bash":"ls"}`）-- JSON 转义防止内容伪造 `{"user":"..."}` 行
3. **alwaysOnThinking 适配**（`yoloClassifier.ts:682-693`）：启用服务端自适应 thinking 的模型额外获得 2048 个 token，防止 thinking 耗尽预算导致 `<block>` 标签无法输出从而产生误阻
4. **CLAUDE.md 注入**（`yoloClassifier.ts:460-477`）：分类器将用户的 CLAUDE.md 作为 `<user_claude_md>` 上下文 -- "用户意图"信号
5. **PowerShell 映射**：`iex (iwr ...)` --> "来自外部的代码"，`Remove-Item -Recurse -Force` --> "不可逆的本地破坏"
6. **上下文比率追踪**（`yoloClassifier.ts:1073-1087`）：监控 `classifierTokensEst / mainLoopTokens` 比率（预期 0.6-0.8）。若 >1.0，auto-compact 无法保护分类器 --> 触发 `transcript_too_long` 错误
