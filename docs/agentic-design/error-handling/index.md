---
title: 错误处理
layout: default
nav_order: 9
parent: Agentic Design Overview
has_children: true
---

# Error Handling：一级设计公民

Claude Code 的核心设计哲学之一是："**Failure modes are first-class**" —— 错误不是异常情况，而是系统必须优雅处理的日常现象。

## 为什么需要显式的错误处理？

Agent 系统运行在不可靠的环境中：

- **API 调用失败**：Claude API timeout、rate limit、服务不可用
- **用户拒绝**：权限提示被拒绝、连续拒绝触发熔断
- **工具执行失败**：Bash 命令返回非零退出码、文件不存在、权限不足
- **资源耗尽**：Context window 超限、Token budget 用完、磁盘满
- **并发竞争**：多个工具并行执行，其中一个失败需要中止其他

如果不处理好这些失败，Agent 要么：
- 💥 完全卡住（等待永不到达的响应）
- 🔄 无限重试（消耗资源但不进展）
- ❌ 丢失工作状态（用户无法恢复）

Claude Code 的策略：**分层失败恢复** + **清晰的 abort 路径**。

---

## 四大失败模式

### 1. Permission Denials（权限拒绝）

用户在权限提示被拒绝：

```
请求 1: Bash(git push) → 用户拒绝
请求 2: Bash(git reset) → 用户拒绝  
请求 3: Bash(git clean) → 用户拒绝
  ↓ 熔断触发！
回退为手动模式（bypass → acceptEdits）
继续工作，但每个命令都提示
```

这叫 **Denial Tracking** —— 连续 3 次拒绝打开熔断器。

### 2. Tool Execution Failures（工具执行失败）

某个工具调用失败（非权限问题）：

```
Tool A (Read File)     → ✅ 成功，发送结果
Tool B (Bash Command)  → ❌ 失败 (exit code 127)
Tool C (Edit File)     → 🚫 中止（sibling abort）

为什么中止 C？因为 C 可能依赖 B 的输出。
A 的结果留给 agent 分析失败原因。
```

### 3. API Errors & Timeouts（API 错误）

Claude API 返回错误或超时：

```
query.ts 捕获错误
  ↓
检查是否可重试（5xx → 是，4xx → 否）
  ↓ 可重试
等待指数退避后重试
  ↓ 不可重试或重试 N 次失败
返回给 agent，让它调整策略
```

### 4. Resource Exhaustion（资源耗尽）

Token budget 或 Context window 用完：

```
Token 用量 → 80%：警告
         → 92%：触发 AutoCompaction
         → 95%：触发 SnipCompaction（激进）
         → 100%：返回 413 error，显示内存压力
             ↓ 用户选择: 保存 session / 清空 context / 继续
```

---

## 关键概念

**Abort Signal** — 通知 agent 和工具停止当前工作
- 来自：用户按 Ctrl+C、sibling 工具失败、权限熔断
- 传播路径：query loop → 活跃工具 → 子 agent

**Circuit Breaker** — 在多次失败后停止重试
- 权限拒绝 3 次 → 熔断，回退到手动确认
- API 连续失败 N 次 → 熔断，返回错误给用户

**Graceful Degradation** — 功能不可用时降级而不是崩溃
- YOLO 模式失败 → 回退到提示模式
- Compaction 失败 → 限制新请求大小，保持运行
- 远程 agent 超时 → 强制杀死，清理资源

---

## 关键文件

| 文件 | 行数 | 用途 |
|------|------|------|
| `utils/abortController.ts` | - | Abort signal 管理和传播 |
| `utils/permissions/denialTracking.ts` | - | 熔断逻辑（3 次拒绝） |
| `query/tokenBudget.ts` | - | Token 耗尽的恢复策略 |
| `services/tools/toolExecution.ts` | - | 单个工具失败处理 |
| `services/tools/StreamingToolExecutor.ts` | - | 并行工具中的 sibling abort |

---

## 导航

- **[Abort Architecture](abort-architecture.md)** — Abort signal 的传播链
- **[Circuit Breakers](circuit-breakers.md)** — 熔断器和恢复策略
- **[Recovery Strategies](recovery-strategies.md)** — 各层的重试和降级逻辑

---

**下一步** → [Terminal UI](../terminal-ui/)
