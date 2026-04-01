---
title: Tool 注册表与权限分类
layout: default
nav_order: 2
parent: Tool System
grand_parent: Agentic Design Overview
---

# Tool 注册表与权限分类

## Tool 注册与过滤

在 `tools.ts` 中，所有 tool 通过 `buildTool` 工厂函数创建，注册到中央注册表：

```typescript
const ALL_TOOLS: Tool[] = [
  AgentTool,
  BashTool,
  FileReadTool,
  FileEditTool,
  ...
]
```

然后根据用户权限和配置进行过滤：

```typescript
const PUBLIC_TOOLS = ALL_TOOLS.filter(t => !t.internalOnly)
const ANT_ONLY_TOOLS = [REPLTool, ConfigTool, ...]
```

**`buildTool` 工厂模式**保证：所有 tool 都有一致的类型安全和默认行为。

## Safe vs Unsafe Tool 分类

| 类型 | 典型工具 | 特点 |
|------|---------|------|
| 只读（低风险） | `FileReadTool`, `GlobTool`, `GrepTool`, `WebFetchTool` | 无副作用，auto mode 下自动执行 |
| 写操作（中风险） | `FileEditTool`, `FileWriteTool`, `BashTool`（git 命令） | 需规则匹配或 YOLO 分类器 |
| 高风险 | `BashTool`（任意命令）, `AgentTool`, `REPLTool` | 通常需要用户确认 |

## 深入阅读

### 相关文档

- [Permission Rules](../permission-system/rules.md) — 规则如何门控工具访问

### 源代码

- `tools.ts`：Tool 注册和过滤
- `utils/permissions/permissions.ts`：权限检查
