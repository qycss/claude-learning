# Tool 系统

## 概述

Claude Code 的 Tool 系统围绕 `buildTool()` 工厂函数构建，该函数基于 Zod Schema 创建标准化的 Tool 对象。约 40 个 Tool 各自位于 `src/tools/` 下的独立子目录中，保持一致的结构。

## Tool 目录结构

```
src/tools/
├── BashTool/
│   ├── BashTool.ts        # 实现
│   ├── UI.tsx             # 基于 Ink 的进度/输出渲染
│   ├── prompt.ts          # 面向 LLM 的 Tool 描述
│   └── bashSecurity.ts    # 2592 行的安全校验器
├── FileReadTool/
│   ├── FileReadTool.ts
│   ├── UI.tsx
│   └── prompt.ts
├── AgentTool/
│   ├── AgentTool.ts
│   ├── UI.tsx
│   ├── prompt.ts
│   └── forkSubagent.ts    # Fork 缓存共享
├── ... (共约 40 个)
```

## buildTool() 工厂函数（`src/Tool.ts`）

一个极其简洁的 4 行工厂函数：

```typescript
export function buildTool<T extends z.ZodType>(config: ToolConfig<T>): Tool<T> {
  return { ...config }
}
```

每个 Tool 配置包含：
- `name` -- Tool 标识符
- `description` -- 面向 LLM 的描述（来自 `prompt.ts`）
- `inputSchema` -- 用于输入校验的 Zod Schema
- `call(input, context)` -- 执行函数
- `isAllowed?(input, context)` -- 逐 Tool 的权限检查
- `beforeHook?(input, context)` -- 执行前 Hook
- `afterHook?(result, context)` -- 执行后 Hook
- `UI` -- 用于渲染的 React/Ink 组件

## 核心 Tool

### BashTool
- 带安全校验的 Shell 命令执行
- `bashSecurity.ts` -- 2,592 行命令分析
- 对已知危险命令进行模式匹配
- 参见 [Bash 安全机制](../security/bash-security.md)

### FileReadTool / FileWriteTool / FileEditTool
- 带路径安全检查的文件系统操作
- CWD 容器化检查
- FileEdit 使用精确字符串匹配进行替换

### AgentTool
- 用于复杂任务的子 Agent 生成
- `forkSubagent.ts` -- 带 LLM 缓存共享的 Fork（Copy-on-Write）
- 参见 [多 Agent 系统](multi-agent.md)

### GrepTool
- 对 ripgrep（`rg`）的封装
- 支持正则表达式、Glob 过滤、上下文行数

### GlobTool
- 文件模式匹配
- 按修改时间排序返回结果

### MCPTool
- 将 Tool 调用代理到 MCP 服务器
- 基于 MCP 服务器能力的动态 Tool 注册

### LSPTool
- Language Server Protocol 集成
- 代码智能（悬停提示、跳转定义、查找引用）

## StreamingToolExecutor

`src/services/tools/StreamingToolExecutor.ts` 在模型流式输出期间管理 Tool 的并发执行：

```
读取类 Tool（Grep, Glob, FileRead）：     并发执行
写入类 Tool（FileWrite, FileEdit, Bash）： 串行执行
```

读写锁语义防止文件的并发修改，同时允许并行读取。

## 权限集成

每次 Tool 调用都会经过权限系统：
1. 检查权限规则（allow/deny/ask）
2. 检查 Tool 自身的 `isAllowed()` 方法
3. 在 auto 模式下，发送至 YOLO 分类器
4. 在 default 模式下，提示用户确认

`SAFE_YOLO_ALLOWLISTED_TOOLS` 中的 Tool（25 个以上）在 auto 模式下跳过分类器。

## Tool 注册

Tool 在 `setup.ts` 初始化期间注册：
- 全部 40 个 Tool 连同其 Zod Schema 一起注册
- MCP Tool 从已连接的服务器动态添加
- Plugin Tool 从已启用的 Plugin 加载
- 每轮对话都会检查 Tool 可用性（部分 Tool 依赖上下文）
