# MCP 集成

## 概述

Claude Code 实现了全面的 MCP (Model Context Protocol) 集成，支持 **8 种传输类型**和**7 层配置作用域**系统，并具备企业级独占模式。

## 架构

```
用户/项目配置 → MCP 配置解析（7 层作用域） → 传输类型选择 → 建立连接
                                                   ↓
                                         Tool/Resource/Prompt 注册
                                                   ↓
                                         MCPTool 代理调用至各服务器
```

## 8 种传输类型

| 传输类型 | 使用场景 |
|----------|----------|
| `stdio` | 本地进程（最常用） |
| `sse` | Server-Sent Events（旧版协议） |
| `streamable-http` | 现代 HTTP 流式传输 |
| `node` | Node.js 子进程 |
| `npx` | npm 包执行 |
| `uvx` | Python uv 工具执行 |
| `docker` | 基于容器的服务器 |
| `url` | 直接 URL 连接 |

## 7 层配置作用域 (`src/services/mcp/config.ts`)

配置来源及优先级：

```
enterprise（最高）
  → managed
    → user
      → project
        → local
          → plugin
            → flag（最低）
```

Enterprise 作用域支持**独占模式** —— 启用后，低优先级作用域中对应服务器的配置将被忽略。这确保了企业强制指定的 MCP 服务器不会被个人用户覆盖。

## 身份认证

### OAuth 2.0
- 为需要认证的 MCP 服务器提供完整的 OAuth 流程
- Token 存储在 keychain 中（与 Claude Code 主 OAuth 共享）

### XAA（跨 Agent 认证）
- 服务器到服务器的认证协议
- 用于 MCP 服务器之间的相互认证

## 连接管理

- 延迟连接 —— 服务器在首次工具调用时才建立连接
- 对高频使用的服务器进行连接池化
- 带退避策略的自动重连
- 健康检查心跳机制

## 工具注册

MCP 服务器暴露的工具会被动态注册：
```
服务器连接 → 服务器发送工具列表 → 工具注册为 MCPTool 实例
                                         ↓
                                   在 LLM tool_use 中可用
```

工具名称以 `mcp__` 为前缀，以区分内置工具。

## 安全性

- MCP 工具名称在 analytics 中脱敏处理（`mcp__*` → `mcp_tool`），防止 PII 泄露
- Worker JWT 通过闭包传递，而非 `process.env`，防止 MCP 服务器读取会话 Token
- 企业独占模式可防止影子 MCP 服务器配置
