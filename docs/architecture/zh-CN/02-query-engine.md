# 查询引擎

## 双层架构

查询系统采用双层 AsyncGenerator 协程设计：

```
QueryEngine（会话级）
  └── query()（轮次级）
        └── StreamingToolExecutor（并发工具）
```

### 第一层：QueryEngine (`src/QueryEngine.ts`)

会话级 AsyncGenerator：
- 跨轮次维护对话历史
- 管理系统 prompt（静态 + 动态部分）
- 通过 AbortController 级联处理会话级中止
- 跟踪 token 使用量和预算
- 通过 `withRetry.ts` 协调重试逻辑

### 第二层：query() (`src/query.ts`)

轮次级循环，处理单次"用户输入 → 助手响应"周期：

```
while (true) {
  response = await streamAPICall(messages)

  if (response 包含 tool_use blocks) {
    results = await executeTools(response.tool_use)
    messages.push(assistant_message, tool_results)
    continue  // 循环回 API
  }

  break  // 无更多工具调用，轮次完成
}
```

## 流式架构

API 调用使用 SSE（Server-Sent Events）流式传输：

```
API 流 → chunks → StreamingToolExecutor
                      ├── 文本 chunks → yield ProgressMessage
                      ├── 工具调用开始 → 开始执行工具
                      ├── 工具调用结束 → 收集结果
                      └── Stop reason → 结束轮次
```

### StreamingToolExecutor

并发工具执行，读写锁语义：
- **读工具**（Grep、Glob、FileRead）：并发执行
- **写工具**（FileWrite、FileEdit、Bash）：串行执行

## 重试引擎 (`src/services/api/withRetry.ts`)

629 行重试引擎：

| 特性 | 细节 |
|------|------|
| 策略 | 指数退避 + 抖动 |
| 最大重试 | 可配置（默认 3） |
| 时钟偏移检测 | 比较服务器 `Date` 头与本地时间 |
| 速率限制 | 遵循 `Retry-After` 头 |
| 过载检测 | 529 状态码 → 延长退避 |
| 中止集成 | 重试间检查 AbortController |

## AbortController 级联

三层中止层次：

```
会话 AbortController
  └── 兄弟 AbortController（每轮次）
        └── 每工具 AbortController（每次工具调用）
```

- 会话中止 → 取消所有轮次和工具
- 兄弟中止 → 取消当前轮次但不取消会话
- 每工具中止 → 仅取消一个工具执行
- `WeakRef` 用于每工具控制器的 GC 安全
