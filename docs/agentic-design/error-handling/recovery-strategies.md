---
title: Recovery Strategies
layout: default
nav_order: 3
parent: Error Handling
grand_parent: Agentic Design Overview
---

# Recovery Strategies：各层的恢复策略

不同层级的失败需要不同的恢复策略。Claude Code 采取**分层恢复**，从最轻微的降级到最后的放弃。

---

## Tool 执行层的恢复

### 工具失败的三层处理

```
[1] 工具执行出错（exit code != 0）
      ↓
[2] 返回错误给 agent，包括错误信息
      ↓
[3] Agent 决定：
      - 重新尝试（修复问题后重试）
      - 跳过（该工具结果不重要）
      - 升级（寻求用户帮助）
```

Claude 由 agent 决定是否重试，而不是工具层自动重试。这给 agent 更多控制权。

### Streaming 工具中途失败

```
工具正在流式返回结果：

Tool.execute() {
  for (const chunk of stream) {
    if (error during read) {
      ↓
      yield partial results
      yield error marker
      return
    }
  }
}
```

关键：已经 yield 的结果不会被取消，agent 可以用这些部分结果。

---

## Permission 层的恢复

### 拒绝后的回退链

```
权限模式: [YOLO] → [Auto] → [AcceptEdits] → [DontAsk] → [Bypass]

如果当前模式失败：
  YOLO 被拒绝 3 次
    ↓
  降级到 Auto（需要用户主动允许）
    ↓
  继续被拒绝
    ↓
  降级到 AcceptEdits（显示所有编辑，需要确认）
    ↓
  仍然失败
    ↓
  降级到 DontAsk（自动通过，但记录所有）
```

### YOLO 分类器失败

如果 ML 分类器不确定（confidence < threshold）：

```
分类器: "75% 确定这个 Bash 命令是安全的"
  ↓ 阈值是 90%（可配置）
  ↓ 不足以自动执行
  ↓
回退为提示用户（跳过 YOLO，显示权限对话）
```

这是一种**拒绝不确定**的策略：宁可问用户也不要自动执行可疑操作。

---

## Query Loop 层的恢复

### 连续错误的回退

```
while (true) {
  try {
    response = await api.ask(messages)
  } catch (error) {
    
    if (consecutiveErrors >= 3) {
      // 3 次连续错误：放弃这个 query
      throwToUser("API 连续失败，请稍后重试")
      break
    }
    
    consecutiveErrors++
    await sleep(2 ^ consecutiveErrors)  // 指数退避
    continue  // 重试
  }
  
  consecutiveErrors = 0  // 成功后重置
}
```

### Token 超限恢复（已详细记录）

```
Token 用量 80% → 主动压缩（AutoCompact）
        92% → 激进压缩（SnipCompact）
        95% → 限制新请求大小
        100% → 返回 413，显示内存压力
```

详见 [Context & Memory](../context-memory/) 文档。

---

## Agent 被杀死时的恢复

### 远程 Agent 超时

```
远程 agent (CCR) 执行超过 30 分钟
  ↓
强制终止该 agent
  ↓
收集最后已知的状态：
  - 已完成的工作
  - 未完成的任务
  - 生成的文件
  ↓
向主 agent 报告："Agent XYZ 因超时被杀死"
  ↓
主 agent 决定：
  - 继续用已完成的工作
  - 重新启动 agent
  - 放弃任务
```

### 子 Agent 崩溃

```
子 agent 进程异常退出
  ↓
AsyncLocalStorage 清理该 agent 的上下文
  ↓
向主 agent 返回：AgentCrashError
  ↓
主 agent 可以：
  - 重新启动子 agent
  - 使用之前的缓存结果
  - 降级为内联执行
```

---

## Coordinator 模式中的恢复

### Research 失败

```
Research 阶段耗时过长或卡住
  ↓
Timeout: 30 分钟
  ↓
向 Coordinator 报告："Research 阶段超时"
  ↓ Coordinator 决定
  ↓
用部分结果继续 Synthesis
```

### Synthesis 失败

```
Synthesis 无法综合 Research 的结果
  ↓
Coordinator 检测：无法继续
  ↓
回到 Research 阶段，寻求更多信息
  或
放弃 Coordinator 模式，回到标准 Query loop
```

---

## 设计决定专栏

### 为什么工具层不自动重试，而是让 Agent 决定？

**自动重试**：
```
BashTool 命令失败 → 自动重试 3 次
问题 1：某些失败是永久的（权限、文件不存在），重试没用
问题 2：浪费时间（每次重试可能 5-10 秒）
问题 3：工具不知道上下文（可能需要先修复环境）
```

**Agent 决定**（当前做法）：
```
工具失败 → 返回错误给 agent
  ↓ agent 看到错误信息
  ↓ agent 可能：
    - 修复错误（如：文件不存在 → 创建文件）
    - 换个方法（如：权限不足 → 用 sudo）
    - 放弃（如：网络不可用 → 告知用户）
```

优势：agent 有完整上下文，可以做出更智能的决定。

### 为什么 Compaction 失败时用激进修剪而不是完全放弃？

**完全放弃**：
```
AutoCompaction 失败
  ↓ 返回错误给用户
  ↓ agent 不能继续工作
  ↓ 长 session 用户崩溃
```

**激进降级**（当前做法）：
```
AutoCompaction 失败
  ↓ 尝试 SnipCompaction（删除更多内容）
  ↓ 如果成功：恢复工作
  ↓ 如果仍失败：最后返回 413
```

优势：给系统尽可能多的机会恢复，用户的工作优先级最高。

---

## 深入阅读

- `query.ts` — Query loop 中的错误处理和重试逻辑
- `services/tools/toolExecution.ts` — 工具执行的错误传播
- `services/api/` — API 调用的重试和降级策略
- [Circuit Breakers](circuit-breakers.md) — 熔断器的实现细节
