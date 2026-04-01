---
title: 任务状态机与 ID 格式
layout: default
nav_order: 4
parent: 多 Agent 协调
grand_parent: Agentic Design Overview
---

# 任务状态机与 ID 格式

## 任务状态机

Agent 和子 agent 通过 `LocalAgentTask` 追踪：

```
pending → running → (completed | failed | killed)
```

每个转换都被记录，便于审计和调试。

## 任务 ID 格式

任务 ID 由**类型前缀 + 8 字符随机字符串**组成：

| 前缀 | 类型 | 示例 | 用途 |
|------|------|------|------|
| `b` | 本地 bash | `b_abc12345` | 后台 shell 命令 |
| `a` | 本地 agent | `a_def67890` | 进程内子 agent |
| `r` | 远程 agent | `r_ghi13579` | CCR 云容器 agent |
| `t` | in-process teammate | `t_jkl24680` | 同进程伴侣 agent |
| `d` | dream 内存巩固 | `d_mno35791` | autoDream 后台任务 |

**36 字符字母表**（0-9 + a-z）确保碰撞概率极低（防暴力猜测）。

## 任务生命周期

```typescript
// 创建
task = new LocalAgentTask({
  id: "a_abc12345",
  type: "local-agent",
  agentDef: exploreAgent,
  prompt: "search for bugs"
})

// 运行
await task.run()

// 追踪状态
task.status  // pending → running → completed
task.output  // 累积输出
task.errors  // 错误日志

// 取消
task.abort()
```

## 深入阅读

- `tasks/LocalAgentTask/`：Agent 任务生命周期
- `utils/taskId.ts`：ID 生成逻辑
