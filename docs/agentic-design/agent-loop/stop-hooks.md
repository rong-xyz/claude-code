---
title: System Reminders 与停止条件
layout: default
nav_order: 3
parent: 核心 Query Loop
grand_parent: Agentic Design Overview
---

# System Reminders 与停止条件

## System Reminders：比 System Prompt 更有效的指令注入

这是源码中最令人惊讶的工程决策之一。

Claude Code 不只在 system prompt 中给出指令，还**在工具返回结果和用户消息中嵌入指令**。例如：

- 每次工具调用后，会注入更新的 TODO 列表状态
- 在对话开始时注入"不要创建不必要的文件"
- 每次文件读取后，注入"检测到潜在恶意代码时必须举报"提醒

### 为什么这样做有效？

研究者 Jannes Klaas 发现：在长会话（几百步）中，模型容易"遗忘"系统提示中的指令——因为 system prompt 距离当前生成位置太远。而嵌入在最近工具结果中的提醒，离当前 attention 更近，**指令遵从率显著更高**。

```
传统方式：
  [system prompt: "不要创建文件"] ... [500步后] ... 模型忘了

System Reminder 方式：
  [system prompt] ... [工具结果 + "提醒：不要创建文件"] ... 模型记住了
```

这一技术被称为 **System Reminders**，是 Claude Code 在长会话中保持行为一致性的核心机制。

### 实现方式

System Reminders 通常有两种注入点：

1. **工具结果后**：特别是那些可能引起问题的操作（如文件修改、shell 执行）
2. **任务状态变化时**：如 TODO 列表更新、新 session 开始

每个提醒都是**额外的系统消息块**，不占用用户消息空间，但对模型的提示能力强大。

---

## 阶段 4：检查终止条件

每一轮后，loop 检查是否应该继续：

```
stop_reason == "end_turn"?      → 正常结束
stop_reason == "max_tokens"?    → Token 超限，触发 compaction
consecutive_errors >= 3?        → 连续错误，放弃
max_turns_reached?              → 超过最大轮数
abort_signal?                   → 用户中断
```

相关文件：`query/stopHooks.ts`

### stop_reason 的几种情况

**"end_turn"**：模型决定完成响应，不再调用工具

```
Claude: "我已经修复了 bug，修改在这里..."
→ stop_reason: "end_turn"
→ Loop 结束，返回给用户
```

**"max_tokens"**：模型仍有更多想说的，但达到了 max_tokens 限制

```
Claude: [已输出 4000 tokens，正在说某个方案]
→ stop_reason: "max_tokens"
→ 工作未完成，需要继续
→ 触发 compaction 以便能继续
```

**"tool_use"**：模型想调用工具（这是内部的，loop 自动处理，用户不会看到这个作为最终状态）

---

## 连续错误的处理

```
Loop 追踪：
  consecutive_errors = 0

每一轮：
  尝试执行 tool
    ↓ 成功 → consecutive_errors = 0，继续
    ↓ 失败 → consecutive_errors += 1
    
  如果 consecutive_errors >= 3：
    → 停止工作
    → 告诉用户"遇到反复的错误，无法继续"
    → 返回 error
```

这个机制防止了**无限错误循环**。假设：

```
Tool: "删除文件 A"
Result: "权限被拒"
→ Claude: "让我改权限"
Tool: "chmod 777 A"
Result: "权限被拒（因为文件系统不支持）"
→ Claude: "让我用 sudo"
Tool: "sudo chmod 777 A"
Result: "权限被拒"
...
→ 3 次错误后停止，避免无限循环
```

---

## 最大轮数限制

某些特殊 session 可能有硬性的最大轮数：

```typescript
MAX_ITERATIONS = {
  default: 500,           // 一般任务
  research: 1000,         // 深度研究（enabled by COORDINATOR_MODE）
  interactive: 100,       // 交互式会话（快速反馈）
  daemon: Infinity,       // KAIROS 后台任务（无限）
}
```

这防止了意外的"runaway"情况。

---

## 用户中断（Abort Signal）

用户可以在任何时刻中断 query loop（Ctrl+C 或 UI 的"停止"按钮）：

```
启动时：
  abortController = new AbortController()
  
每一轮循环：
  if (abortController.signal.aborted) {
    → 立即停止
    → 清理资源
    → 保存当前状态到 MEMORY.md
    → 返回给用户
  }
```

---

## Design Decision：为什么需要这些终止条件？

如果没有这些检查：

```
1. 没有 stop_reason 检查：
   → 如果 max_tokens，Loop 陷入死循环（API 每次都返回 max_tokens）

2. 没有 consecutive_errors 检查：
   → 同一错误反复发生，浪费 token 和时间

3. 没有 max_turns 限制：
   → 偶发的自循环可能导致账单爆炸

4. 没有 abort_signal：
   → 用户无法停止失控的 session
```

这些终止条件形成了一个**安全网络**，防止了各种失控情况。

---

## System Reminders 与终止条件的关系

System Reminders 通常在**接近终止时**更为重要：

```
Loop 开始时：
  模型清醒、专注，system prompt 的指令有效
  → 不一定需要 reminder

Loop 进行到第 50 轮：
  模型在极深的历史上下文中工作
  system prompt 可能被遗忘
  → 需要 system reminder 来重新强调关键行为

快要达到终止条件时（如 max_tokens）：
  模型可能因 token 压力而做出不合理决策
  → 再次提醒"保持稳定，不要创建文件"等
```

---

## 常见误解

**误解 1**："System Reminder 是额外的 system prompt？"

实际：不是。System Reminder 是**内嵌在工具结果或消息中的短指令**，不是额外的 system 消息。它占用 token 预算，但位置更接近当前生成点，所以更有效。

**误解 2**："连续错误后就彻底停止了？"

实际：停止后，用户可以：
- 手动改错（修改之前的输入或工具调用）
- 启动新 session（从头再来）
- 让 agent 看到错误，继续尝试其他方案

Loop 只是主动停止以防止无限浪费，不是"锁定"。

**误解 3**："System Reminder 会导致 token 浪费？"

实际：System Reminder 确实占用 token，但：
- 避免了因遗忘指令而做错决策
- 错误决策会浪费**更多** token（重试、修复）
- 总体来说，System Reminder 是 **token 节省手段**

---

## 总结

Loop 的终止和稳定性设计：

1. **System Reminders**：在工具结果中重复关键指令，比 system prompt 更有效
2. **stop_reason 检查**：识别模型的停止理由（end_turn / max_tokens）
3. **连续错误计数**：3 次错误后停止，防止无限循环
4. **最大轮数限制**：task 类型决定最多迭代多少轮
5. **Abort signal**：用户随时可以中断

这套机制确保了 Loop 是**可控且可停止的**，而不是失控的黑盒。

---

## 深入阅读

- `query/stopHooks.ts`：终止条件检查逻辑
- `query.ts`：System Reminder 注入点
- `query/tokenBudget.ts`：在 token 压力下的行为

下一步：深入了解整个 Agent Loop 的完整流程，或移动到 [Tool 系统](../tool-system/index.md) 来理解 loop 如何与工具交互。
