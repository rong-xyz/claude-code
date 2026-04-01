---
title: 多 Agent 协调
layout: default
nav_order: 5
parent: Agentic Design Overview
has_children: true
---

# 多 Agent 协调：并行与分解

## 为什么需要多个 Agent？

有些任务太复杂，一个 agent 力不从心：

- 用户说："帮我把代码库从 TypeScript 迁到 Rust"
- 一个 agent 从第一个文件开始，改 5 个小时后，context 满了

更好的方式：**分解任务**

```
Coordinator agent 解析任务、制定计划
  ↓
Worker agent 1（并行）处理"转换 utils 模块"
Worker agent 2（并行）处理"转换 API 层"
Worker agent 3（并行）处理"转换 UI 组件"
  ↓
Coordinator 总结结果、验证一致性
```

这就是**多 agent 协调**，由 `COORDINATOR_MODE` 启用。

---

## Agent 类型：四种部署模式

Claude Code 支持多种 agent 部署方式，由 `AgentTool` 统一管理（`tools/AgentTool/AgentTool.tsx`，~233 KB）：

### 1. Local Agent（进程内）

特点：
- 在当前 Node.js 进程中运行
- 共享内存（同一 V8 引擎）
- 无网络延迟，最快的交互
- **上下文隔离**通过 `AsyncLocalStorage` 实现

文件：`tools/AgentTool/forkSubagent.ts`

### 2. Remote Agent（CCR 云容器）

特点：
- 在 Cloud Container Runtime（CCR）中运行
- 独立的资源池和网络隔离
- 支持长时间运行：最多 **30 分钟**（ULTRAPLAN）
- 网络传输延迟，但隔离度最高

### 3. Forked Agent（进程派生）

特点：
- 创建新的 Node.js 子进程（`child_process.fork`）
- 完全隔离的 V8 引擎
- 支持 CPU 密集任务（不会阻塞主线程）
- 便于调试和独立监控

### 4. In-Process Teammate（进程内同伴，代号 "amber_flint"）

特点：
- 多个 agent **同时运行**在同一进程
- 完全隔离的逻辑邮箱（inter-agent 消息传递）
- 通过 `AsyncLocalStorage` 实现上下文隔离
- **无网络开销**，同步协调

---

## 常见误解

**误解 1**："Coordinator 也会执行 tool？"

实际：Coordinator **只** spawn worker 和管理任务。所有的 tool 执行都在 worker 中进行。

**误解 2**："Worker 之间可以直接通信？"

实际：Worker 通过 **Coordinator 中转**，便于统一管理和决策。

**误解 3**："多个 agent 肯定比单个快？"

实际：取决于任务的可并行化程度。高度依赖的任务可能反而更慢。

---

## 关键要点

1. **四种 Agent 类型**：local（in-process）、remote（CCR 30min）、forked（子进程）、teammate（同进程同伴）
2. **AsyncLocalStorage 隔离**：子 agent 的全局状态完全独立
3. **Coordinator 模式**：4 个阶段（research → synthesis → implementation → verification）
4. **XML Protocol 通信**：Worker 通过 `<task-notification>` 报告
5. **Tengu Scratchpad**：大文件存储，不占 context window
6. **Team Memory**：共享发现和进展，file lock 防竞态

---

## 深入阅读

- [Agent 类型详解](agent-types.md)
- [AsyncLocalStorage 上下文隔离](context-isolation.md)
- [Coordinator 4 阶段模式](coordinator-mode.md)
- [任务状态机与 ID 格式](task-machine.md)
- [Team Memory 同步](team-memory.md)
