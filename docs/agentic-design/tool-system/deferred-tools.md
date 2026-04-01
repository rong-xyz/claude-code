---
title: 延迟工具发现机制
layout: default
nav_order: 3
parent: Tool 系统
grand_parent: Agentic Design Overview
---

# 延迟工具发现机制

## 延迟工具发现（Deferred Tool Discovery）

约 **18 个工具**标记为 `shouldDefer: true`，在默认情况下**不出现在模型的工具列表中**，隐藏直到通过 `ToolSearchTool` 明确搜索才激活。

```typescript
// 模型调用 ToolSearchTool 来发现被延迟的工具
Agent: "我需要搜索一个管理 cron 任务的工具..."
Tool Call: ToolSearchTool { query: "schedule recurring" }
→ 返回: ["ScheduleCronTool", "CronCreateTool", ...]
→ 这些工具现在可以被调用了
```

## 为什么这样设计？

所有工具同时放入 system prompt 会消耗约 200K tokens，直接超出部分模型的上下文窗口。延迟发现让：

- **基础 prompt 保持精简**，初始工具列表只有常用的 20 个
- **支持弹性扩展**，通过 ToolSearchTool 按需激活其他工具
- **当 MCP 工具描述超过上下文窗口的 10% 时**，系统自动切换到 MCPSearch 模式，避免 prompt 爆炸

## 按名称字母排序（Prompt Cache 优化）

工具注册表按名称**字母顺序排序**——这不是为了美观，而是一个**关键的 prompt cache 优化**。

```
工具列表顺序：
  AgentTool
  BashTool
  ConfigTool
  FileEditTool
  FileReadTool
  ...（字母序）

每次请求这个顺序都相同 → Claude API 的 prompt cache 命中率最大化
```

Anthropic 工程师确认的工程原则：**"Cache Rules Everything Around Me"**（缓存统治一切）。保持工具顺序在跨请求间一致，是减少 API 成本的核心手段之一。

## 深入阅读

### 相关文档

- [Slash Commands](../slash-commands/) — 命令系统依赖延迟工具发现

### 源代码

- `tools/ToolSearchTool.ts`：工具搜索实现
- `utils/messages/systemInit.ts`：System prompt 组装（工具列表位置）
