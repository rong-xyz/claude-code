---
title: MCP Tools 集成
layout: default
nav_order: 4
parent: Tool System
grand_parent: Agentic Design Overview
---

# MCP Tools 集成

Claude Code 通过三个工具将 MCP（Model Context Protocol）生态无缝接入原生工具系统：`MCPTool`、`ListMcpResourcesTool` 和 `ToolSearchTool`（MCPSearch 模式）。

---

## MCPTool：远端工具的代理

`MCPTool` 把外部 MCP 服务器暴露的工具**包装成原生 Tool**，让 agent 无感知地调用远端能力：

```typescript
class MCPTool implements Tool<ZodSchema> {
  // 命名格式：mcp__<服务器名>__<工具名>
  name = "mcp__postgres__query"

  // 从远端服务器拉取的描述（原汁原味）
  description = "Execute a SQL query against the PostgreSQL database"

  // 从远端服务器拉取的参数 schema（自动转换为 Zod）
  inputSchema = z.object({
    query: z.string().describe("SQL query to execute"),
    params: z.array(z.unknown()).optional()
  })

  async *call(input, ctx): AsyncGenerator<ToolResult> {
    // 1. 从 ctx.mcpClients 取出对应服务器的连接
    const client = ctx.mcpClients.get("postgres")

    // 2. 发送 MCP tools/call JSON-RPC 请求
    const result = await client.callTool({
      name: "query",
      arguments: input
    })

    // 3. 将 MCP 响应格式转换为 Claude Code 的 ToolResult
    yield { type: "text", text: JSON.stringify(result.content) }
  }
}
```

**命名约定**：`mcp__<服务器名>__<工具名>`

这个前缀确保：
- 不与内置工具名称冲突（如 `BashTool`、`FileReadTool`）
- Agent 在工具名中即可识别来源（便于 prompt 工程）
- 同一名称工具在不同 MCP 服务器间不冲突

---

## 工具注册：Bootstrap 时动态注入

`MCPTool` 实例在 bootstrap 时动态创建，而不是静态定义：

```
bootstrap/state.ts 启动
  ↓ 读取 .mcp.json，发现 3 个外部服务器
  ↓ 连接每个服务器
  ↓ 调用 tools/list 获取工具列表
  ↓ 对每个远端工具，创建对应的 MCPTool 实例
  ↓ 注入到 ALL_TOOLS 列表

最终工具列表（示例）：
  AgentTool           ← 内置
  BashTool            ← 内置
  FileReadTool        ← 内置
  mcp__github__search_code      ← MCPTool（动态）
  mcp__postgres__query          ← MCPTool（动态）
  mcp__filesystem__read_file    ← MCPTool（动态）
```

---

## ListMcpResourcesTool：列出 MCP Resources

MCP 协议区分 **Tools**（有副作用的操作）和 **Resources**（只读数据对象）。`ListMcpResourcesTool` 让 agent 发现后者：

```
Agent 调用：ListMcpResourcesTool { server: "postgres" }
  ↓ MCP resources/list 请求发往 postgres 服务器
  ↓ 服务器返回：
      [
        {
          uri: "postgres://mydb/users",
          name: "users table",
          mimeType: "application/json",
          description: "User accounts table"
        },
        {
          uri: "postgres://mydb/orders",
          name: "orders table",
          mimeType: "application/json"
        }
      ]
```

Agent 拿到 resources 列表后，可以通过 MCPTool 读取具体资源内容（MCP `resources/read` 请求）。

**Resources vs Tools 的关键区别**：

| 维度 | Resources | Tools |
|------|-----------|-------|
| 副作用 | 无（只读） | 可能有 |
| 权限检查 | 自动批准（isSafe = true） | 按规则/YOLO 分类器 |
| 典型场景 | 读数据库表结构、读文件 | 执行 SQL、修改文件 |
| 缓存 | 可缓存（内容不变则复用） | 不缓存 |

---

## ToolSearchTool：MCPSearch 模式

当 MCP 服务器提供的工具太多（描述文本 > context window 的 10%），将所有工具放入 system prompt 会挤压可用的 context 空间。此时切换到 **MCPSearch 模式**：

### 触发条件

```typescript
const MCP_TOOLS_THRESHOLD = 0.1 // 10% of context window

const mcpToolsTokens = estimateTokens(allMCPToolDescriptions)
const contextWindowSize = getModelContextWindow()

if (mcpToolsTokens / contextWindowSize > MCP_TOOLS_THRESHOLD) {
  // 切换到 MCPSearch 模式
  enableMCPSearchMode()
}
```

### 运作方式

```
Normal 模式（< 10% context）：
  System Prompt: "Available tools: [BashTool, FileReadTool, ...,
                  mcp__github__search_code, mcp__postgres__query, ...]"
  Agent 直接看到所有工具

MCPSearch 模式（> 10% context）：
  System Prompt: "You have 47 MCP tools available. Use ToolSearchTool
                  to discover them before use."
  Agent 按需搜索：
    ToolSearchTool { query: "SQL database query" }
    → 返回：["mcp__postgres__query", "mcp__postgres__execute"]
    → 这两个工具激活，可直接调用
```

### 与内置工具延迟发现的对比

`ToolSearchTool` 同样被内置工具（`shouldDefer: true` 的工具）使用，但有细微差别：

| 维度 | 内置 Deferred 工具 | MCPSearch 模式 |
|------|-------------------|---------------|
| 触发条件 | 工具标记了 `shouldDefer: true` | MCP 工具总描述 > 10% context |
| 搜索范围 | 只搜索 deferred 内置工具 | 只搜索 MCP 工具 |
| 激活方式 | 搜索后永久激活 | 搜索后本轮激活 |

详见：[延迟工具发现](deferred-tools.md)

---

## mcpClients：ToolUseContext 中的连接注册表

每次工具调用时，`ToolUseContext.mcpClients` 提供所有活跃 MCP 连接的访问：

```typescript
interface MCPClientRegistry {
  // 按服务器名获取连接
  get(serverName: string): MCPClient | undefined

  // 列出所有活跃连接
  listServers(): string[]

  // 检查连接健康状态
  isConnected(serverName: string): boolean
}
```

`MCPTool` 通过这个注册表找到目标服务器连接，发起 JSON-RPC 调用。如果连接中断，`MCPTool` 会：
1. 尝试重连（最多 3 次）
2. 重连失败 → 返回错误给 agent，触发 Error Handling 流程

---

## 深入阅读

### 相关文档

- [MCP Integration](../mcp/) — MCP 的 Server/Client 双重角色架构
- [Deferred Tool Discovery](deferred-tools.md) — 内置工具的延迟发现机制（与 MCPSearch 同源）
- [Tool Interface](tool-interface.md) — MCPTool 实现的 Tool 接口规范

### 源代码

- `tools/MCPTool.ts` — MCPTool 实现
- `tools/ListMcpResourcesTool.ts` — 资源列表工具
- `tools/ToolSearchTool.ts` — MCPSearch 模式实现
- `services/mcp/` — MCP 客户端连接管理
- `bootstrap/state.ts` — MCPTool 动态注入逻辑

---

**下一步** → [MCP Integration](../mcp/)
