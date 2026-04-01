---
title: System Prompt 与 Cache Boundary
layout: default
nav_order: 2
parent: Context & Memory
grand_parent: Agentic Design Overview
---

# System Prompt 与 Cache Boundary

## System Prompt 架构：Cache-Aware 设计

不是简单的一个大 string，而是**模块化的**，分为**静态部分**和**动态部分**：

```typescript
const systemPrompt = [
  // 第 1 部分：静态（缓存）
  SYSTEM_PROMPT_PREFIX,          // 关于 Claude Code 的基础指导
  TOOL_DESCRIPTIONS,             // 所有 tool 的描述
  SYSTEM_PROMPT_SAFETY,          // 安全指导

  // ← 缓存边界（Cache Boundary Marker）

  // 第 2 部分：动态（每个 session 不同）
  USER_MEMORY,                   // 用户的个人知识库（MEMORY.md）
  RECENT_CONTEXT,                // 最近几轮对话总结
  CURRENT_TASK,                  // 当前任务的上下文
]
```

## Cache Boundary 的妙处

```
初始化时：                     │首个请求：
cache_key = MD5(prefix)       │发送所有 system prompt（~5000 tokens）
缓存大小：10 KB               │
                              │API 缓存 prefix（10 KB）
                              │处理动态部分（~3000 tokens）
同一 session 中再次使用：      │总成本：~40 KB（约 8000 tokens）
cache 仍有效                  │
直接跳过 prefix（10 KB）     │再次请求：
成本：动态部分 ~1000 tokens   │API 直接用缓存的 prefix
                              │只发送新的动态部分（~1000 tokens）
                              │总成本：~20 KB（节省 50%！）
```

如果 session 很长（100 次请求），节省 = **~4.9 MB 的传输量**！

## 为什么区分静态和动态？

不区分（一大块）：
```
发第 1 个请求：发 5000 tokens 的 system prompt + 输入
发第 2 个请求：又发 5000 tokens 的 system prompt + 输入
...
浪费了大量 token 在重复发送
```

区分（cache boundary）：
```
发第 1 个请求：
  发送静态 prefix（5000 tokens）+ 动态部分（1000 tokens）
  API 缓存这 5000 tokens

发第 2 个请求：
  直接用缓存的 5000 tokens
  只发送新的动态部分（1000 tokens）
  节省了 5000 tokens！

100 次请求的 session：
  节省 = 5000 * 99 = 495K tokens ≈ $5 savings
```

---

## System Prompt 与 Feature Flag 的关系

不同的 feature flag 组合会生成不同的 system prompt：

```typescript
const systemPrompt = [
  BASE_PROMPT,

  // 编译期条件
  feature("BRIDGE_MODE") ? BRIDGE_MODE_INSTRUCTIONS : "",
  feature("VOICE_MODE") ? VOICE_INSTRUCTIONS : "",

  // 运行期条件
  flags.get("tengu_coordinator") ? COORDINATOR_INSTRUCTIONS : "",
  flags.get("chicago_beta") ? COMPUTER_USE_INSTRUCTIONS : "",
]
```

这也影响 **prompt cache**：
```
缓存键 = MD5(system_prompt_prefix)

如果 flag 变化，cache key 变化，旧 cache 失效
```

所以：高频变化的 runtime flag 应该放在 cache boundary 之后（动态部分）。

---

## 相关文件

- `utils/messages/systemInit.ts`：System prompt 组装
- `context.ts`：上下文构建逻辑
- `constants/systemPromptSections.ts`：System prompt 各部分定义
