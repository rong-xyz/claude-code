---
title: Abort Architecture
layout: default
nav_order: 1
parent: Error Handling
grand_parent: Agentic Design Overview
---

# Abort Architecture：中止信号的传播链

## 什么是 Abort Signal？

JavaScript 标准 `AbortController` 和 `AbortSignal` 用来通知异步操作"停止，现在"。Claude Code 沿着工具执行链传播这些信号。

```typescript
// 触发 abort 的几种方式
abortController.abort()  // 立即中止
abortController.abort(reason)  // 带原因的中止

// 工具和 agent 检查信号
if (signal.aborted) {
  throw new Error('Aborted')
}
```

---

## Abort 的来源

### 1. 用户按 Ctrl+C

```
终端捕获 SIGINT
  ↓
Query loop 接收中止信号
  ↓
trigger abortController.abort()
  ↓
所有活跃工具立即停止
```

### 2. Sibling Tool 失败

当多个工具并行执行，其中一个失败：

```
工具组: [Tool A, Tool B, Tool C] (并发执行)

Tool A → ✅ 成功 (5 秒)
Tool B → ❌ 失败 (2 秒，exit code 1)
Tool C → 🚫 中止 (sibling abort，因为 B 失败)
```

**为什么要中止 C?**

Tool C 的结果可能被 Tool B 的结果影响。例如：
- B = "查找文件列表"
- C = "编辑 B 找到的第一个文件"

B 失败意味着 C 没有明确的目标，继续执行可能出错。

### 3. 权限熔断

连续 3 次被拒绝后，系统回退到手动确认模式，此时会中止自动操作：

```
3 denials in a row
  ↓ Circuit breaker opens
  ↓ 后续所有自动操作中止
回退为 acceptEdits 模式，需用户逐个确认
```

### 4. 子 Agent 超时或失败

远程 agent 或复杂任务超过时间限制：

```
CoordinatorMode task
  ↓ Research phase: 5 分钟
  ↓ Synthesis phase: timeout!
  ↓ 向 coordinator 发送 abort 信号
  ↓ Verification phase 不再启动
```

---

## Abort 的传播路径

### 顶层：Query Loop

```
Query Loop 状态机
  ↓
  while (true) {
    if (abortSignal.aborted) {
      throw new Error("Aborted by user")
    }
    
    // QUERY → TOOL_USE → RESULT → ...
    const result = await executeTools(tools, { signal })
  }
```

### 中层：Tool 执行器

```
StreamingToolExecutor.execute(tools, { signal })
  ↓
Promise.race([
  Promise.all(tools.map(t => t.execute(signal))),
  abortPromise(signal)
])
  ↓
如果 signal.aborted：
  - 已完成的工具：保留结果
  - 进行中的工具：立即 cancel
  - 未启动的工具：不启动
```

### 底层：单个工具

```
Tool.execute() {
  for (let chunk of stream) {
    if (signal.aborted) {
      stream.cancel()  // 停止读取
      cleanup()
      throw new Error("Aborted")
    }
    
    yield chunk
  }
}
```

### 子 Agent：AsyncLocalStorage 传播

```
主 Agent 生成子 agent
  ↓
asyncLocalStorage.run({ abortSignal }, () => {
  子 agent 启动
    ↓
  子 agent 内的 query loop 使用相同 signal
    ↓
  主 agent abort → 子 agent 自动 abort
})
```

---

## Sibling Abort 的实现

### 并发执行的工具

```
StreamingToolExecutor 收到 [Tool A, Tool B, Tool C]
  ↓
启动 Promise.all([
  executeWithTimeout(A, 30s),
  executeWithTimeout(B, 30s),
  executeWithTimeout(C, 30s)
])
```

### 其中一个失败时

```
Promise.all 中任何一个 reject
  ↓
自动 reject 整个 Promise.all（不等待其他）
  ↓
abortController.abort()
  ↓
其他工具的 cleanup 逻辑
  ↓
向 agent 返回：
  {
    succeeded: [A 的结果],
    failed: [B 的错误],
    aborted: [C 被中止]
  }
```

Agent 看到这个结果，知道：
- A 有结果，可以用
- B 失败了，问题是什么
- C 被中止了，不需要处理

---

## Design Decision 专栏

### 为什么 Abort 信号要传播到子 Agent?

**不传播**：
```
主 agent abort → 子 agent 继续运行
问题：浪费资源，主 agent 已经放弃这个任务，为什么还在运算？
```

**传播**（当前做法）：
```
主 agent abort
  ↓
asyncLocalStorage 中的 signal 被设置为 aborted
  ↓
子 agent 的 query loop 看到 signal.aborted，立即停止
优势：整个系统快速响应用户的 abort 请求
```

### 为什么失败的工具要导致 Sibling 中止，而不是继续?

**继续执行**：
```
工具 B 失败，C 继续
问题：C 可能产生垃圾输出（没有 B 的数据），agent 需要分析无效结果
→ 浪费 token，增加复杂性
```

**立即中止**（当前做法）：
```
工具 B 失败
  ↓
立即 abort 所有 siblings
  ↓
agent 看到：部分成功，部分失败，某些被中止
  ↓
明确知道哪些数据可用，哪些不可用
→ 更简洁的失败处理
```

权衡：牺牲少量完成的工作（C 可能即将完成），换取清晰的失败语义。

---

## 深入阅读

- `utils/abortController.ts` — Abort signal 的封装和管理
- `services/tools/StreamingToolExecutor.ts` — 并发工具执行中的 sibling abort 实现
- `query.ts` — Query loop 中的 abort 检查点
