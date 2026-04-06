# 状态管理

## Zustand Store (`src/state/AppState.tsx`)

中央状态容器，Zustand store，约 1,200 行：

```typescript
// 简化结构
const useAppState = create<AppState>((set, get) => ({
  // 对话状态
  messages: [],
  conversationId: null,
  // 会话状态
  sessionId: null,
  isLoading: false,
  permissionMode: 'default',
  // 工具状态
  activeTools: [],
  pendingPermissions: [],
  // UI 状态
  scrollPosition: 0,
  searchQuery: '',
  // 设置
  settings: mergedSettings,
  // ~100+ 个 actions
}))
```

## Store 架构 (`src/state/store.ts`)

精简的 34 行 Store 抽象：
- 包装 Zustand 的 `create()`
- 添加选择器 memoization
- 通过订阅提供变更追踪

## 消息类型

| 类型 | 描述 |
|------|------|
| `UserMessage` | 用户输入（文本 + 附件） |
| `AssistantMessage` | 模型响应（文本 + tool_use blocks） |
| `SystemMessage` | 系统 prompt 部分 |
| `ToolUseBlock` | 工具调用（名称 + 输入） |
| `ToolResultBlock` | 工具执行结果 |
| `ProgressMessage` | 流式进度更新 |

## 持久化

### 1. 会话存储 (JSONL)
- `src/utils/sessionStorage.ts`（~5,000 行）
- 追加式 JSONL 日志
- parentUuid 链式结构
- 批量写入队列（100ms 排空间隔）

### 2. 全局配置
- `src/utils/config.ts`（~500 行）
- `insideGetConfig` 防递归守卫，防止 `getConfig → logEvent → getGlobalConfig → getConfig` 无限循环

## 变更追踪

- 基于订阅的变更追踪
- React Compiler 集成（`_c()` memo cache）减少无效重渲染
- `OffscreenFreeze` 防止长会话（2800+ 条消息）中的脏标志级联传播
