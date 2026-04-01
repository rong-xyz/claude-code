---
title: Memory 隔离与 Design Decision
layout: default
nav_order: 4
parent: Context & Memory
grand_parent: Agentic Design Overview
---

# Memory 隔离与 Design Decision

## 子智能体的 Memory 隔离

当 spawn 子 agent 时，其 memory 也被隔离：

```typescript
// 主 agent 的 memory
memdir = "~/.claude/memory/"

// 子 agent 的 memory（通过 AsyncLocalStorage 覆盖）
asyncLocalStorage.run({
  memdir: "/tmp/child-agent-memory",
  parentMemdir: "~/.claude/memory/",
}, () => {
  runChildAgent()  // 子 agent 读取 /tmp/child-agent-memory
})
```

这防止了子 agent 的学习污染父 agent 的 memory。

---

## Design Decision 专栏

### 为什么区分静态和动态 system prompt？

不区分（一大块）：
```
发第 1 个请求：发 5000 tokens 的 system prompt + 输入
发第 2 个请求：又发 5000 tokens 的 system prompt + 输入
...
浪费了大量 token 在重复发送
```

区分（cache boundary）：
```
第 1 个请求：5000 + 1000 tokens，API 缓存前 5000
第 2 个请求：直接用缓存的 5000，只发新的 1000
节省 = 5000 * 99 = 495K tokens ≈ $5 savings（100 次请求）
```

### 为什么 Compaction 触发阈值是 20%，而不是 1%？

等到 1% 时才 compact：
```
已用 99%，只有 1% 剩余（~2000 tokens）
启动 compaction，需要消耗大量 tokens（总结历史）
可能没有足够 tokens 完成 → 失败，系统崩溃
```

提前到 20% 时 compact：
```
已用 80%，还有 20% 剩余（~40K tokens）
有充足 tokens 做总结
完成后，token 预算恢复到 50%
继续工作
```

权衡：早点 compact 多付出一些 token（压缩成本），但换来系统稳定性和可预测性。

### 为什么用后台 dream 而不是 inline compaction？

Inline（同步）：
```
在 query loop 中调用 compaction
  ↓ 等待 compaction 完成（5-10 秒）
  ↓ 用户看到卡顿
```

后台 dream：
```
query loop 继续运行
  ↓ 在空闲时或定时触发 dream（24 小时一次）
  ↓ 用户感觉不到延迟
```

代价：增加系统复杂性（并发处理）、需要文件锁（防同时 dream）。但值得，因为用户体验好得多。

---

## 常见误解

**误解 1**："Context window 满了就完蛋？"

实际：有多层缓冲和恢复机制。系统会逐步：
1. 显示警告
2. 触发 compaction
3. 限制 tool 输出大小
4. 拒绝新请求
5. 返回错误，让用户选择

完全"卡住"很少发生。

**误解 2**："Compaction 会丢失所有细节？"

实际：Compaction 由 Claude 做，所以**关键信息被保留**。丢失的只是：
- 冗余的解释
- 尝试失败的细节（对后续不相关）
- 已解决的问题的讨论过程

关键事实、洞察、解决方案都被总结保留。

**误解 3**："Memory 隔离会导致 token 浪费？"

实际：隔离**节省** token：
- 子 agent 不会读取不相关的 memory
- 子 agent 的学习不会污染父 memory
- 查询相关性搜索时，自动过滤无关内容

---

## 总结：Context & Memory 的整体设计

1. **System prompt 分两部分**：静态（可缓存） + 动态（每次变化）
2. **Token budget 主动追踪**：到达 80–95% 时触发压缩
3. **三层压缩策略**：MicroCompact（快）→ AutoCompact（中）→ SnipCompact（极端）
4. **Memory 系统持久化知识**：MEMORY.md 最多 200 行，跨 session 保留
5. **autoDream 后台巩固**：每 24 小时自动更新，无需用户介入
6. **相关性搜索**：新 session 只注入相关内容，节省空间
7. **优化层级**：70% → 20% → 10% → 5% → 停止

这套机制确保了 Claude Code 可以进行**长时间的 session**，同时保持稳定的性能和成本。

---

## 深入阅读

### 相关文档

- [Team Memory Sync](../multi-agent/team-memory.md) — 多 agent 下的 memory 同步

### 源代码

- `query/tokenBudget.ts`：Token 追踪逻辑
- `services/autoDream/autoDream.ts`：Dream 实现（~11 KB）
- `services/autoDream/consolidationPrompt.ts`：Consolidation 提示词
- `memdir/memdir.ts`：Memory 目录管理
- `memdir/findRelevantMemories.ts`：相关性搜索
- `utils/messages/systemInit.ts`：System prompt 组装
