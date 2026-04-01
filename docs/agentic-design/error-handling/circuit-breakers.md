---
title: Circuit Breakers
layout: default
nav_order: 2
parent: 错误处理
grand_parent: Agentic Design Overview
---

# Circuit Breakers：熔断器模式

熔断器（Circuit Breaker）是一个经典的可靠性模式：在重复失败后停止尝试，等待系统恢复。

## 权限拒绝熔断

### Denial Tracking 机制

```
用户在权限提示被拒绝时：

Denial #1: Bash(rm -rf /)
  用户拒绝 → 记录
  
Denial #2: Bash(git push)
  用户拒绝 → 记录
  
Denial #3: BashTool / FileEditTool
  用户拒绝 → 记录
  ↓ 触发！
  
熔断器打开：CIRCUIT_OPEN
  后续所有 Bash 工具 → 自动拒绝，不再提示
  回退为 acceptEdits 模式（或 dontAsk 模式）
  ↓ 用户现在必须逐个确认每个操作
```

### 为什么是 3 次？

| 次数 | 含义 | 处理 |
|------|------|------|
| 1 次 | 可能是一次性反悔 | 继续自动 |
| 2 次 | 模式开始显现，但可能是特定类别 | 继续自动 |
| 3 次 | 确认用户不想自动执行 | 打开熔断 |

3 次是心理学上的"模式确认"阈值——足够确定但不过度反应。

### 熔断器状态机

```
CLOSED（正常）
  ↓ 拒绝 #3
  ↓
OPEN（熔断打开）
  ↓ 等待 Θ 时间（如 5 分钟）
  ↓
HALF_OPEN（尝试恢复）
  ↓ 用户明确允许一个操作
  ↓
CLOSED（恢复正常）
```

实际实现可能简化为：
- CLOSED: 自动执行
- OPEN: 手动确认模式

---

## Token 预算熔断

当 Context window 接近上限时：

```
Token 使用百分比 → 行为

0–70%      → 正常，继续工作
70–80%     → WARNING: 显示警告，建议清空历史
80–92%     → AUTO_COMPACT: 自动触发 compaction
92–95%     → SNIP_COMPACT: 触发激进修剪
95–100%    → REJECT: 拒绝新请求，提示内存不足
```

这不是经典的"打开 → 半开 → 关闭"熔断器，而是**分层降级**。

### Compaction 失败的回滚

```
AutoCompaction 在进行中：
  ↓
遇到错误（API 超时、格式错误）
  ↓
回滚：恢复原始 context
  ↓
降级为 SNIP_COMPACT（激进修剪，成本更低）
  ↓
如果还是失败：
  返回 413 error 给用户，显示"context window 已满"
```

**关键**：Compaction 失败时不会让系统瘫痪，而是有备用降级方案。

---

## API 连续失败熔断

Claude API 返回错误时的重试策略：

```
API 调用失败
  ↓
检查错误类型：
  - 5xx（Server Error）→ 可重试
  - 4xx（Client Error）→ 不可重试
  - 429（Rate Limit）→ 指数退避重试
  ↓
可重试：等待 2^n 秒后重试
  尝试 1: 2s 等待
  尝试 2: 4s 等待
  尝试 3: 8s 等待
  尝试 4: 放弃，返回错误
  ↓
不可重试：立即返回错误，agent 决定下一步
```

### 连续失败判定

```
最后 N 个 API 调用中超过 M% 失败
  ↓ 打开熔断
  ↓
后续 API 调用：先检查熔断器状态
  如果 OPEN：直接返回错误（不尝试），节省时间
  如果 CLOSED：正常重试流程
```

---

## 设计决定专栏

### 为什么权限熔断需要用户交互来重置？

**自动重置**：
```
打开 → 等待 5 分钟 → 自动尝试恢复
问题：用户可能根本不想自动执行，5 分钟后又卡住
```

**用户主导重置**（当前做法）：
```
打开 → 显示"熔断打开，现在需要手动确认"
  ↓ 用户明确允许一个操作
  ↓ 测试是否恢复正常
  ↓ 是 → 回到自动模式，CLOSED
  ↓ 否 → 继续手动，OPEN
```

优势：主动权在用户，符合 permission system 的哲学。

### 为什么 Token 预算不用经典的熔断器，而是分层降级？

**经典熔断**：
```
到达 95% → 打开 → 拒绝所有新请求
问题：过于激进，还有 5% 的空间可以用
```

**分层降级**（当前做法）：
```
70% → 警告（给用户时间）
80% → 自动压缩（成本 ~2% token，但节省 20%）
92% → 激进修剪（成本更高，但节省更多）
95% → 最后手段：拒绝新请求
```

优势：最大化利用 context window，同时保持安全边界。

---

## 深入阅读

- `utils/permissions/denialTracking.ts` — 权限拒绝熔断实现
- `query/tokenBudget.ts` — Token 预算和分层降级逻辑
- `services/api/` — API 重试策略

---

**下一步** → [Recovery Strategies](recovery-strategies.md)
