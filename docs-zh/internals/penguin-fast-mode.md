# Penguin Mode（快速模式）

> **源文件**: `src/utils/fastMode.ts`（533 行）

## 简介

**Penguin Mode** 是 Claude Code 快速模式（Fast Mode）的内部代号。它**并不会**切换到其他模型——使用的仍然是同一个 Claude Opus 4.6，只是通过更快的输出通道提供服务。

## 核心参数

| 组件 | 值 |
|------|------|
| API 端点 | `/api/claude_code_penguin_mode` |
| GrowthBook 紧急开关 | `tengu_penguins_off` |
| 模型字符串 | `'opus' + (isOpus1mMergeEnabled() ? '[1m]' : '')` |
| 持久化缓存 | 全局配置中的 `penguinModeOrgEnabled` |
| 最小预取间隔 | 30 秒（防止 API 请求过载） |

## 状态机

```
FastModeRuntimeState: 'active' | 'cooldown'
```

冷却期使用信号模式：
```typescript
createSignal<[resetAt: number, reason: CooldownReason]>()
```

### CooldownReason
- `'rate_limit'` — 触发速率限制
- `'overloaded'` — API 过载

## 组织级别控制

```typescript
FastModeOrgStatus: 'pending' | 'enabled' | 'disabled'
```

### 禁用原因

| 原因 | 说明 |
|------|------|
| `'free'` | 免费套餐账户 |
| `'preference'` | 组织偏好设置已禁用 |
| `'extra_usage_disabled'` | 未启用额外用量 |
| `'network_error'` | 无法访问状态端点 |
| `'unknown'` | 兜底值 |

## 特殊行为

- **Anthropic 员工**：默认为 `enabled`（`resolveFastModeStatusFromCache`）
- **超额处理**：`handleFastModeOverageRejection()` 处理 7 种以上的超额场景
- **预取缓存**：结果缓存到磁盘（`penguinModeOrgEnabled`），最小重新获取间隔为 30 秒
- **切换方式**：用户可通过 `/fast` 斜杠命令切换

## API 集成

快速模式端点与标准 messages API 分离：
```
标准模式: POST /v1/messages
快速模式: POST /api/claude_code_penguin_mode
```

两者使用相同的模型，但快速端点通过专门优化的服务基础设施进行路由，以实现更低延迟和更快的 token 输出速度。
