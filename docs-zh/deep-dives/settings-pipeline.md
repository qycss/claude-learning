# 设置管线

## 概述

设置系统是一个 **7 来源分层合并管线**：Plugin base → userSettings → projectSettings → localSettings → flagSettings → policySettings。所有来源通过 Zod v4 验证，使用 lodash `mergeWith` 合并（数组去重拼接，`undefined` 表示删除）。企业策略通过 MDM 配置文件和远程管理服务器下发。

## 文件清单

| 文件路径 | 行数 | 职责 |
|-----------|-------|---------------|
| `src/utils/settings/settings.ts` | 1015 | 7 来源加载、合并、更新 |
| `src/utils/settings/types.ts` | 1148 | SettingsSchema（Zod v4），700+ 行 |
| `src/utils/settings/changeDetector.ts` | 488 | chokidar 文件监听 + MDM 轮询 |
| `src/utils/settings/settingsCache.ts` | 80 | 三级缓存 |
| `src/utils/settings/validation.ts` | 265 | Zod 错误格式化 + 规则验证 |
| `src/utils/settings/mdm/settings.ts` | 316 | MDM：macOS plist / Windows HKLM/HKCU |
| `src/utils/settings/mdm/rawRead.ts` | 130 | 零依赖子进程原始读取 |
| `src/services/analytics/growthbook.ts` | 1155 | GrowthBook remoteEval + 磁盘缓存 |

## 7 来源合并

```
Plugin base（最低优先级）
  → userSettings (~/.claude/settings.json)
    → projectSettings (.claude/settings.json)
      → localSettings (.claude/settings.local.json)
        → flagSettings (SDK/CLI 标志)
          → policySettings（最高优先级）
```

### 策略层"首来源优先"

policySettings 有自己的内部优先级（`settings.ts:677-738`）：
```
remote > HKLM/plist > managed-settings.json + drop-in 目录 > HKCU
```
第一个有内容的来源生效 — 策略来源之间不做合并，避免冲突。

## MDM 集成

### 并行加载
`startMdmSettingsLoad()` 在 CLI 最早阶段触发（`cli.tsx`）。`rawRead.ts` 不引入任何重依赖 — 可以在模块求值阶段运行：

```
main.tsx 求值
  ├── rawRead.ts 启动子进程（macOS: defaults read，Windows: reg query）
  ├── 重模块加载中...
  └── 当 settings.ts 需要时，MDM 结果已就绪
```

### Drop-in 目录
`managed-settings.d/` 采用 systemd/sudoers 约定：
- 多个团队可以独立部署策略片段
- 文件按名称排序决定覆盖优先级（`10-otel.json`、`20-security.json`）

## 变更检测 (`changeDetector.ts`)

- **chokidar** 监听所有设置文件路径
- `awaitWriteFinish`：稳定阈值 1000ms，轮询间隔 500ms
- 删除事件：`DELETION_GRACE_MS` 宽限期（处理删除后重建的模式）
- MDM：独立的 30 分钟轮询（注册表/plist 没有文件系统事件）
- 内部写入：`markInternalWrite` + 5 秒 `INTERNAL_WRITE_WINDOW_MS` 抑制窗口

## GrowthBook 功能标志

### Remote Eval 模式
`remoteEval: true` — 服务端预计算所有功能标志。客户端缓存到 Map + 磁盘。

### 三层覆盖
```
环境变量 (CLAUDE_INTERNAL_FC_OVERRIDES)
  > 配置覆盖（仅限 Anthropic 内部的 /config Gates）
    > remote eval 返回值
      > 磁盘缓存（离线回退）
```

### 磁盘缓存
`cachedGrowthBookFeatures` 写入全局配置，保证离线可用，采用 stale-while-revalidate 语义。

## Zod Schema (`types.ts`)

700+ 行涵盖：
- 权限规则（allow/deny/ask）
- Hooks（12+ 种 Hook 类型及匹配器）
- 沙箱配置（网络/文件系统）
- MCP 服务器白名单/黑名单
- 模型覆盖
- 环境变量
- 归因设置

使用 `lazySchema()` 延迟构造，避免循环引用。`z.passthrough()` 保留未知字段以保持向后兼容。

## 独特发现

1. **导入循环叶模块** (`syncCacheState.ts`)：从 `syncCache.ts` 中提取的纯状态叶模块，用于打破 `settings → syncCache → auth → settings` 循环依赖。注释中解释了强连通分量问题和时序约束。
2. **`insideGetConfig` 反递归标志** (`config.ts:51`)：防止 `getConfig → logEvent → getGlobalConfig → getConfig` 的无限循环。
3. **`CUSTOMIZATION_SURFACES`** (`types.ts:248-253`)：`strictPluginOnlyCustomization` 将 skills/agents/hooks/mcp 等定制面限制为仅允许插件修改 — 用于企业标准化管控。
4. **三级缓存** (`settingsCache.ts`)：parseFile → perSource → session 合并结果，外加 `pluginSettingsBase` 作为插件默认值。
