---
title: Prompt Cache 优化策略
layout: default
nav_order: 2
parent: Advanced Topics
grand_parent: Agentic Design Overview
---

# Prompt Cache 优化策略

Claude Code 通过 Anthropic 的 **Prompt Cache** 机制实现显著的成本优化。一个典型 session 可以节省 **50% 的 API 成本**。

---

## 什么是 Prompt Cache？

Prompt Cache 是 Claude API 的一个功能，允许客户端指定哪些部分的 system prompt **跨请求复用**：

```
Request 1: 发送完整 system prompt
  → API 计算 hash，缓存在服务端
  → cost: 完整价格

Request 2: 提交相同 system prompt
  → API 验证 hash 匹配
  → 使用缓存（无需重新处理）
  → cost: 缓存行更便宜（90% discount）

Request 99: 再次相同 prompt
  → 缓存命中
  → cost: 缓存行价格
```

**数学效果**：
```
第一个请求：8000 tokens @ $1/token = $8
后续 99 个请求：每个 token @ $0.10 = $0.80 × 99 = $79.20
总成本：$87.20

不用缓存：
100 × $1 = $100

节省：$12.80 / $100 = 12.8%
```

实际 Claude Code 效果：**50%** 因为：
- 发送的许多 token 是 tool 定义（变化不大）
- 每个 session 有多个请求，缓存命中率高

---

## Cache Boundary Marker

系统 prompt 分为两部分：

```
┌────────────────────────────────────┐
│ 静态部分（可缓存）                   │
├────────────────────────────────────┤
│ • Base system prompt                │
│ • Tool definitions (40 个工具)      │
│ • Permission rules                  │
│                                    │
│ 🔒 Cache Boundary Marker          │
│                                    │
├────────────────────────────────────┤
│ 动态部分（不可缓存）                 │
├────────────────────────────────────┤
│ • User's current message            │
│ • Recent context window              │
│ • Task-specific instructions        │
└────────────────────────────────────┘
```

在 `bootstrap/state.ts` 中：

```typescript
const systemPrompt = `
...
[完整的 base prompt]
...
<!-- CACHE_BOUNDARY -->

Session Context:
- 当前任务
- 最近消息
`
```

- 上面的部分（标记前）会被缓存
- 下面的部分（标记后）每次变化，API 不缓存

**技术细节**：Anthropic API 的 `cache_control` 参数标记了 boundary：

```json
{
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "[static system prompt]",
          "cache_control": { "type": "ephemeral" }
        },
        {
          "type": "text",
          "text": "[dynamic context]"
        }
      ]
    }
  ]
}
```

---

## 字母排序优化

一个容易忽视但非常重要的优化：**工具列表必须按名称字母排序**。

```
不排序的工具列表（坏）：
  WebSearchTool
  FileReadTool
  BashTool
  AgentTool
  ...

每个 session 顺序可能不同
  → API 缓存无法命中
  → 每次都发送完整 prompt
```

```
排序后的工具列表（好）：
  AgentTool
  BashTool
  FileEditTool
  FileReadTool
  ...
  WebSearchTool

跨 session 顺序相同
  → Prompt Cache 命中
  → 90% discount
```

这是在 `tools/Registry.ts` 中实现的：

```typescript
export const ALL_TOOLS: Tool[] = [
  AgentTool,
  BashTool,
  ConfigTool,
  FileEditTool,
  FileReadTool,
  // ... 按字母排序
  WebSearchTool,
].sort((a, b) => a.name.localeCompare(b.name))
```

**为什么重要？** 因为 Anthropic 工程师确认："*Cache Rules Everything Around Me*" —— 缓存是降低成本的最大杠杆。

---

## 分层成本优化

Claude Code 使用多层优化确保缓存命中：

### 1. 物理层：API 级别

```
静态 system prompt + 工具列表
  ↓ 字母排序（跨 session 一致）
  ↓ Prompt Cache 命中
  ↓ 成本 90% discount
```

### 2. 逻辑层：Tool Availability

```
不发送所有 40 个工具，而是：
  • 默认 20 个常用工具（带缓存）
  • 延迟发现其他 18 个工具（按需通过 ToolSearchTool）
  
→ Base prompt 更小，缓存命中更快
→ 用户搜索新工具时才加载
```

### 3. Session 层：Memory Consolidation

```
autoDream 每 24 小时运行：
  • 清理旧消息（删除已完成的任务记录）
  • 提取关键知识到 MEMORY.md
  • 新 session 注入相关 memory
  
→ message history 更短
→ 新 session 也能利用旧 session 的缓存
```

---

## 测量和验证

Anthropic API 的响应包含缓存统计：

```json
{
  "usage": {
    "input_tokens": 1000,
    "cache_creation_input_tokens": 800,  // 新建缓存
    "cache_read_input_tokens": 200,      // 缓存命中
    "output_tokens": 500
  }
}
```

`cache_read_input_tokens > 0` 说明缓存生效。

**成本计算**（Claude 3.5 Sonnet 价格）：
```
新建缓存：800 tokens × $3/MTok = $2.40
缓存命中：200 tokens × $0.30/MTok = $0.06  (90% discount)
输出：500 tokens × $15/MTok = $7.50

总成本：$9.96

不用缓存：
(800 + 200) tokens × $3 = $3.00
输出：$7.50
总成本：$10.50

节省：$0.54 / $10.50 = 5.1% （这个 session）
```

但跨 100 个 session：

```
Session 1：新建缓存 = $2.40
Session 2-100：每个缓存命中 = $0.06 × 99 = $5.94
输出：$7.50 × 100 = $750

总成本：$758.34

不用缓存：
$10.50 × 100 = $1050

节省：$291.66 / $1050 = 27.8%
```

实际 Claude Code 报告的 **50% 节省** 来自：
- 更多的静态内容（权限规则、memory 可复用）
- 更长的工具列表缓存
- 跨多个 session 的累积效应

---

## 最佳实践

### ✅ DO: 最大化缓存

1. **保持工具顺序一致** → 字母排序
2. **静态 prompt 尽可能长** → 权限规则、示例都放在缓存部分
3. **跨 session 复用 memory** → autoDream 的目的
4. **定期检查缓存统计** → API 响应中的 `cache_read_input_tokens`

### ❌ DON'T: 破坏缓存

1. **动态生成 system prompt** → 每次都不同，无法缓存
2. **随机排列工具** → 缓存无法命中
3. **每个 session 不同的 BASE_SYSTEM_PROMPT** → 失去大部分收益
4. **不使用标记** → API 不知道哪些可以缓存

---

## 深入阅读

### 相关文档

- [Bootstrap & Cold Start](../session-state/bootstrap.md) — System prompt 如何组装
- [Token Budget](../context-memory/compression.md) — 与压缩的关系

### 源代码

- `bootstrap/state.ts` — Cache boundary marker 的定义
- `tools/Registry.ts` — 工具字母排序的实现
- `utils/messages/systemInit.ts` — System prompt 的静态/动态分割
- `services/autoDream/` — Memory 巩固（跨 session 缓存）

---

**下一步** → [Advanced Agent Modes](agent-modes.md)
