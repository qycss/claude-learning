# OAuth 与凭证管理

## 概述

Claude Code 实现了完整的 OAuth 2.0 Authorization Code + PKCE 流程，支持双模式授权码获取（自动 localhost 回调 + 手动复制粘贴）。凭证存储采用分层架构：macOS Keychain（通过 `security` CLI）为首选，回退到 `~/.claude/.credentials.json` 明文存储。Token 刷新使用文件锁 + 三重检查模式，防止跨进程竞争。

## 文件清单

| 文件路径 | 行数 | 职责 |
|-----------|-------|---------------|
| `src/services/oauth/index.ts` | 199 | OAuthService — PKCE 流程编排 |
| `src/services/oauth/client.ts` | 567 | OAuth 客户端 — URL 构建、Token 交换/刷新 |
| `src/services/oauth/crypto.ts` | 24 | PKCE 加密 — code_verifier、code_challenge、state |
| `src/services/oauth/auth-code-listener.ts` | 212 | 用于 OAuth 回调的 localhost HTTP 服务器 |
| `src/utils/auth.ts` | 1900+ | 认证核心 — Token 读取/刷新/保存/缓存 |
| `src/utils/secureStorage/macOsKeychainStorage.ts` | 232 | macOS Keychain 实现 |
| `src/utils/secureStorage/macOsKeychainHelpers.ts` | 112 | Keychain 辅助工具 — 缓存、TTL、generation 计数器 |
| `src/utils/secureStorage/fallbackStorage.ts` | 71 | 回退组合（主存储 -> 备用存储） |

## PKCE 流程

```
1. 生成 code_verifier = base64url(randomBytes(32))
2. 生成 code_challenge = base64url(SHA-256(code_verifier))
3. 生成 state = base64url(randomBytes(32))  // 防止 CSRF
4. 构建包含 code_challenge_method=S256 的授权 URL
5. 双通道并行：
   ├── 自动模式：openBrowser(url) → localhost:PORT/callback
   └── 手动模式：显示 URL 供用户复制粘贴
6. Promise.race 等待任一授权码返回
7. 通过 exchangeCodeForTokens() 将授权码换取 Token
```

## 三重检查 Token 刷新 (`auth.ts:1427-1562`)

```
检查 1：缓存中的 Token 是否过期？
   ↓ 是
检查 2：清除所有缓存，异步重新读取。仍然过期？
   ↓ 是
检查 3：获取文件锁，在锁内重新读取。仍然过期？
   ↓ 是
执行 refreshOAuthToken()
```

锁竞争使用随机退避策略（`1000 + Math.random() * 1000` 毫秒），最多重试 5 次。

## 401 请求中去重 (`auth.ts:1338-1343`)

```typescript
const pending401Handlers = new Map<string, Promise<boolean>>()
```

当 N 个代理连接器同时收到 401 响应（启动时常见 — #20930），按 `staleAccessToken` 去重：只有第一个请求清除缓存并读取 Keychain，其他请求加入同一个 Promise。

## Keychain 存储（macOS）

### 读取
```
security find-generic-password -a "${username}" -w -s "${storageServiceName}"
```

### 写入
1. JSON 转十六进制编码（`Buffer.from(json).toString('hex')`）
2. 优先使用 `security -i` stdin 方式（避免 CrowdStrike 等进程监控看到载荷内容 — INC-3028）
3. 如果命令超过 4032 字节（`4096 - 64`），回退到 argv 方式

### 4096 字节 Stdin 限制

macOS `security -i` 使用 `fgets()` 且 `BUFSIZ=4096`。超出此限制时：
- 前 4096 字节作为一条命令解析（引号未闭合导致失败）
- 溢出部分被解析为下一条未知命令
- 结果：返回非零退出码，但旧的 Keychain 条目被保留
- 预留 64 字节用于应对行终止符差异

### Stale-While-Error 缓存

30 秒 TTL（`KEYCHAIN_CACHE_TTL_MS = 30_000`）。读取失败但缓存中已有数据时，返回过期值而非 `null` — 防止"未登录"状态传播（#23192）。

### Generation 计数器

`keychainCacheState.generation` 在每次 `clearKeychainCache()` 时递增。`readAsync()` 在子进程启动前捕获 generation 值，如果执行过程中 generation 发生变化（说明有 update() 写入了新数据），则丢弃结果。

## 回退存储 (`fallbackStorage.ts`)

```
createFallbackStorage(macOsKeychainStorage, plaintextStorage)
```

- 读取：先尝试主存储，若为 null 则尝试备用存储
- 写入：主存储失败则写入备用存储；若备用存储写入成功且主存储有过期数据，则删除主存储条目
- 首次主存储写入成功后，删除备用存储中的数据（迁移清理）

## 独特发现

1. **刷新时的作用域扩展** (`client.ts:152-163`)：刷新时显式传递 `CLAUDE_AI_OAUTH_SCOPES`，利用服务端的 `ALLOWED_SCOPE_EXPANSIONS` 在不重新登录的情况下扩展作用域（例如添加 `user:file_upload`）。
2. **每日 700 万请求优化** (`client.ts:189-199`)：当全局配置和安全存储中已有完整的 profile 数据时，跳过 `/api/oauth/profile` 调用。注释中提到此优化在全集群范围每天节省约 700 万次请求。
3. **Keychain 预取的导入隔离** (`macOsKeychainHelpers.ts:1-15`)：该模块不能导入 execa — Bun 的 `__esm` 包装器在访问任何符号时会执行整个模块，而 execa 的 `human-signals/cross-spawn` 依赖链会消耗约 58ms 的同步初始化时间。
4. **isMacOsKeychainLocked 生命期缓存** (`macOsKeychainStorage.ts:210-231`)：`security show-keychain-info` 退出码 36 表示已锁定（SSH 环境中常见）。缓存整个进程生命周期，防止重复创建子进程导致的 27ms/消息渲染延迟。
5. **跨进程磁盘变更检测** (`auth.ts:1320-1336`)：在每次 Token 操作前检查 `.credentials.json` 的修改时间戳 — 修复了当另一个 Claude Code 实例执行 `/login` 后 memoize 缓存过期的问题（CC-1096, GH#24317）。
