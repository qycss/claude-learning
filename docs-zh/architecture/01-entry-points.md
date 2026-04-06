# 入口与启动

## 四个入口点

### 1. CLI 入口 (`src/main.tsx`)

主入口。三阶段并行预取架构：

```
阶段 1（立即执行，在重型导入之前）：
  ├── MDM 设置预取（rawRead.ts — 零重型导入）
  ├── Keychain 预取（keychainPrefetch.ts — 隔离导入）
  └── API 预连接（TCP 连接预热）

阶段 2（Commander.js 定义）：
  ├── 解析 CLI 参数
  ├── 注册子命令
  └── 验证 flags

阶段 3（Ink 渲染器）：
  ├── 初始化 React + Ink
  ├── 挂载 App 组件
  └── 启动 REPL 循环
```

关键洞察：`rawRead.ts` 和 `keychainPrefetch.ts` 被刻意设计为零重型导入。Bun 的 `__esm` 包装器在访问任何 symbol 时评估整个模块图 — 导入 execa 的 `human-signals/cross-spawn` 链需要 ~58ms 同步初始化，这会抵消预取的并行化收益。

### 2. Leader/Coordinator CLI (`src/entrypoints/cli.tsx`)

位于 `main.tsx` 之上的编排层，管理 daemon 模式的 leader 选举和多会话桥接连接。

### 3. MCP 服务器 (`src/entrypoints/mcp.ts`)

以 MCP 服务器模式运行 Claude Code，将工具暴露为 MCP 工具定义。

### 4. Agent SDK (`src/entrypoints/sdk/`)

无头模式（无终端 UI），直接消息传递 API，供外部应用嵌入 Claude Code。

## 启动序列

```
1. main.tsx 评估
   ├── 触发并行预取（MDM、Keychain、预连接）
   ├── 导入 Commander.js（轻量级）
   └── 定义 CLI 程序

2. Commander.js 解析参数
   ├── 确定模式（交互 / MCP / bridge / 恢复）
   └── 提取 flags（模型、权限模式等）

3. setup.ts 运行
   ├── 初始化会话存储
   ├── 加载设置（7 源合并）
   ├── 解析 API 凭证
   ├── 初始化遥测
   └── 构建系统 prompt（context.ts）

4. Ink 渲染器挂载
   ├── App.tsx（providers: FpsMetrics, Stats, AppState）
   └── REPL.tsx（5005 行 — 主交互循环）

5. 首次查询循环
   ├── QueryEngine.ts（会话级）
   └── query.ts（轮次级）
```

## 并行预取细节

| 组件 | 策略 | 原因 |
|------|------|------|
| MDM 设置 | `rawRead.ts` 子进程 | 零导入，可在模块评估期运行 |
| Keychain | `keychainPrefetch.ts` 异步 | 与 execa 隔离避免 58ms 同步初始化 |
| API 预连接 | TCP connect | 在首次查询前预热连接池 |
| GrowthBook | 磁盘缓存 + 异步刷新 | 离线安全，stale-while-revalidate |
