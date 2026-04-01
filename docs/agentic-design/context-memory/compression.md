---
title: Token Budget 与三层压缩
layout: default
nav_order: 1
parent: Context & Memory
grand_parent: Agentic Design Overview
---

# Token Budget 与三层压缩

## Token Budget 生命周期：四个阶段

### 阶段 1：初始化

```typescript
const CONTEXT_WINDOW = 200_000  // 200K tokens
const SYSTEM_PROMPT_TOKENS = 5_000
const RESERVED_BUFFER = 10_000

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
  triggerAutoCompact()
}

if (remaining < available * 0.10) {  // 还剩 10% 以下？
  limitToolOutputSize()
}

if (remaining < 10_000) {  // 紧急状态？
  return ERROR_OUT_OF_TOKENS
}
```

### 阶段 3：警告和恢复

```
还剩 20%：显示黄色警告，继续工作
  ↓ 触发 AutoCompact
  ↓ 压缩完成后，token 预算恢复到 ~50%
  ↓ Agent 继续工作
```

### 阶段 4：最终拒绝

```
还剩 < 5%：停止接受新请求
  → 返回错误，让用户选择
```

---

## Compaction：三层递进压缩

### 第一层：MicroCompact（微压缩）

**触发**：无需 API 调用，直接在本地编辑缓存内容

特点：
- **零 API 开销**
- 极快（毫秒级）
- 影响最小（只改工具输出）

### 第二层：AutoCompact（自动压缩）

**触发**：上下文利用率达 **~92–95%**

流程：
```
1. 分析历史，找出能压缩的部分（长响应消息）
2. 按时间优先级排序（最早的先压缩）
3. 调用 Claude 生成摘要（最多 20K token）：
   "用户让我读了 package.json 和 tsconfig.json，
    发现项目是 TypeScript monorepo..."
4. 替换原始消息
```

结果：从 100 条消息减少到 51 条，token 预算恢复到 ~60–70%。

相关文件：`query.ts` 中的 `compactMessages()` 函数（~200 行）

### 第三层：SnipCompact（激进修剪）

**触发**：特性门控 `HISTORY_SNIP`，API 返回 413 错误时激活

特点：
- 保留"受保护尾部"（最近 N 条消息）
- 删除中间部分（早期历史）
- 最激进，可能丢失信息
- 最后的救命稻草

---

## 为什么触发阈值是 20%？

等到 1% 时才 compact：
```
已用 99%，只有 1% 剩余（~2000 tokens）
启动 compaction，需要消耗大量 tokens
可能没有足够 tokens 完成 → 失败
```

提前到 20% 时 compact：
```
已用 80%，还有 20% 剩余（~40K tokens）
有充足 tokens 做总结
完成后，token 预算恢复到 50%
```

权衡：早点 compact 多付出一些 token（压缩成本），但换来系统稳定性。

---

## Context 使用的优化层级

```
Tier 1（还有 70% token）→ 正常工作
Tier 2（还有 20% token）→ 触发 AutoCompact，限制大工具输出
Tier 3（还有 10% token）→ 只允许低成本操作
Tier 4（还有 < 5% token）→ 停止工作，返回错误
```

---

## 深入阅读

### 相关文档

- [Agent Loop Token Budget](../agent-loop/token-budget.md) — Agent Loop 侧的 token budget 视角

### 源代码

- `query/tokenBudget.ts`：Token 追踪逻辑
- `services/compact/`：Compaction 服务实现
