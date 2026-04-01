---
title: 核心 Query Loop
layout: default
nav_order: 3
parent: Agentic Design Overview
has_children: true
---

# 核心 Query Loop：Agent 的心脏

## 为什么需要 Loop？

想象一个简单的"一问一答"的 AI：用户问 → API 回复 → 完成。但 agent 不一样：

- 用户说："帮我修复 bug"
- Agent 读取文件 → 分析问题 → 尝试修复 → 运行测试 → 测试失败 → 再次修改 → 再次测试 → 成功

这个过程需要**多轮往返**，每一轮中 agent 决定"下一步应该执行哪个 tool"。这就是 **Query Loop**。

---

## 真相：令人意外的简洁

多位独立研究者分析源码后，得出相同结论：Claude Code 的核心是一个极其简洁的 `while(tool_use)` 循环——内部代号 **`nO`**。没有 critic 模式、没有角色切换、没有复杂的状态机。

```
用户输入 → messages[] → Claude API (streaming) → 响应
  stop_reason == "tool_use"?
    是 → 执行工具 → 追加 tool_result → 循环
    否 → 返回文本给用户
```

一个典型任务运行 **5–50 个迭代**。模型自身的推理能力驱动整个流程，架构本身只负责"搬运"。

---

## 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| TypeScript（严格模式） | latest | 全部业务逻辑 |
| Bun | latest | 运行时 + 构建工具 |
| React | 18 | 终端 UI 组件 |
| Ink | latest | 终端 React 渲染器 |
| Yoga | latest | Flexbox 布局引擎（终端） |
| Zod | v4 | Schema 验证 |
| Commander.js | latest | CLI 参数解析 |

值得注意：**Claude Code 约 90% 的代码由 Claude 自己编写**——这一数据来自 Anthropic 创始工程师 Boris Cherny 和 Sid Bidasaria 的公开确认。

---

## 核心概念：状态机

Query Loop 本质上是一个**状态机**，在以下几个状态间循环：

```
START
  ↓
QUERY（提交 prompt 给 Claude API）
  ↓
TOOL_USE（API 返回 tool_use block，包含 tool 名 + 输入）
  ↓
执行 Tool（BashTool、FileEditTool 等）
  ↓
RESULT（Tool 返回结果）
  ↓
提交结果给 Claude（作为 user 消息）
  ↓
QUERY（继续循环）
  ↓
END（API 返回 stop_reason="end_turn" 或其他终止条件）
```

---

## 代码位置

关键文件：`query.ts`（~1700 行）

这个文件包含：
- Loop 的完整实现（使用 `AsyncGenerator`）
- 状态跟踪
- Tool 调用的编排
- Streaming 支持
- Compaction 触发
- 错误处理和恢复

```typescript
export function* query(...): AsyncGenerator<QueryMessage | QueryEvent, ...>
```

它返回一个 `AsyncGenerator`，每次 `yield` 都是一个"事件"（API 消息、tool 调用、错误等）。

---

## Loop 的生命周期

### 阶段 1：初始化

当你调用 `query()` 时：
- `messages` 数组初始化（包含 system prompt + 历史消息）
- `token_budget` 初始化（计算剩余 token）
- Compaction 检查（如果历史太长，自动压缩）
- Abort controller 初始化（支持中途停止）

### 阶段 2：发送 Prompt

```typescript
// 构建消息数组
let messages = [system_prompt, ...history];

// 调用 Claude API（streaming 模式）
let response = await claude.beta.messages.create({
  messages,
  system: system_prompt,
  max_tokens: remaining_budget,
  stream: true,  // 所有请求默认流式传输
  ...config
});
```

API 调用使用 `@anthropic-ai/sdk` 的 `beta.messages.create` 方法。模型分工：
- **Sonnet 4/4.5/4.6**：处理核心推理任务
- **Haiku 3.5**：处理轻量任务（配额检查、话题检测、bash 命令安全分类）
- **Opus 4.6**：处理复杂架构推理和 ULTRAPLAN 深度规划

### 阶段 3：Streaming 处理

当 API 返回 `tool_use` block 时，loop 收集完整的参数，然后：

```
接收 tool_use block
  ↓
验证 tool 名称是否存在
  ↓
检查权限（useCanUseTool hook）
  ↓
执行 tool（StreamingToolExecutor）
  ↓
Yield tool 结果给消费者
  ↓
添加到 messages 作为 user 消息
```

**关键优化**：工具执行在**模型仍在 streaming 输出时即开始**，无需等待整条消息结束。读操作可最多 10 个并发执行，写操作串行执行，通过分区调度减少整体延迟。

### 阶段 4：检查终止条件

每一轮后，loop 检查是否应该继续：

```
stop_reason == "end_turn"?      → 正常结束
stop_reason == "max_tokens"?    → Token 超限，触发 compaction
consecutive_errors >= 3?        → 连续错误，放弃
max_turns_reached?              → 超过最大轮数
abort_signal?                   → 用户中断
```

---

## Design Decision 专栏

### 为什么用 `AsyncGenerator` 而不是 callback？

不好的设计（callback）：
```typescript
query(messages, {
  onMessage: (msg) => {...},
  onToolCall: (tool, input) => {...},
  onError: (err) => {...},
  onProgress: (progress) => {...},
})
```

问题：4 种不同的 callback，难以理解执行顺序；消费者需要自己维护状态机；测试困难。

**更好的设计**（Generator）：
```typescript
for await (const event of query(messages, config)) {
  if (event.type === 'message') {...}
  else if (event.type === 'tool_use') {...}
  else if (event.type === 'error') {...}
}
```

优点：单一的 `for await...of` 循环，清晰的顺序；每个 event 都是联合类型，编译器帮你检查；可以在 loop 中添加复杂条件逻辑；易于测试。

Generator 让"一系列事件"变成一级公民。

### 为什么架构极其简单？

Claude Code 的源码揭示了一个核心洞察：**当底层模型足够强大时，最优架构是最简单的架构**。没有 critic 模式、没有复杂编排框架、没有数据库——一个 `while(tool_use)` 循环就足以构建最先进的 AI 编程工具。

Anthropic 工程师称之为 **co-evolution design**：harness 被设计为会随模型能力增强而缩减（"The harness is designed to shrink"）。今天的复杂工程围栏，终将被更智能的模型所简化。

---

## 常见误解

**误解 1**："Query loop 就是不断问 Claude？"

实际：Loop 实现的是一个**两层状态机**：
- 外层：Claude API 的请求/响应循环
- 内层：Tool 的执行和结果收集

Tool 执行完全不涉及 Claude API 的额外调用（除非 tool 本身调用 API）。

**误解 2**："Compaction 会丢失信息？"

实际：Compaction 使用 Claude 自己来总结历史，所以关键信息被保留。丢失的只是"冗余的解释"或"中间尝试"，这些对后续工作没用。

**误解 3**："System prompt 写得越详细越好？"

实际：System prompt 太长会稀释每条指令的权重，且在长会话中容易被遗忘。System Reminders 机制（在工具结果中重复关键指令）比单纯加长 system prompt 更有效。

---

## 关键要点

1. **Query loop 是 `while(tool_use)` 的 `AsyncGenerator`**，内部代号 `nO`，极其简洁
2. **循环的关键状态转移**：QUERY → TOOL_USE → RESULT → QUERY，典型 5–50 轮
3. **Tool 执行提前启动**：在 model streaming 时即开始，最多 10 个读操作并发
4. **System Reminders** 比 system prompt 更有效——在工具结果中注入指令，解决长会话遗忘问题
5. **Compaction 三层递进**，上下文使用率约 92–95% 时触发
6. **Co-evolution design**：架构随模型能力提升而缩减

---

## 深入阅读

- `query.ts`：完整 loop 实现（必读）
- `query/tokenBudget.ts`：Token 追踪逻辑
- `services/tools/StreamingToolExecutor.ts`：Tool 执行编排
- `query/stopHooks.ts`：终止条件检查

下一步：去了解 **[Streaming 与 Tool 交织](streaming.md)**，明确 loop 如何并发执行多个 tool。
