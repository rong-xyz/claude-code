---
title: MCP Integration
layout: default
nav_order: 14
parent: Agentic Design Overview
has_children: true
---

# MCP Integration：双重角色架构

MCP（Model Context Protocol）是 Anthropic 制定的开放协议，允许 AI 工具与外部系统通过标准化接口交互。Claude Code 在 MCP 生态中扮演**双重角色**：

- **作为 MCP Server**：暴露自身能力，让 IDE 插件、Claude Desktop 等客户端接入并控制 Claude Code
- **作为 MCP Client**：连接外部 MCP 服务器，把远端工具当作原生工具使用

---

## 为什么需要双重角色？

```
┌─────────────────────────────────────────────────────┐
│                    MCP 生态系统                       │
│                                                       │
│  IDE Plugin ──MCP──▶ Claude Code ◀──MCP── Database  │
│  (VS Code)   Server    (中心)   Client    Server     │
│                           │                           │
│                     自身 Agent Loop                   │
│                     工具执行、session 管理             │
└─────────────────────────────────────────────────────┘
```

- **Server 角色**：让用户从 IDE 或 claude.ai 控制 Claude Code，无需在终端直接操作
- **Client 角色**：让 Claude Code 使用任意外部工具（数据库查询、代码搜索、知识库），而不需要内置这些工具

这两个角色**相互独立**：一个 session 可以同时作为 server 接受来自 VS Code 的请求，并作为 client 调用外部的 PostgreSQL MCP 服务器。

---

## 关键概念

### MCP Server（对外暴露能力）

```
Claude Code 启动时：
  ↓ entrypoints/mcp.ts 启动 MCP 服务器
  ↓ 监听 stdio 或 HTTP
  ↓ 等待 MCP 客户端连接

VS Code 插件连接：
  ↓ 读取 .mcp.json 找到 Claude Code 的地址
  ↓ 发送 MCP Initialize 请求
  ↓ 获取 Claude Code 暴露的 tools 列表
  ↓ 调用 tools（"query", "abort", "getStatus"）
```

### MCP Client（使用外部工具）

```
用户配置外部 MCP 服务器：
  ↓ 在 .mcp.json 中注册 "postgres-mcp-server"
  ↓ Claude Code 在 bootstrap 时连接

Agent 运行时：
  ↓ mcpClients 列出可用的远端工具
  ↓ "query_postgres" 出现在工具列表
  ↓ Agent 调用 → MCPTool 代理转发
  ↓ 结果返回 agent
```

---

## 关键文件

| 文件 | 角色 | 用途 |
|------|------|------|
| `entrypoints/mcp.ts` | Server | Claude Code 作为 MCP 服务器的入口 |
| `services/mcp/` | Client | MCP 客户端连接管理 |
| `tools/MCPTool.ts` | Client | 把远端 MCP 工具包装成原生 Tool |
| `tools/ListMcpResourcesTool.ts` | Client | 列出远端 MCP 资源（文件、数据库） |
| `tools/ToolSearchTool.ts` | Client | MCPSearch 模式：按需发现 MCP 工具 |
| `.mcp.json` | Config | 服务器注册和客户端发现配置 |

---

## 导航

- **[Claude Code 作为 MCP Server](mcp-server.md)** — 如何暴露能力给 IDE 和 claude.ai
- **[Claude Code 作为 MCP Client](mcp-client.md)** — 如何连接和使用外部 MCP 服务器

---

**下一步** → [MCP Server](mcp-server.md)
