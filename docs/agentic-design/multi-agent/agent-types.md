---
title: 四种 Agent 类型
layout: default
nav_order: 1
parent: Multi-Agent Coordination
grand_parent: Agentic Design Overview
---

# 四种 Agent 类型

## Local Agent（进程内）

```typescript
const subAgent = await spawnAgent({
  type: "local",
  agentDef: generalPurposeAgent,
  initialMessage: "帮我测试这个函数"
})
```

**特点**：
- 在当前 Node.js 进程中运行
- 共享内存（同一 V8 引擎）
- 无网络延迟，最快的交互
- 上下文隔离通过 `AsyncLocalStorage` 实现

**文件**：`tools/AgentTool/forkSubagent.ts`

---

## Remote Agent（CCR 云容器）

```typescript
const remoteAgent = await spawnAgent({
  type: "remote",
  agentDef: exploreAgent,
  initialMessage: "搜索 codebase 中的 bug"
})
```

**特点**：
- 在 Cloud Container Runtime（CCR）中运行
- 独立的资源池和网络隔离
- 支持长时间运行：最多 **30 分钟**（ULTRAPLAN）
- 网络传输延迟，但隔离度最高

---

## Forked Agent（进程派生）

```typescript
const forkedAgent = spawnForkedAgent({
  agentScript: "./my-agent.ts",
  args: ["--mode", "debug"]
})
```

**特点**：
- 创建新的 Node.js 子进程（`child_process.fork`）
- 完全隔离的 V8 引擎
- 支持 CPU 密集任务（不会阻塞主线程）
- 便于调试和独立监控

---

## In-Process Teammate（进程内同伴）

```typescript
const teammates = await createTeam({
  name: "research-squad",
  agents: [agent1, agent2, agent3],
  strategy: "parallel"
})
```

**特点**：
- 多个 agent **同时运行**在同一进程
- 完全隔离的逻辑邮箱（inter-agent 消息传递）
- 通过 `AsyncLocalStorage` 实现上下文隔离
- **无网络开销**，同步协调

**组件**：
- `InProcessTeammateTask`：Teammate 生命周期管理
- `leaderPermissionBridge`：权限审批同步
- `teammateMailbox`：异步消息队列
- `teammateLayoutManager`：终端 UI 布局协调

---

## 对比表

| 维度 | Local | Remote | Forked | Teammate |
|------|-------|--------|--------|----------|
| 部署位置 | 进程内 | 云容器 | 子进程 | 进程内 |
| 启动延迟 | 极低 | 高 | 中 | 极低 |
| 最长运行 | session 限制 | 30 分钟 | session 限制 | session 限制 |
| 隔离度 | 低 | 极高 | 极高 | 中 |
| 网络开销 | 无 | 有 | 无 | 无 |
| 内存共享 | 是 | 否 | 否 | 否 |
