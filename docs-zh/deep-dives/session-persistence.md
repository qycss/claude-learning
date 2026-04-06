# 会话持久化

## 概述

Claude Code 的会话持久化基于 **JSONL 追加写入日志**和 **parentUuid 链式结构**构建。每个会话对应 `~/.claude/projects/{sanitized_project_path}/{sessionId}.jsonl` 文件。消息通过 UUID 父指针形成链表，支持分支（sub-agent 的侧链）和合并。

## 文件清单

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| `src/utils/sessionStorage.ts` | 5000+ | 核心持久化引擎 —— 读写/链重建 |
| `src/utils/sessionStoragePortable.ts` | 280 | 可移植层（与 VS Code 扩展共享） |
| `src/utils/conversationRecovery.ts` | 600 | 反序列化 + 会话恢复 |
| `src/utils/sessionRestore.ts` | 550 | 状态恢复编排 |
| `src/utils/messages.ts` | 2200+ | 消息归一化（`normalizeMessagesForAPI`） |
| `src/types/logs.ts` | 331 | 21 种 Entry 类型（discriminated union） |

## 消息格式

每条 `TranscriptMessage`（`types/logs.ts:221-231`）：

```typescript
type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null        // 指向前一条消息的链指针
  logicalParentUuid?: UUID       // compact 后保留的逻辑父节点
  isSidechain: boolean           // Sub-agent 分支标记
  agentId?: string               // Sub-agent 标识符
  promptId?: string              // OTel 关联 ID
}
```

## 21 种 Entry 类型

`Entry` 类型是一个 discriminated union，涵盖：`transcript`、`summary`、`custom-title`、`tag`、`agent-name`、`pr-link`、`file-history-snapshot`、`attribution-snapshot`、`content-replacement`、`context-collapse-commit`，以及另外 11 种。

## 写入路径 —— 批量队列

`sessionStorage.ts:532-686` 实现了一个带有批量写入的 `Project` 类：

1. `enqueueWrite(filePath, entry)` —— 推入按文件分组的队列
2. `scheduleDrain()` —— 每 100ms 触发一次
3. `drainWriteQueue()` —— 序列化为 JSONL，以 100MB 分块追加写入
4. `appendToFile()` —— 对 NFS 非标准错误码自动执行 `mkdir` 回退

## 链重建

`buildConversationChain()`（`sessionStorage.ts:2069-2093`）：

```
1. 找到最新的叶子消息
2. 沿 parentUuid 链回溯至根节点
3. 反转得到时间顺序
4. 执行 recoverOrphanedParallelToolResults() 后处理
```

`recoverOrphanedParallelToolResults` 后处理（`sessionStorage.ts:2096-2206`）用于处理并行工具执行产生的 DAG 结构。当 N 个并行 tool_use 块生成 N 条 assistant 消息（相同 message.id，不同 uuid）时，单链遍历只能保留一个分支。后处理通过 message.id 分组 + parentUuid 反向索引恢复孤立节点。

## 64KB 恢复边界

`LITE_READ_BUF_SIZE = 65536`（`sessionStoragePortable.ts:17`）。`readHeadAndTail()` 读取文件的前后各 64KB 用于轻量级元数据（title、tag、agentName），避免全文件读取。

`reAppendSessionMetadata`（`sessionStorage.ts:450-465`）通过在写入后重新追加元数据，确保元数据始终保持在尾部 64KB 窗口内。

## 中断检测

`detectTurnInterruption()`（`conversationRecovery.ts:164-333`）：

| 类型 | 条件 | 处理方式 |
|------|------|----------|
| `none` | 最后一条消息来自 assistant | 正常 |
| `interrupted_prompt` | 用户输入已提交但无响应 | 恢复 |
| `interrupted_turn` | 工具执行被中断 | 自动注入 "Continue from where you left off." |

## 性能优化

- **50MB OOM 保护**（`sessionStorage.ts:229`）：`MAX_TRANSCRIPT_READ_BYTES = 50MB`。超过此阈值的 Tombstone 重写操作将被拒绝（修复 inc-3930）。
- **Pre-compact 跳过**（`sessionStorage.ts:3536-3579`）：超过 `SKIP_PRECOMPACT_THRESHOLD` 的文件使用流式读取，跳过 compact 边界之前的消息。将内存占用从文件大小降低到输出大小（例如 151MB 文件 → 约 32MB 分配）。
- **walkChainBeforeParse**（`sessionStorage.ts:3572-3579`）：对于 compact 边界后大于 5MB 的缓冲区，在 JSON 解析之前执行字节级链遍历，预先裁剪死分支。

## 特殊发现

1. **progressBridge 向后兼容**（`sessionStorage.ts:3623-3641`）：PR #24099 从链中移除了 progress 消息。加载时构建 `progressBridge: Map<UUID, UUID|null>` 以跳过 progress 节点，而无需修改磁盘文件。
2. **并发写入安全**：写入使用文件锁（`lockfile.lock()`）。`getHistory()` 优先读取当前会话条目，防止跨会话历史交错。
3. **Stale-while-error 撤销历史**（`history.ts:442-464`）：双层策略 —— 快速路径从 `pendingEntries` 弹出；如果已刷入磁盘（竞态失败），则将时间戳添加到 `skipSet` 以供后续过滤。
