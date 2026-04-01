---
title: Team Memory 同步
layout: default
nav_order: 5
parent: 多 Agent 协调
grand_parent: Agentic Design Overview
---

# Team Memory 同步

## Agent 内存和记忆

每个 agent 有独立的 memory 层级：

```
~/.claude/memory/
├── MEMORY.md                  # 全局知识库（所有 session 共享）
├── team-memory/
│   ├── shared-findings.md     # 团队共享发现
│   └── progress.md            # 团队进展
├── agent-abc123/
│   ├── MEMORY.md              # Agent 特定知识库
│   └── session-transcript.log # Session 记录
```

## Team Memory Sync（`services/teamMemorySync/`）

确保：
- 所有 agent 可以读取共享的 findings
- 写操作时使用 **file lock** 防止竞态
- 记忆自动垃圾收集（24 小时清理）

## File Lock 机制

为什么需要？

```
未加锁时：
  Worker A 读 shared-findings.md
  Worker B 读 shared-findings.md
  Worker A 写入新发现 + 保存
  Worker B 写入新发现 + 保存
  结果：Worker A 的修改被覆盖了

加锁时：
  Worker A 获取 lock（阻塞 B）
  Worker A 读 + 写 + 保存
  Worker A 释放 lock
  Worker B 获取 lock
  Worker B 读 + 写 + 保存
  Worker B 释放 lock
  结果：两个修改都被保留
```

## 深入阅读

### 相关文档

- [Memdir System](../context-memory/memdir-system.md) — memdir 实现细节

### 源代码

- `services/teamMemorySync/`：Team memory 同步实现
- `memdir/memdir.ts`：Memory 目录管理
- `coordinator/coordinatorMode.ts`：Coordinator 中的 memory 协调
