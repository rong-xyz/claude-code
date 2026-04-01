---
title: Claude Code 作为 MCP Server
layout: default
nav_order: 1
parent: MCP 集成
grand_parent: Agentic Design Overview
---

# Claude Code 作为 MCP Server

Claude Code 可以作为 MCP 服务器运行，将自身的能力（发起 query、获取状态、取消任务）暴露给任何 MCP 客户端——包括 VS Code 插件、JetBrains 插件和 claude.ai WebUI。

---

## 入口：`entrypoints/mcp.ts`

当用户以 MCP server 模式启动 Claude Code 时，`entrypoints/mcp.ts` 负责：

1. 初始化 MCP 服务器（基于 `@modelcontextprotocol/sdk`）
2. 注册暴露给外部的 tools 和 resources
3. 监听连接（stdio 模式或 HTTP/SSE 模式）
4. 将 MCP 请求路由到内部的 session 和 query 逻辑

```
Claude Code 以 server 模式启动：
  claude --mcp-server
  ↓
entrypoints/mcp.ts 运行：
  ↓ 创建 McpServer 实例
  ↓ 注册 tools: ["query", "abort", "getMessages", "getStatus"]
  ↓ 监听 stdin/stdout（stdio 传输层）
  ↓
VS Code 插件连接：
  ↓ MCP Initialize 握手
  ↓ 列出可用 tools
  ↓ 用户触发 → VS Code 调用 "query" tool
  ↓ Claude Code 执行 query
  ↓ 流式输出通过 MCP 协议返回给 VS Code
```

---

## 暴露的 MCP Tools

Claude Code 作为 server 时，向外部客户端暴露以下 tools：

| Tool 名称 | 用途 |
|-----------|------|
| `query` | 发送一个新的用户消息，触发 Claude 处理 |
| `abort` | 中止当前正在运行的 query（等价于 Ctrl+C） |
| `getMessages` | 获取当前 session 的消息历史 |
| `getStatus` | 获取 agent 当前状态（idle/running/waiting） |

这些工具让 IDE 插件可以**完全控制** Claude Code 的行为——就像用户在终端输入一样，但通过程序化接口。

---

## `.mcp.json` 配置

MCP 客户端通过 `.mcp.json` 文件发现和连接 Claude Code：

```json
{
  "mcpServers": {
    "claude-code": {
      "command": "claude",
      "args": ["--mcp-server"],
      "env": {
        "ANTHROPIC_API_KEY": "${env:ANTHROPIC_API_KEY}"
      }
    }
  }
}
```

**配置字段说明**：

| 字段 | 说明 |
|------|------|
| `command` | 启动 Claude Code 的命令（通常是 `claude`） |
| `args` | 传递 `--mcp-server` 标志以启用 MCP server 模式 |
| `env` | 传入所需的环境变量（如 API key） |

`.mcp.json` 可以放在：
- 项目根目录（项目范围配置）
- `~/.claude/.mcp.json`（全局配置）

---

## 传输层：stdio vs HTTP

Claude Code MCP server 支持两种传输层：

### stdio（默认）

```
IDE 插件 → 子进程启动 claude --mcp-server
         → stdin/stdout 作为传输通道
         → JSON-RPC 消息通过 stdio 交换
```

- **优点**：简单，无需网络配置，进程生命周期绑定
- **缺点**：只支持单一客户端连接

### HTTP/SSE（远程模式）

```
远端客户端 → HTTP POST /mcp/messages
           → SSE 流式响应 /mcp/events
```

- **优点**：支持多客户端，可远程连接（配合 bridge-remote）
- **缺点**：需要网络，需要认证（结合 JWT）

---

## Session 生命周期

每个 MCP 客户端连接都映射到一个独立的 Claude Code session：

```
MCP 客户端连接
  ↓ Claude Code 创建新 session（或恢复已有 session）
  ↓ session 绑定到连接的生命周期
  ↓
客户端调用 "query" tool
  ↓ 进入 agent query loop（与终端模式完全相同）
  ↓ 流式响应通过 MCP protocol 返回
  ↓
客户端断开
  ↓ session 进入 SUSPENDED 状态（保留历史）
  ↓ 重连时可恢复
```

这意味着：VS Code 插件打开的 session 和终端打开的 session **行为完全一致**——同样的 query loop、权限检查、context 管理。

---

## IDE 集成：VS Code 和 JetBrains

Claude Code 的 MCP server 模式是 IDE 插件的基础：

```
VS Code 插件：
  - 在插件激活时读取 .mcp.json
  - 启动 claude --mcp-server 作为语言服务器
  - 将编辑器选中代码作为上下文注入 query
  - 通过 getStatus 显示 Claude 工作状态（状态栏）

JetBrains 插件：
  - 同样流程，通过 IntelliJ Platform API 集成
  - 支持 NDJSON 输出格式（cli/print.ts 的结构化输出）
```

---

## Design Decision：为什么选择 MCP 而不是私有 API？

| 方案 | 优势 | 劣势 |
|------|------|------|
| **MCP（当前）** | 开放标准，社区生态，IDE 插件无需维护私有协议 | 协议版本管理复杂 |
| 私有 REST API | 完全掌控，更简单 | 每个 IDE 需要单独适配，锁定在 Anthropic 生态 |
| WebSocket 专有协议 | 低延迟，双向 | 同样的锁定问题 |

选择 MCP 的核心原因：**MCP 已经成为 AI 工具集成的事实标准**。使用 MCP 意味着 Claude Code 自动兼容所有支持 MCP 的编辑器和工具链，无需专门适配。

---

## 深入阅读

### 相关文档

- [Claude Code 作为 MCP Client](mcp-client.md) — 使用外部 MCP 服务器
- [Session State](../session-state/) — MCP 连接如何映射到 session 生命周期
- [Bridge & Remote](../bridge-remote/) — MCP over HTTP 的认证机制

### 源代码

- `entrypoints/mcp.ts` — MCP 服务器入口
- `services/mcp/mcpServer.ts` — McpServer 实现
- `remote/SessionsWebSocket.ts` — HTTP/SSE 传输层（与 bridge-remote 共享）

---

**下一步** → [Claude Code 作为 MCP Client](mcp-client.md)
