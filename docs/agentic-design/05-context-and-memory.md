---
title: Context & Memory
layout: default
nav_order: 7
parent: Agentic Design Overview
---

# Context 和 Memory：有限的脑容量

## 为什么这很重要？

Claude 的 context window（约 **200K tokens**）是**有限的**。一个长时间运行的 agent task：

```
第 1 轮：用户输入（200 tokens）+ 响应（1000 tokens）
第 2 轮：新输入（300 tokens）+ 响应（2000 tokens）
...
第 100 轮：？

总 tokens: 200 + 300 + 400 + ... + 整个历史

在第 50 轮时：messages 数组有 100 条消息，超过 context window 的 80%
```

一旦超限，系统必须做些什么。Claude Code 的答案是三管齐下：

1. **System Prompt 优化**：区分静态和动态部分，利用 prompt cache
2. **三层主动压缩**：MicroCompact → AutoCompact → SnipCompact
3. **长期 Memory**：用户的持久化知识库，跨 session 保留

---

## System Prompt 架构：Cache-Aware 设计

不是简单的一个大 string，而是**模块化的**，分为**静态部分**和**动态部分**：

```typescript
const systemPrompt = [
  // 第 1 部分：静态（缓存）
  SYSTEM_PROMPT_PREFIX,          // 关于 Claude Code 的基础指导
  TOOL_DESCRIPTIONS,             // 所有 tool 的描述
  SYSTEM_PROMPT_SAFETY,          // 安全指导

  // ← 缓存边界（Cache Boundary Marker）

  // 第 2 部分：动态（每个 session 不同）
  USER_MEMORY,                   // 用户的个人知识库（MEMORY.md）
  RECENT_CONTEXT,                // 最近几轮对话总结
  CURRENT_TASK,                  // 当前任务的上下文
]
```

### Cache Boundary 的妙处

```
初始化时：                     │首个请求：
cache_key = MD5(prefix)       │发送所有 system prompt（~5000 tokens）
缓存大小：10 KB               │
                              │API 缓存 prefix（10 KB）
                              │处理动态部分（~3000 tokens）
同一 session 中再次使用：      │总成本：~40 KB（约 8000 tokens）
cache 仍有效                  │
直接跳过 prefix（10 KB）     │再次请求：
成本：动态部分 ~1000 tokens   │API 直接用缓存的 prefix
                              │只发送新的动态部分（~1000 tokens）
                              │总成本：~20 KB（节省 50%！）
```

如果 session 很长（100 次请求），节省 = **~4.9 MB 的传输量**！

相关文件：
- `utils/messages/systemInit.ts`：System prompt 组装
- `context.ts`：上下文构建逻辑

---

## Token Budget 生命周期：四个阶段

### 阶段 1：初始化

```typescript
const CONTEXT_WINDOW = 200_000  // 比如 200K tokens
const SYSTEM_PROMPT_TOKENS = 5_000
const RESERVED_BUFFER = 10_000  // 总是留出来，防止意外超限

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
  console.warn("Token budget 即将耗尽，触发 compaction")
  triggerAutoCompact()
}

if (remaining < available * 0.10) {  // 还剩 10% 以下？
  console.warn("Token budget 严重不足，限制 tool 使用")
  limitToolOutputSize()
}

if (remaining < 10_000) {  // 紧急状态？
  console.error("无法继续，context 严重超限")
  return ERROR_OUT_OF_TOKENS
}
```

### 阶段 3：警告和恢复

```
还剩 20%：显示黄色警告，继续工作
  ↓
Agent 执行更多 tool（看不到警告或无视了）
  ↓
还剩 10%：触发 AutoCompact（在后台自动压缩）
  ↓
压缩完成后，token 预算恢复到 ~50%
  ↓
Agent 继续工作
```

### 阶段 4：最终拒绝

```
还剩 < 5%：停止接受新请求
  → 返回错误，等待用户决定

用户选择：
  a) 启动新 session（重置 token 预算）
  b) 手动编辑 MEMORY.md，删除过时信息
  c) 继续工作（如果有足够 token 完成）
```

相关文件：`query/tokenBudget.ts`

---

## Compaction：三层递进压缩

当 token 预算吃紧时，系统自动压缩历史。这不是一次性操作，而是三层递进策略：

### 第一层：MicroCompact（微压缩）

**触发**：无需 API 调用，直接在本地编辑缓存内容

```typescript
// 扫描工具结果，找出能压缩的部分
// 基于时间（清除超时工具结果）和大小（截断超标输出）

// 只压缩这些工具的结果：
// FileRead, Bash, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite

// Cache-aware 变体：保护 prompt cache 完整性
```

特点：
- **零 API 开销**
- 极快（毫秒级）
- 影响最小（只改工具输出）

### 第二层：AutoCompact（自动压缩）

**触发**：上下文利用率达 **~92–95%**

```
1. 分析历史
   - 找出"能压缩的部分"（长响应消息）
   - 保留用户消息（对话核心）
   - 按时间优先级排序（最早的先压缩）

2. 优先排序
   messages 1-10（最早的 10 条）被标记为可压缩

3. 调用 Claude 做总结
   Claude 生成摘要（最多 20K token）：
   "用户让我读了 package.json 和 tsconfig.json，
    发现项目是 TypeScript monorepo，依赖 React 18..."

4. 替换
   messages = [msg_summary, msg_51, msg_52, ..., msg_100]
   //          ^被压缩成1条    ^保留后面的消息（近期的）
```

结果：从 100 条消息减少到 51 条，token 预算恢复到 ~60–70%。

相关文件：`query.ts` 中的 `compactMessages()` 函数（~200 行）

### 第三层：SnipCompact（激进修剪）

**触发**：特性门控 `HISTORY_SNIP`，在 API 返回 413 错误（context 超限）时激活

```
保留"受保护尾部"（最近 N 条消息）
删除中间部分（早期历史）
```

特点：
- 最激进，可能丢失信息
- 最后的救命稻草

---

## autoDream：后台内存巩固

Compaction 是**临时的**（压缩 session 内的历史）。但如果用户有**多个 session**，同一信息会被重复压缩。

解决方案：**持久化 memory**，通过后台"做梦"过程自动维护。

### Memory 文件结构

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

### autoDream 过程：四个阶段

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

### 为什么自动化很关键？

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

## Context 使用的优化层级

当 context 变紧时，系统依次采取行动：

```
Tier 1（还有 70% token）
  ✓ 正常工作
  ✗ 监视使用，准备压缩

Tier 2（还有 20% token）
  ✓ 继续工作（使用压缩后的历史）
  ✗ 触发 AutoCompact
  ✗ 不接受新的大型工具输出（如读 10 MB 文件）

Tier 3（还有 10% token）
  ✓ 只允许低 token 成本的操作
  ✗ 禁止高 token 成本的操作（WebSearch, LLM-heavy)

Tier 4（还有 < 5% token）
  ✗ 停止工作，返回错误
  → 用户可以：
     a) 启动新 session（重置 token 预算）
     b) 手动编辑 MEMORY.md，删除过时信息
     c) 要求系统做一次彻底的 SnipCompact
```

---

## Design Decision 专栏

### 为什么区分静态和动态 system prompt？

不区分（一大块）：
```
发第 1 个请求：发 5000 tokens 的 system prompt + 输入
发第 2 个请求：又发 5000 tokens 的 system prompt + 输入
...
浪费了大量 token 在重复发送
```

区分（cache boundary）：
```
发第 1 个请求：
  发送静态 prefix（5000 tokens）+ 动态部分（1000 tokens）
  API 缓存这 5000 tokens

发第 2 个请求：
  直接用缓存的 5000 tokens
  只发送新的动态部分（1000 tokens）
  节省了 5000 tokens！

100 次请求的 session：
  节省 = 5000 * 99 = 495K tokens ≈ $5 savings
```

### 为什么 Compaction 触发阈值是 20%，而不是 1%？

等到 1% 时才 compact：
```
已用 99%，只有 1% 剩余（~2000 tokens）
这时启动 compaction，自己需要消耗大量 tokens（调用 Claude 做总结）
可能没有足够 tokens 来完成 compaction
→ 失败，系统崩溃
```

提前到 20% 时 compact：
```
已用 80%，还有 20% 剩余（~40K tokens）
这时启动 compaction，有充足 tokens 来做总结
完成后，token 预算恢复到 50%
→ 继续工作，很多时间都有富足的 token 可用
```

权衡：早点 compact 意味着多付出一些 token（压缩的成本），但换来系统稳定性。

### 为什么用后台 dream 而不是 inline compaction？

Inline（同步）：
```
在 query loop 中调用 compaction
  ↓
等待 compaction 完成（5-10 秒）
  ↓
用户看到卡顿
```

后台 dream：
```
query loop 继续运行
  ↓
在空闲时或定时触发 dream（24 小时一次）
  ↓
用户感觉不到延迟
```

代价：增加系统复杂性（并发处理）、需要文件锁（防同时 dream）。但值得，因为用户体验好得多。

---

## 常见误解

**误解 1**："Context window 满了就完蛋？"

实际：有多层缓冲和恢复机制。系统会逐步：
1. 显示警告
2. 触发 compaction
3. 限制 tool 输出大小
4. 拒绝新请求
5. 返回错误，让用户选择

完全"卡住"很少发生。

**误解 2**："Compaction 会丢失所有细节？"

实际：Compaction 由 Claude 做，所以**关键信息被保留**。丢失的是：
- 冗余的解释
- 尝试失败的细节（对后续不相关）
- 已解决的问题的讨论过程

关键事实、洞察、解决方案都被总结保留。

**误解 3**："MEMORY.md 会无限增长？"

实际：维持在 **~200 行**（可配置），通过 dream 的 prune 阶段定期清理过时信息。年纪太大（>6 个月）的笔记会被自动归档。

---

## 关键要点

1. **System prompt 分两部分**：静态（可缓存）+ 动态（每次变化），节省 50% API 成本
2. **Token budget 主动追踪**：到达 80–95% 时触发压缩，防止冲出上限
3. **三层压缩策略**：MicroCompact（快） → AutoCompact（中）→ SnipCompact（慢但救命）
4. **Memory 系统持久化知识**：MEMORY.md 最多 200 行，跨 session 保留
5. **autoDream 后台巩固**：每 24 小时自动更新 MEMORY.md，无需用户介入
6. **相关性搜索**：新 session 只注入相关的 MEMORY.md 内容，节省空间
7. **优化层级**：70% → 20% → 10% → 5% → 停止，每个阈值有对应策略

---

## 深入阅读

- `query/tokenBudget.ts`：Token 追踪逻辑
- `services/autoDream/autoDream.ts`：Dream 实现（~11 KB）
- `services/autoDream/consolidationPrompt.ts`：Consolidation 提示词
- `memdir/memdir.ts`：Memory 目录管理
- `memdir/findRelevantMemories.ts`：相关性搜索
- `utils/messages/systemInit.ts`：System prompt 组装

下一步：了解**Feature Gating 系统**，看看 Claude Code 怎么区分内部功能和公开功能。
