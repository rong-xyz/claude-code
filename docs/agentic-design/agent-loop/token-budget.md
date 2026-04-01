---
title: Token Budget 与 Compaction
layout: default
nav_order: 2
parent: 核心 Query Loop
grand_parent: Agentic Design Overview
---

# Token Budget 与 Compaction

## Context Window 的有限性

Context window 是有限的（Claude Code 使用约 **200K tokens** 的上下文窗口，Sonnet 4.0 上为 195,072 tokens）。Token budget 逻辑：

```
available_tokens = context_window_size - system_prompt_tokens - reserved_buffer

每一轮后：
  usage = response.usage（API 返回的实际使用）
  remaining = available_tokens - usage.input_tokens - usage.output_tokens

  if remaining < threshold:
      触发 compaction（压缩历史）

  if remaining < min_required:
      拒绝继续，返回错误
```

**Threshold** 通常设得比较激进——上下文使用率达约 **92–95%** 时就开始压缩，以避免在真的耗尽时无法完成压缩操作。

相关文件：`query/tokenBudget.ts`

---

## Token Budget 生命周期：四个阶段

### 阶段 1：初始化

```typescript
const CONTEXT_WINDOW = 200_000  // 比如 200K tokens
const SYSTEM_PROMPT_TOKENS = 5_000
const RESERVED_BUFFER = 10_000  // 总是留出来，防止意外超限

available = CONTEXT_WINDOW - SYSTEM_PROMPT_TOKENS - RESERVED_BUFFER
          = 185_000 tokens  // 工作预算
```

### 阶段 2：追踪使用

每一轮 API 调用后，系统获得实际使用数据：

```typescript
response = await claude.beta.messages.create({...})

remaining = available - response.usage.input_tokens - response.usage.output_tokens

// 三个阈值
if (remaining < available * 0.20) {  // 还剩 20% 以下？
  console.warn("Token budget 即将耗尽，触发 compaction")
  triggerAutoCompact()
}

if (remaining < available * 0.10) {  // 还剩 10% 以下？
  console.warn("Token budget 严重不足，限制 tool 使用")
  limitToolOutputSize()
}

if (remaining < 10_000) {  // 紧急状态？
  console.error("无法继续，context 严重超限")
  return ERROR_OUT_OF_TOKENS
}
```

### 阶段 3：警告和恢复

```
还剩 20%：显示黄色警告，继续工作
  ↓
Agent 执行更多 tool（看不到警告或无视了）
  ↓
还剩 10%：触发 AutoCompact（在后台自动压缩）
  ↓
压缩完成后，token 预算恢复到 ~50%
  ↓
Agent 继续工作
```

### 阶段 4：最终拒绝

```
还剩 < 5%：停止接受新请求
  → 返回错误，等待用户决定

用户选择：
  a) 启动新 session（重置 token 预算）
  b) 手动编辑 MEMORY.md，删除过时信息
  c) 继续工作（如果有足够 token 完成）
```

---

## Compaction：三层递进压缩

当 token 预算吃紧时，系统自动压缩历史。这不是一次性操作，而是三层递进策略：

### 第一层：MicroCompact（微压缩）

**触发**：无需 API 调用，直接在本地编辑缓存内容

```typescript
// 扫描工具结果，找出能压缩的部分
// 基于时间（清除超时工具结果）和大小（截断超标输出）

// 只压缩这些工具的结果：
// FileRead, Bash, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite

// Cache-aware 变体：保护 prompt cache 完整性
```

特点：
- **零 API 开销**
- 极快（毫秒级）
- 影响最小（只改工具输出）

### 第二层：AutoCompact（自动压缩）

**触发**：上下文利用率达 **~92–95%**

```
1. 分析历史
   - 找出"能压缩的部分"（长响应消息）
   - 保留用户消息（对话核心）
   - 按时间优先级排序（最早的先压缩）

2. 优先排序
   messages 1-10（最早的 10 条）被标记为可压缩

3. 调用 Claude 做总结
   Claude 生成摘要（最多 20K token）：
   "用户让我读了 package.json 和 tsconfig.json，
    发现项目是 TypeScript monorepo，依赖 React 18..."

4. 替换
   messages = [msg_summary, msg_51, msg_52, ..., msg_100]
   //          ^被压缩成1条    ^保留后面的消息（近期的）
```

结果：从 100 条消息减少到 51 条，token 预算恢复到 ~60–70%。

相关文件：`query.ts` 中的 `compactMessages()` 函数（~200 行）

### 第三层：SnipCompact（激进修剪）

**触发**：特性门控 `HISTORY_SNIP`，在 API 返回 413 错误（context 超限）时激活

```
保留"受保护尾部"（最近 N 条消息）
删除中间部分（早期历史）
```

特点：
- 最激进，可能丢失信息
- 最后的救命稻草

---

## 为什么 Compaction 触发阈值是 20%，而不是 1%？

等到 1% 时才 compact：
```
已用 99%，只有 1% 剩余（~2000 tokens）
这时启动 compaction，自己需要消耗大量 tokens（调用 Claude 做总结）
可能没有足够 tokens 来完成 compaction
→ 失败，系统崩溃
```

提前到 20% 时 compact：
```
已用 80%，还有 20% 剩余（~40K tokens）
这时启动 compaction，有充足 tokens 来做总结
完成后，token 预算恢复到 50%
→ 继续工作，很多时间都有富足的 token 可用
```

权衡：早点 compact 意味着多付出一些 token（压缩的成本），但换来系统稳定性。

---

## 为什么 Compaction 会成功而不是丢失关键信息？

Compaction 由 Claude 做，所以**关键信息被保留**。丢失的是：
- 冗余的解释
- 尝试失败的细节（对后续不相关）
- 已解决的问题的讨论过程

关键事实、洞察、解决方案都被总结保留。

---

## Context 使用的优化层级

当 context 变紧时，系统依次采取行动：

```
Tier 1（还有 70% token）
  ✓ 正常工作
  ✗ 监视使用，准备压缩

Tier 2（还有 20% token）
  ✓ 继续工作（使用压缩后的历史）
  ✗ 触发 AutoCompact
  ✗ 不接受新的大型工具输出（如读 10 MB 文件）

Tier 3（还有 10% token）
  ✓ 只允许低 token 成本的操作
  ✗ 禁止高 token 成本的操作（WebSearch, LLM-heavy)

Tier 4（还有 < 5% token）
  ✗ 停止工作，返回错误
  → 用户可以：
     a) 启动新 session（重置 token 预算）
     b) 手动编辑 MEMORY.md，删除过时信息
     c) 要求系统做一次彻底的 SnipCompact
```

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

实际：Compaction 由 Claude 做，所以关键信息被保留。丢失的只是冗余部分。

---

## 总结

Token budget 管理的核心思想：

1. **主动追踪**：每轮 API 调用后立即计算剩余 token
2. **提前触发**：在 20% 时触发压缩，而不是等到 1%
3. **分层策略**：MicroCompact（快、零成本）→ AutoCompact（中、有成本）→ SnipCompact（极端、救命）
4. **保护关键信息**：由 Claude 做压缩，确保关键信息被保留
5. **优雅降级**：当 token 不足时，限制高成本操作，而不是直接崩溃

---

## 下一步

了解 [System Reminders 与停止条件](stop-hooks.md)，看看如何通过在工具结果中注入指令来改进长会话的稳定性。
