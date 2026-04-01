---
title: Ink 渲染引擎
layout: default
nav_order: 1
parent: Terminal UI
grand_parent: Agentic Design Overview
---

# Ink Rendering Engine：cli/print.ts 的故事

`cli/print.ts` 是 Claude Code 最大的文件之一（212 KB），它是整个系统的"输出黑洞"。所有文本最终都通过这里流向用户。

---

## print.ts 的职责

### 1. 消息格式化

```typescript
print.message("Hello", {
  prefix: "✓",
  color: "green"
})

// 可能的输出：
// ✓ Hello
// [agent-A] Hello
// [13:45:02] Hello
```

格式化的维度：
- 时间戳
- Agent 标记（哪个 agent 的输出）
- 前缀符号（✓, ✗, ⚠, ...）
- 颜色和样式

### 2. 结构化输出（NDJSON）

某些场景需要机器可读的输出：

```typescript
print.result({
  type: "task_completed",
  taskId: "abc-123",
  status: "success",
  duration: 5432
})

// 输出（当 --format=ndjson 时）：
// {"type":"task_completed","taskId":"abc-123","status":"success","duration":5432}
```

### 3. 多后端支持

```
输出目标：
  ├── stdout（标准输出）
  ├── HTTP（远程客户端）
  ├── 文件（日志记录）
  └── 记忆系统（归档）

cli/print.ts 提供统一接口，底层自动路由到正确的后端
```

---

## 输出通道（Streams）

### 生产者 → print.ts → 消费者

```typescript
// 生产者（Agent）
agent.log("Starting task")
  ↓ 进入 print.ts

// print.ts 的队列
message queue:
  ├── "Starting task" (type: INFO)
  ├── "✓ Task 1 done" (type: SUCCESS)
  └── "Context: 75%" (type: METRIC)
  ↓

// 消费者（终端）
console.log("[13:45:02] [agent-A] Starting task")
process.stdout.write("✓ Task 1 done\n")
```

### 背压处理（Backpressure）

当输出太快时：

```
Agent 产生消息的速度 > 终端输出的速度
  ↓
print.ts 内部队列堆积
  ↓
当队列达到 X 条消息时，返回压力信号
  ↓
Agent 的 log() 调用返回 Promise，需要 await
  ↓
Agent 暂停，等待队列处理
```

---

## Ink 集成

### Ink 组件如何发送输出

```typescript
// 组件定义
<AgentProgressLine agentId="A" progress={45} />

// Ink 渲染模型
component render()
  ↓ 返回字符串
  ↓ 例如："[████░░] 45% 完成"
  ↓ 交给 cli/print.ts
  ↓ print.ts 发送给终端
```

### 混合 Ink 组件和直接输出

```typescript
// 同时存在：

<MessagePane>  // Ink 组件，响应式
  {messages.map(m => <Message key={m.id} />)}
</MessagePane>

// 以及

print.log("User typed: /commit")  // 直接输出

// print.ts 需要协调这两者，避免覆盖
```

---

## 消息编码

### NDJSON 格式（Newline Delimited JSON）

每一行是一个独立的 JSON 对象：

```json
{"type":"query_start","timestamp":"2026-04-01T13:45:02Z"}
{"type":"tool_use","tool":"FileReadTool","path":"/home/user/file.txt"}
{"type":"tool_result","exitCode":0,"lineCount":42}
{"type":"query_end","timestamp":"2026-04-01T13:45:05Z","duration":3000}
```

**为什么是 NDJSON？**
- 每行独立：可以流式解析
- JSON：机器可读
- 行分隔：兼容 `tail -f`, `grep` 等 Unix 工具

### 调用者指定输出格式

```typescript
// 终端使用（人类可读）
const cli = new Claude()
cli.run()  // 彩色输出

// 脚本使用（机器可读）
const cli = new Claude({ outputFormat: "ndjson" })
const data = cli.run()
  .on("data", (json) => {
    const { type, payload } = JSON.parse(json)
    // 处理...
  })
```

---

## 性能优化

### 输出去重

```
同一消息被多次打印时：
  "Context: 75%"
  "Context: 75%"
  "Context: 75%"
  
print.ts 可能只输出一次，省带宽（远程客户端）
```

### 批量输出

```
如果有大量小消息：
  msg1: "Processing..."
  msg2: "Processing..."
  msg3: "Processing..."
  
合并为一条：
  "Processing... (3)"
```

### 速率限制

```
如果终端太慢，本地队列可能满：
  ↓ 丢弃非关键消息（如详细日志）
  ↓ 保留关键消息（如错误、提示）
```

---

## 设计决定专栏

### 为什么要统一所有输出到 print.ts？

**分散输出**：
```
Agent 直接 console.log()
QueryEngine 直接 process.stdout.write()
Tool 直接 return result as string

问题：
- 无法统一格式化
- 无法支持多后端
- 无法做去重优化
- 远程客户端收不到输出
```

**统一漏斗**（当前做法）：
```
所有输出 → print.ts → 终端/HTTP/文件

优势：
- 单一控制点
- 易于添加新后端
- 可以做全局优化（去重、批处理）
- 结构化输出天然支持
```

成本：所有 log 调用都要经过 print.ts，但这个开销极小。

### 为什么支持 NDJSON 而不是 JSON Lines 或其他格式？

**选择 NDJSON**：
- 标准化：RFC 7464 定义
- 兼容：每行是完整 JSON，独立处理
- 简单：基于 newline 分隔，易于流式处理
- Unix 风格：兼容 `tail`, `grep`, `jq` 等工具

---

## 深入阅读

- `cli/print.ts` — 完整的输出引擎实现（212 KB）
- `cli/structuredIO.ts` — NDJSON 编码和解码
- `cli/handlers/` — 不同命令的输出 handler
- `cli/transports/` — 多种输出后端

---

**下一步** → [State → UI Pipeline](state-to-ui.md)
