---
title: AsyncLocalStorage 上下文隔离
layout: default
nav_order: 2
parent: Multi-Agent Coordination
grand_parent: Agentic Design Overview
---

# AsyncLocalStorage 上下文隔离

## 为什么需要隔离？

子 agent 的全局状态不应该污染父 agent：

```typescript
// 主 agent 的全局状态
const sessionId = "parent-session-id"
const permissions = "default"
const memoryDir = "~/.claude/memory/"

// 在子 agent 中，这些应该被**覆盖**
asyncLocalStorage.run({
  sessionId: "child-agent-id",
  permissions: "bubble",        // 特殊权限给 worker
  memoryDir: "/tmp/child-memory",
  parentSessionId: "parent-session-id"  // 追踪父子关系
}, () => {
  runChildAgent()  // 子 agent 看到的是覆盖后的值
})
```

## AsyncLocalStorage 是什么？

`AsyncLocalStorage` 是 Node.js 19+ 内置的上下文隔离机制，支持 async/await 边界的值传播。

特点：
- 每个 async 执行上下文有独立的存储
- 不会跨越 async 边界泄露
- 支持 `run()` 创建新的隔离作用域
- 完全线程安全（在 async 层面）

## Agent Spawning 流程

```typescript
// Claude 调用 AgentTool
Agent: "让我 spawn 一个 explore agent 来搜索代码库"

Tool Call:
  name: "AgentTool"
  input: {
    type: "spawn",
    agentDef: "exploreAgent",
    prompt: "搜索 utils/ 下的性能问题",
    isolationMode: "local"  // 或 "remote"
  }

返回:
  agentId: "abc123"
  status: "running"
```

## 隔离的值

子 agent 中被覆盖的值：

- `sessionId`：新的 session ID（防止混淆）
- `permissions`：可能提升或降低（根据父决策）
- `memoryDir`：独立的内存目录
- `parentSessionId`：追踪来源
- `contextDepth`：当前嵌套深度（防止无限递归）
- `readFileCache`：独立的文件读取缓存

## 深入阅读

- `tools/AgentTool/forkSubagent.ts`：Fork 实现
- `utils/asyncLocalStorage.ts`：AsyncLocalStorage 初始化
