---
title: Coordinator 4 阶段模式
layout: default
nav_order: 3
parent: 多 Agent 协调
grand_parent: Agentic Design Overview
---

# Coordinator 4 阶段模式

启用 `CLAUDE_CODE_COORDINATOR_MODE=1` 时，agent 进入 coordinator 模式。

## 四个阶段

```
用户请求: "帮我重构这个 monorepo"
  ↓
[1] RESEARCH PHASE（研究阶段）
    Coordinator 说："我需要理解项目结构"
    Worker 1 探索 package.json + tsconfig
    Worker 2 探索 src/ 目录结构
    Worker 3 探索 dependencies 图
    ↓ Workers 并行执行，通过 <task-notification> 报告
    → 输出: 项目现状报告
  ↓
[2] SYNTHESIS PHASE（综合阶段）
    Coordinator 读完全部 worker 报告
    Coordinator 说："基于这些发现，我的计划是..."
    → 输出: 详细重构计划
  ↓
[3] IMPLEMENTATION PHASE（实施阶段）
    Coordinator 把计划分解为具体任务
    Worker A 修改 package.json
    Worker B 更新 tsconfig
    Worker C 调整 src/ 组织
    ↓ Workers 并行执行，结果存入 Tengu Scratchpad
  ↓
[4] VERIFICATION PHASE（验证阶段）
    Coordinator 说："让我验证改动"
    Worker 运行测试
    Worker 检查类型错误
    → 输出: 验证报告
  ↓
完成：Coordinator 总结全流程，输出最终结果
```

## 通信机制：XML Protocol

Worker 通过 XML 消息向 Coordinator 报告进度：

```xml
<task-notification>
  <task-id>research-1</task-id>
  <status>complete</status>
  <summary>Found 3 monorepo packages</summary>
  <details>
    - Package A: React components (3.2 MB)
    - Package B: Utils library (1.1 MB)
    - Package C: CLI tool (0.5 MB)
  </details>
  <output>
    [完整的分析结果]
  </output>
</task-notification>
```

**为什么用 XML？** Claude 可以**直接读懂并生成** XML 格式，便于调试和扩展。

## Tengu Scratchpad：共享工作台

Worker 可以在共享空间写临时数据（特性门控：`tengu_scratch`）：

```typescript
// Worker 1 写入分析结果
await writeToScratchpad("project_structure.md", """
# Monorepo Structure
- packages/ui/
  - components/
  - styles/
- packages/core/
  - types/
  - utils/
""")

// Worker 2 读取并补充信息
const structure = await readFromScratchpad("project_structure.md")
await appendToScratchpad("project_structure.md", """
## Dependencies
- ui depends on core
- core has no external deps
""")

// Coordinator 读取完整内容
const fullReport = await listScratchpad()
```

**优点**：
- Worker 不需要把所有信息放在 user 消息中
- 大文件直接读取，不占用 context window
- 自然的"工作台"抽象

## Design Decision：为什么要等 Research 完全结束才 Synthesis？

如果不等（流式处理）：
```
Worker 1 报告 → Coordinator 开始 synthesis
Worker 2 报告（晚到）→ Coordinator 需要重新 synthesis
Worker 3 报告（更晚）→ 又得重新来一遍
```
浪费 token 在重复的 synthesis。

如果等待（阶段隔离）：
```
所有 Worker 完成 Research
  ↓ Coordinator 一次性读完所有报告
  ↓ 一次 Synthesis，不需要改
```
时间线性增长，但 prompt 清晰度和可预测性更高。

## 深入阅读

### 相关文档

- [Context & Memory](../context-memory/) — Coordinator 如何使用 Memory
- [Error Handling](../error-handling/) — Coordinator 阶段失败恢复

### 源代码

- `coordinator/coordinatorMode.ts`：Coordinator system prompt 和阶段逻辑（~330 行）
