---
title: autoDream 后台记忆巩固
layout: default
nav_order: 3
parent: Context & Memory
grand_parent: Agentic Design Overview
---

# autoDream 后台记忆巩固

## Memory 文件结构

```
~/.claude/memory/
├── MEMORY.md              ← 用户的长期知识库（最多 200 行）
├── projects/
│   └── claude-code.md     ← 项目特定的知识
├── patterns/
│   └── async-patterns.md  ← 学到的模式
├── people/
│   └── team.md            ← 关于人的信息
└── 24h-old-sessions/      ← 即将清理的旧 session
```

**MEMORY.md** 的内容示例：

```markdown
# 我的知识库

## 工作偏好
- 不喜欢在 main 分支上直接提交
- 总是用 TypeScript，避免 JavaScript
- 代码审查前必须运行测试

## 项目信息
- Claude Code 仓库: ~/projects/claude-code
- 技术栈: TypeScript + React + Ink
- 关键贡献者：Alice, Bob

## 编码习惯
- 函数 <100 行为最佳
- 避免 deep nesting，最多 3 层
- 总是加 error handling
```

---

## autoDream 过程：四个阶段

**触发条件**（三门齐全）：
- 距离上次 dream ≥ **24 小时**？
- 距离上次 dream，产生了 ≥ **5 个新 session**？
- 能获得 `consolidation lock`（防并发）？

都满足 → 执行 dream：

```
[1] ORIENT（定向）
    读取用户的 MEMORY.md
    扫描最近 N 个 session 的 transcripts

[2] GATHER（收集）
    提取新的学习点：
      - "用户在这个 session 中学到了异步 Rust"
      - "发现项目从 4.1 升级到了 4.2"
      - "优化了 CI/CD，速度提升 30%"
      - "团队新加入两个成员"

[3] CONSOLIDATE（巩固）
    合并到 MEMORY.md：
      - 如果已经有同类笔记，合并
      - 如果是新的，添加
      - 保持在 200 行以内（做优先级选择，删除过时的）
      - 更新时间戳

[4] PRUNE & INDEX（修剪 & 索引）
    - 删除 24 小时内不用的 session transcript
    - 更新搜索索引（便于后续 memory 查询）
    - 释放磁盘空间
```

相关文件：
- `services/autoDream/autoDream.ts`：主逻辑（~11 KB）
- `services/autoDream/consolidationPrompt.ts`：告诉 Claude 怎么总结
- `memdir/memdir.ts`：Memory 目录管理

---

## 为什么自动化很关键？

```
手动维护：
  用户要记得更新 MEMORY.md
  → 大部分人会忘记（需要主动意识）
  → Memory 逐渐变陈旧

自动化：
  后台每 24 小时自动 consolidate
  → 用户无感
  → Memory 总是最新的
  → 长期知识不会丢失
```

---

## Memory 查询：相关性搜索

当开始新 session 时，系统不是加载**整个 MEMORY.md**，而是：

```
用户输入："帮我修复 Redux 性能问题"
  ↓
系统搜索 MEMORY.md：
  - "Redux"
  - "性能"
  - "optimization"
  相关行：第 23-27 行（曾经遇到过的 Redux 问题）
  ↓
只注入相关的 5 行，保留 context window 空间
  ↓
Claude 看到：
  "之前用户提到 Redux store 有 memory leak，
   解决方案是用 reselect 做 memoization..."
```

相关文件：`memdir/findRelevantMemories.ts`

---

## 常见误解

**误解 1**："MEMORY.md 会无限增长？"

实际：维持在 **~200 行**（可配置），通过 dream 的 prune 阶段定期清理过时信息。年纪太大（>6 个月）的笔记会被自动归档。

**误解 2**："autoDream 需要用户主动触发？"

实际：完全自动化。后台每 24 小时自动检查并执行，用户无需介入。

---

## 深入阅读

### 相关文档

- [Session State](../session-state/) — Dream 在 session 后台运行

### 源代码

- `services/autoDream/autoDream.ts`：Dream 实现
- `memdir/memdir.ts`：Memory 目录管理
- `memdir/findRelevantMemories.ts`：相关性搜索
