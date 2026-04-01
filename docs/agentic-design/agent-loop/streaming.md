---
title: Streaming 与 Tool 交织
layout: default
nav_order: 1
parent: 核心 Query Loop
grand_parent: Agentic Design Overview
---

# Streaming 与 Tool 交织

## 复杂的交织

这是一个复杂的地方。API 可能在单条消息中返回混合内容：

```json
{
  "content": [
    { "type": "text", "text": "让我先读文件..." },
    { "type": "tool_use", "id": "t_1", "name": "FileReadTool", "input": {...} },
    { "type": "text", "text": "现在我看到问题了..." },
    { "type": "tool_use", "id": "t_2", "name": "FileEditTool", "input": {...} }
  ]
}
```

所以一条消息中可能**混合了文本和 tool call**。Loop 需要：
- 收集每个 `tool_use` block 的完整输入参数（可能分段到达）
- 在 `tool_use` 结束时**立即执行**（不等待消息全部返回）
- 继续接收后续的文本或 tool call
- 当消息完全接收后，将所有 tool 结果作为一条 user 消息发回

这使得 **interleaved thinking** 成为可能：模型可以一边思考一边调用工具。

---

## Tool 并发执行

一条 Claude 消息中可能包含多个 `tool_use` block：

```
Claude:
  1. 读文件 A（tool_use id=1）
  2. 读文件 B（tool_use id=2）
  3. 读文件 C（tool_use id=3）
```

这三个 tool 可以**并发执行**，但有几个复杂性：

### In-Order Emission（保序输出）

虽然执行可能乱序，但**结果必须按调用顺序返回**：

```
调用顺序: tool_1, tool_2, tool_3
执行时序:         tool_2 ✓（快速）
        tool_1 ✓（较慢）
                         tool_3 ✓

返回顺序: tool_1 结果 → tool_2 结果 → tool_3 结果
```

实现方式：结果缓冲 + 有序 emit：

```typescript
const results = new Map<number, ToolResult>()
let nextIdToEmit = 0

for each completed tool {
  results.set(tool.id, tool.result)
  while (results.has(nextIdToEmit)) {
    yield results.get(nextIdToEmit)
    nextIdToEmit++
  }
}
```

这确保了模型看到的结果总是有序的，即使执行顺序不同。

### Sibling Abort（快速失败）

如果一个 tool 失败，其**兄弟 tool 会被立即取消**：

```
tool_1 → 运行中
tool_2 → 抛异常 ✗
tool_3 → 运行中

↓ tool_2 失败时立即 abort tool_1 和 tool_3
↓ 整个批次失败，告知 Claude 出了问题
```

这个机制防止了级联的浪费：如果其中一个 tool 因为权限被拒或路径不对而失败，没有理由继续执行其他的 tool。整个批次失败，模型可以重新规划。

---

## 读写分区调度

系统对工具执行进行了**分类调度**：

- **读操作**（FileReadTool, GlobTool, GrepTool 等）：最多 **10 个并发**执行
- **写操作**（FileEditTool, FileWriteTool, BashTool 等）：**串行执行**（防止冲突）

为什么这样分类？

```
读操作：
  多个读是安全的（不改状态）
  可以并发，减少总延迟
  限制为 10 个防止内存爆炸（10 个大文件读取）

写操作：
  必须串行（防止竞态条件）
  假设 tool_1 删除文件，tool_2 修改该文件
  如果并发执行，文件可能不存在
  串行执行：tool_1 删除 → tool_2 看到文件不存在 → 正确处理
```

---

## 关键设计决策

### 为什么工具执行在 streaming 时即启动？

如果等待消息完全接收：

```
模型 streaming 中...（已输出 70%）
  ← API 还没说"最后一个 tool_use"
  所以 loop 还不知道有哪些 tool 要执行
  一直等待消息完成
  
消息完全接收（100%）
  ↓ 现在知道了所有 3 个 tool_use
  ↓ 开始执行
  ↓ 总耗时：消息接收 + 工具执行
```

如果在 streaming 时即启动：

```
模型 streaming 中...（输出 30%）
  ← 收到第一个 tool_use block
  ↓ 立即执行（不等后续）
  
模型继续 streaming（40-70%）
  ← 继续收到新 tool_use block
  ↓ 立即执行
  
模型完成（100%）
  ↓ 第一个工具早就执行完了
  ↓ 总耗时：max(消息接收, 工具执行)
```

第二种方式通过**流水线并行**，大幅减少总耗时。

---

## 进度渲染与反馈

工具执行时会 emit 类型化的进度数据，实时显示在终端：

| 进度类型 | 场景 |
|---------|------|
| `BashProgress` | Shell 命令的 stdout / stderr 流式输出 |
| `MCPProgress` | 远程 tool 调用状态 |
| `WebSearchProgress` | 搜索词 + 结果数量 |
| `REPLToolProgress` | 代码求值的中间输出 |

用户看到的是**实时的工作进度**，而不是"卡住了"的假象。

大输出通过 `ContentReplacementState` 截断或转为附件，避免撑爆 context window。

---

## 总结

Streaming 与 Tool 交织的核心设计：

1. **混合内容处理**：一条消息可能同时包含文本和多个 tool call
2. **提前启动执行**：不等待消息完成，立即执行已收到的 tool
3. **保序输出**：执行可乱序，结果必须按调用顺序返回
4. **快速失败**：一个 tool 失败立即取消兄弟 tool
5. **分区调度**：读并发、写串行，防止资源浪费和竞态条件
6. **实时反馈**：工具执行中不断 emit 进度，用户看到实时输出

这套机制让 Claude Code 的交互体验非常流畅——不是一个工具完成后再执行下一个，而是所有工具尽可能并发执行，中间穿插模型的思考和输出。

---

## 下一步

了解 [Token Budget 与 Compaction](token-budget.md)，看看当工具执行产生大量输出时，系统如何管理有限的 context window。
