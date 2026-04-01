---
title: Claude Code 作为 MCP Client
layout: default
nav_order: 2
parent: MCP Integration
grand_parent: Agentic Design Overview
---

# Claude Code 作为 MCP Client

Claude Code 可以连接外部 MCP 服务器，将远端工具无缝集成到 agent 的工具列表中。对 agent 来说，外部 MCP 工具和内置工具没有区别——同样的调用接口，同样的权限检查。

---

## 架构：mcpClients 上下文字段

每次 agent 调用工具时，`ToolUseContext` 包含一个 `mcpClients` 字段：

```typescript
interface ToolUseContext {
  readFileState: FileStateCache
  getAppState: () => AppState
  setToolJSX: (jsx: ReactNode) => void
  mcpClients: MCPClientRegistry   // ← MCP 客户端注册表
}
```

`mcpClients` 由 `services/mcp/` 在 bootstrap 时初始化，连接所有在 `.mcp.json` 中注册的外部服务器。工具执行时通过 `mcpClients` 发起实际的远端调用。

---

## 连接外部 MCP 服务器

用户在 `.mcp.json` 中注册外部 MCP 服务器：

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost/mydb"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    }
  }
}
```

**Bootstrap 时的连接流程**：

```
bootstrap/state.ts 启动
  ↓ 读取 .mcp.json
  ↓ 对每个 mcpServers 条目：
      启动子进程（command + args）
      等待 MCP Initialize 握手
      获取该服务器暴露的 tools 列表
      注册到 mcpClients
  ↓ 所有连接建立完成
  ↓ 合并到全局工具列表（如果 < 10% context window）
```

---

## MCPTool：远端工具的包装器

每个远端 MCP 工具被包装成一个 `MCPTool` 实例，实现标准的 `Tool` 接口：

```typescript
class MCPTool implements Tool<ZodSchema> {
  name: string          // 格式："mcp__<服务器名>__<工具名>"
  description: string   // 从远端服务器获取的描述
  inputSchema: ZodSchema // 从远端服务器获取的参数 schema

  async *call(input, ctx): AsyncGenerator<ToolResult> {
    // 1. 通过 ctx.mcpClients 找到对应的服务器连接
    // 2. 发送 JSON-RPC tools/call 请求
    // 3. 流式返回结果
  }
}
```

**命名约定**：远端工具名称前缀为 `mcp__<服务器名>__`，避免与内置工具冲突：

```
内置工具：  BashTool, FileReadTool, FileEditTool
MCP 工具：  mcp__postgres__query_db
            mcp__github__search_code
            mcp__filesystem__read_file
```

Agent 在工具列表中看到这些工具，可以直接调用，就像调用内置工具一样。

---

## ListMcpResourcesTool

除了 tools，MCP 服务器还可以暴露 **resources**（只读数据对象，如文件、数据库表、API 响应）。`ListMcpResourcesTool` 让 agent 发现这些 resources：

```
Agent 调用：ListMcpResourcesTool { server: "postgres" }
  ↓ 向 postgres MCP 服务器发送 resources/list 请求
  ↓ 返回：
      [
        { uri: "postgres://mydb/users", mimeType: "application/json" },
        { uri: "postgres://mydb/orders", mimeType: "application/json" }
      ]
  ↓ Agent 可以通过 MCPTool 读取具体 resource
```

Resources 与 Tools 的区别：
- **Resources**：只读，描述数据（文件内容、数据库表结构）
- **Tools**：有副作用，执行操作（运行查询、修改文件）

---

## MCPSearch 模式：当工具太多时

MCP 服务器可能暴露数十甚至数百个工具（如连接了大型代码库索引服务）。如果把所有工具描述放入 system prompt，会占用 context window 的巨大比例。

**触发条件**：MCP 工具描述总字数 > context window 的 10%

```
正常模式（< 10% context）：
  所有 MCP 工具描述放入 system prompt
  Agent 直接看到并调用

MCPSearch 模式（> 10% context）：
  system prompt 只告知 agent：
    "有 N 个 MCP 工具可用，使用 ToolSearchTool 搜索"
  Agent 按需搜索：
    ToolSearchTool { query: "database query" }
      → 返回匹配的工具列表
      → 这些工具激活，可调用
```

这与内置工具的[延迟发现机制](../tool-system/deferred-tools.md)完全相同——MCPSearch 是这一机制在 MCP 域的应用。

---

## 动态刷新机制

MCP 服务器的工具列表可以在 session 运行期间**动态变化**（例如用户安装了新插件，远端服务器重启）。

```
定期轮询（每 30 秒）：
  ↓ 向所有连接的 MCP 服务器发送 tools/list 请求
  ↓ 对比当前工具列表
  ↓ 如有变化：
      新增工具 → 注册到 mcpClients，加入工具列表
      消失工具 → 从工具列表移除（不影响正在进行的调用）
```

**为什么需要动态刷新？** MCP 服务器可能是长时间运行的进程（如本地文件系统服务器），其暴露的工具取决于当前目录、权限等外部状态。不重启 session 就能感知变化，是重要的用户体验保证。

---

## 权限：MCP 工具的安全性

MCP 工具同样受到 Claude Code 权限系统的管控：

```
Agent 决定调用 mcp__postgres__execute_sql
  ↓ 权限系统判断：
      isSafe? → false（有副作用）
      规则匹配？→ 无匹配规则
      YOLO 分类器？→ 检测到 DROP TABLE → 高风险
  ↓ 弹出权限提示：
      "Claude 想要执行 SQL: DROP TABLE users
       允许 / 拒绝"
```

MCP 工具的权限分类由 MCPTool 包装时从远端服务器的工具描述中推断（有无副作用声明），与内置工具的权限体系完全兼容。

---

## Design Decision：为什么不把外部工具"内置"？

| 方案 | 优势 | 劣势 |
|------|------|------|
| **MCP Client（当前）** | 任意外部工具即插即用；生态共享；无需 Claude Code 发版 | 网络依赖，调试复杂 |
| 内置所有工具 | 零延迟；完全掌控 | 工具爆炸（每个数据库、API 都需要内置工具）；维护负担 |
| 插件系统（专有） | 掌控生态 | 碎片化；开发者需要学习私有 API |

**MCP 作为通用接口**：任何团队都可以将自己的内部系统（私有代码库搜索、企业数据库、CI 系统）通过 MCP 协议接入 Claude Code，无需 Anthropic 介入。这是 Claude Code 企业化的核心扩展点。

---

## 深入阅读

### 相关文档

- [Claude Code 作为 MCP Server](mcp-server.md) — 对外暴露能力
- [Deferred Tool Discovery](../tool-system/deferred-tools.md) — 与 MCPSearch 相同的延迟发现模式
- [MCP Tools 集成](../tool-system/mcp-tools.md) — MCPTool 和 ListMcpResourcesTool 的详细实现
- [Permission System](../permission-system/) — MCP 工具的权限检查

### 源代码

- `services/mcp/` — MCP 客户端连接管理
- `tools/MCPTool.ts` — 远端工具包装器
- `tools/ListMcpResourcesTool.ts` — 资源列表工具
- `tools/ToolSearchTool.ts` — MCPSearch 模式实现
- `bootstrap/state.ts` — bootstrap 时的 MCP 连接初始化

---

**下一步** → [Feature Gating](../advanced/feature-gating.md)
