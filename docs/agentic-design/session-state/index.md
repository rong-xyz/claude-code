---
title: Session State
layout: default
nav_order: 11
parent: Agentic Design Overview
has_children: true
---

# Session State：跨重启的持久化

一个 "session" 是用户和 Claude Code 之间的一次对话。Session 可以暂停、保存、以及从上次中断的地方恢复。

## Session 的生命周期

```
[1] COLD_START
    用户首次运行 claude-code
    ↓
[2] ACTIVE
    用户正在交互（输入命令、等待响应）
    ↓
[3] SUSPENDED
    用户按 Ctrl+C 或关闭终端
    ↓
[4] RESUMED
    用户再次运行 claude-code
    系统恢复之前的 session（如果存在）
```

## Session 数据的三个部分

### [1] Session Metadata（会话元数据）

```
{
  sessionId: "abc-123-def-456",
  createdAt: "2026-04-01T13:45:00Z",
  resumedAt: "2026-04-01T14:00:00Z",
  userId: "user@example.com",
  projectDir: "/home/user/claude-code",
  tags: ["feature/agent-loop"]
}
```

### [2] Message History（消息历史）

```
[
  { id: "msg-1", role: "user", content: "...", timestamp: ... },
  { id: "msg-2", role: "assistant", content: "...", timestamp: ... },
  ...
]
```

会话中发生的所有交互。

### [3] Memory State（记忆状态）

```
~/.claude/memory/
├── MEMORY.md          # 跨 session 的持久化知识
├── projects/
│   └── claude-code.md # 项目特定的记忆
└── ...
```

详见 [Context & Memory](../context-memory/) 文档。

---

## 为什么需要 Session？

### 长时间对话的支持

```
Session 1 (day 1)：
  用户："帮我设计一个 agent 系统"
  Claude："... (生成详细设计)"
  ↓ 用户关闭

Session 2 (day 2)：
  用户："继续上次的实现"
  Claude："我记得上次我们讨论的架构是..."
  ↓ 恢复上下文，继续工作
```

### 工作的可追溯性

```
用户想要回顾：
  - 昨天在这个项目上做了什么？
  - 哪些决定是如何做的？
  - 之前遇到了什么 bug？
  ↓
session 历史 + MEMORY.md 提供完整上下文
```

---

## 关键文件

| 文件 | 用途 |
|------|------|
| `bootstrap/state.ts` | 56 KB，系统启动时初始化全局状态 |
| `remote/RemoteSessionManager.ts` | 多客户端 session 管理 |
| `tasks/LocalMainSessionTask.ts` | 本地 session 的任务跟踪 |
| `memdir/memdir.ts` | 21 KB，Memory 目录管理 |

---

## 导航

- **[Bootstrap & Cold Start](bootstrap.md)** — 启动时发生了什么
- **[Session Persistence](persistence.md)** — Session 如何保存和恢复

---

**下一步** → [Bootstrap](bootstrap.md)
