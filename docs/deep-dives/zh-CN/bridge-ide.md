# Bridge 与 IDE 集成

## 概述

Bridge 系统提供 Claude Code 与 IDE 扩展（VS Code、JetBrains）以及 claude.ai Web 界面之间的双向通信。它有两代协议和一个"无环境变量"变体。

## 文件清单

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| `src/bridge/types.ts` | 263 | 类型定义（BridgeConfig、SessionHandle、SpawnMode） |
| `src/bridge/bridgeMain.ts` | 700+ | 独立 Bridge（多会话模式） |
| `src/bridge/replBridge.ts` | 800+ | REPL 内嵌 Bridge |
| `src/bridge/remoteBridgeCore.ts` | 1009 | 无环境变量 V2 Bridge |
| `src/bridge/replBridgeTransport.ts` | 371 | 传输抽象层（V1/V2 适配器） |
| `src/bridge/jwtUtils.ts` | 257 | JWT 解码 + 刷新调度器 |
| `src/bridge/flushGate.ts` | -- | 历史刷新期间的消息排队 |

## 协议对比

| 方面 | V1 | V2 |
|------|----|----|
| 读取 | WebSocket（HybridTransport） | SSE（通过 `from_sequence_num` 实现断点续传） |
| 写入 | HTTP POST 到 Session-Ingress | CCRClient（SerialBatchEventUploader） |
| 序列号 | 无（返回 0） | 完整的序列号追踪 |
| 送达确认 | 无 | `received` + `processed` ACK |
| 状态 | 无 | `reportState()`：idle/running/requires_action |
| 元数据 | 无 | `reportMetadata()` |

## 无环境变量 V2 —— 三步连接

最简路径，绕过 Environments API 生命周期：

```
1. POST /v1/code/sessions (OAuth) → session.id
2. POST /v1/code/sessions/{id}/bridge (OAuth) → worker_jwt + epoch
3. createV2ReplTransport(worker_jwt, epoch) → SSE + CCRClient
```

每次调用 `/bridge` 会递增 epoch —— 它本身**就是**注册动作。无需 register/poll/ack/deregister 生命周期。

## Echo 去重 —— 三层机制

写入的消息会通过 SSE 回显。三层去重：

| 层级 | 实现方式 | 用途 |
|------|----------|------|
| `recentPostedUUIDs` | BoundedUUIDSet（2000 上限环形缓冲区） | 实时消息 |
| `initialMessageUUIDs` | 无界 Set | 初始历史（可能超过 2000） |
| `recentInboundUUIDs` | BoundedUUIDSet | 防止重复接收 |

## FlushGate

确保 `[history..., realtime...]` 的有序传输：

```
flushGate.start()  → 开始排队
  writeMessages()  → flushGate.enqueue(msgs) → 缓冲
flushGate.end()    → 返回缓冲的消息，立即发送
```

## JWT 刷新

`jwtUtils.ts:72-256` 为每个会话创建主动刷新定时器：
- 在 `expiresIn - refreshBuffer`（最少 30 秒）时调度刷新
- Generation 计数器防止取消/重新调度后的过期回调
- 连续失败上限 `MAX_REFRESH_FAILURES = 3` 次后放弃
- 刷新成功后调度后续刷新（回退间隔 30 分钟）

## 崩溃恢复

**独立 Bridge**（`bridgeMain.ts:141+`）：
- `runBridgeLoop` 通过 `activeSessions` Map 维护超时看门狗
- 401 JWT 过期 → `reconnectSession` 将会话重新入队
- 睡眠/唤醒检测：如果轮询间隔超过 `connCapMs * 2`，则判定系统刚唤醒，重置退避

**REPL Bridge**（`remoteBridgeCore.ts:530-590`）：
- `recoverFromAuthFailure()`：刷新 OAuth → 获取新的 /bridge 凭证 → `rebuildTransport()`（新 epoch + JWT + SSE seq-num 续传）
- 主动刷新和 401 恢复通过 `authRecoveryInFlight` 标志互斥

## 特殊发现

1. **Processed 即时 ACK**（`replBridgeTransport.ts:249-252`）：同时发送 `received` 和 `processed`，防止 daemon 重启累积 21→25 个幻影 prompt。
2. **Per-instance 认证闭包**（`remoteBridgeCore.ts:233-234`）：Worker JWT 通过闭包传递，而非 `process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN`，因为 `mcp/client.ts` 会无条件读取该环境变量 —— 这会将 JWT 泄露给用户配置的 MCP 服务器。
3. **容量唤醒**（`bridgeMain.ts:194`）：当会话槽位已满时，轮询循环休眠直到 `capacityWake` 信号触发（会话完成），避免无意义的 API 轮询。
4. **SpawnMode**（`types.ts:69`）：`single-session | worktree | same-dir`。Worktree 模式为每个会话创建 Git worktree 隔离环境。
