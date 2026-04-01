---
title: Context 与 Memory
layout: default
nav_order: 7
parent: Agentic Design Overview
has_children: true
---

# Context 和 Memory：有限的脑容量

## 为什么这很重要？

Claude 的 context window（约 **200K tokens**）是**有限的**。一个长时间运行的 agent task：

```
第 1 轮：用户输入（200 tokens）+ 响应（1000 tokens）
第 2 轮：新输入（300 tokens）+ 响应（2000 tokens）
...
第 100 轮：？

总 tokens: 200 + 300 + 400 + ... + 整个历史

在第 50 轮时：messages 数组有 100 条消息，超过 context window 的 80%
```

一旦超限，系统必须做些什么。Claude Code 的答案是三管齐下：

1. **System Prompt 优化**：区分静态和动态部分，利用 prompt cache
2. **三层主动压缩**：MicroCompact → AutoCompact → SnipCompact
3. **长期 Memory**：用户的持久化知识库，跨 session 保留

---

## 核心设计理念

Context window 是最稀缺的资源。这影响系统的每个决策：

- **System Prompt 分割**：静态部分缓存，动态部分每次更新
- **主动压缩**：在 80–95% 时触发，而不是等到接近极限
- **记忆巩固**：自动化的后台 consolidation，而不是手动维护
- **相关性搜索**：新 session 只注入相关的 memory 内容

---

## 关键要点

1. **System prompt 分两部分**：静态（可缓存）+ 动态（每次变化），节省 50% API 成本
2. **Token budget 主动追踪**：到达 80–95% 时触发压缩，防止冲出上限
3. **三层压缩策略**：MicroCompact（快） → AutoCompact（中）→ SnipCompact（慢但救命）
4. **Memory 系统持久化知识**：MEMORY.md 最多 200 行，跨 session 保留
5. **autoDream 后台巩固**：每 24 小时自动更新 MEMORY.md，无需用户介入
6. **相关性搜索**：新 session 只注入相关的 MEMORY.md 内容，节省空间
7. **优化层级**：70% → 20% → 10% → 5% → 停止，每个阈值有对应策略

---

## 深入阅读

- [Compression 三层递进](compression.md)
- [System Prompt 与 Cache Boundary](memdir-system.md)
- [autoDream 后台巩固](auto-dream.md)
- [Memory 隔离与 Design Decision](memory-isolation.md)

核心文件：
- `query/tokenBudget.ts`：Token 追踪逻辑
- `services/autoDream/autoDream.ts`：Dream 实现（~11 KB）
- `memdir/memdir.ts`：Memory 目录管理
