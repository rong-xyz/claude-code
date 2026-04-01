---
title: Multi-Agent Coordination
layout: default
nav_order: 5
parent: Agentic Design
---

# 多 Agent 协调：并行与分解

## 为什么需要多个 Agent？

有些任务太复杂，一个 agent 力不从心：

- 用户说："帮我把代码库从 TypeScript 迁到 Rust"
- 一个 agent 从第一个文件开始，改 5 个小时后，context 满了

更好的方式：**分解任务**

```
Coordinator agent 解析任务、制定计划
  ↓
Worker agent 1（并行）处理"转换 utils 模块"
Worker agent 2（并行）处理"转换 API 层"
Worker agent 3（并行）处理"转换 UI 组件"
  ↓
Coordinator 总结结果、验证一致性
```

这就是**多 agent 协调**，由 `COORDINATOR_MODE` 启用。

---

## Agent 类型：四种部署模式

Claude Code 支持多种 agent 部署方式，由 `AgentTool` 统一管理（`tools/AgentTool/AgentTool.tsx`，~233 KB）：

### 1. Local Agent（进程内）

```typescript
const subAgent = await spawnAgent({
  type: "local",
  agentDef: generalPurposeAgent,
  initialMessage: "帮我测试这个函数"
})
```

特点：
- 在当前 Node.js 进程中运行
- 共享内存（同一 V8 引擎）
- 无网络延迟，最快的交互
- **上下文隔离**通过 `AsyncLocalStorage` 实现

文件：`tools/AgentTool/forkSubagent.ts`

### 2. Remote Agent（CCR 云容器）

```typescript
const remoteAgent = await spawnAgent({
  type: "remote",
  agentDef: exploreAgent,
  initialMessage: "搜索 codebase 中的 bug"
})
```

特点：
- 在 Cloud Container Runtime（CCR）中运行
- 独立的资源池和网络隔离
- 支持长时间运行：最多 **30 分钟**（ULTRAPLAN）
- 网络传输延迟，但隔离度最高

### 3. Forked Agent（进程派生）

```typescript
const forkedAgent = spawnForkedAgent({
  agentScript: "./my-agent.ts",
  args: ["--mode", "debug"]
})
```

特点：
- 创建新的 Node.js 子进程（`child_process.fork`）
- 完全隔离的 V8 引擎
- 支持 CPU 密集任务（不会阻塞主线程）
- 便于调试和独立监控

### 4. In-Process Teammate（进程内同伴，代号 "amber_flint"）

```typescript
const teammates = await createTeam({
  name: "research-squad",
  agents: [agent1, agent2, agent3],
  strategy: "parallel"
})
```

特点：
- 多个 agent **同时运行**在同一进程
- 完全隔离的逻辑邮箱（inter-agent 消息传递）
- 通过 `AsyncLocalStorage` 实现上下文隔离
- **无网络开销**，同步协调

| 组件 | 职责 |
|------|------|
| `InProcessTeammateTask` | Teammate 生命周期管理 |
| `leaderPermissionBridge` | 权限审批同步（leader → teammates） |
| `teammateMailbox` | 异步消息队列 |
| `teammateLayoutManager` | 终端 UI 布局协调 |

---

## Agent Spawning 和上下文隔离

### 怎样 Spawn 一个 Agent？

通过 `AgentTool` 的唯一接口：

```typescript
// Claude 调用 AgentTool
Agent: "让我 spawn 一个 explore agent 来搜索代码库"

Tool Call:
  name: "AgentTool"
  input: {
    type: "spawn",
    agentDef: "exploreAgent",
    prompt: "搜索 utils/ 下的性能问题",
    isolationMode: "local"  // 或 "remote"
  }

返回:
  agentId: "abc123"
  status: "running"
```

### AsyncLocalStorage：上下文隔离的关键

为什么需要隔离？子 agent 的全局状态不应该污染父 agent：

```typescript
// 主 agent 的全局状态
const sessionId = "parent-session-id"
const permissions = "default"
const memoryDir = "~/.claude/memory/"

// 在子 agent 中，这些应该被**覆盖**
asyncLocalStorage.run({
  sessionId: "child-agent-id",
  permissions: "bubble",        // 特殊权限给 worker
  memoryDir: "/tmp/child-memory",
  parentSessionId: "parent-session-id"  // 追踪父子关系
}, () => {
  runChildAgent()  // 子 agent 看到的是覆盖后的值
})
```

实现：`AsyncLocalStorage` 是 Node.js 19+ 内置的上下文隔离机制，支持 async/await 边界的值传播。

---

## Coordinator 模式：4 个阶段

启用 `CLAUDE_CODE_COORDINATOR_MODE=1` 时，agent 进入 coordinator 模式。

### 阶段结构

```
用户请求: "帮我重构这个 monorepo"
  ↓
[1] RESEARCH PHASE（研究阶段）
    Coordinator 说："我需要理解项目结构"
    Worker 1 探索 package.json + tsconfig
    Worker 2 探索 src/ 目录结构
    Worker 3 探索 dependencies 图
    ↓
    Workers 并行执行，通过 <task-notification> 报告
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
    ↓
    Workers 并行执行，结果存入 Tengu Scratchpad
  ↓
[4] VERIFICATION PHASE（验证阶段）
    Coordinator 说："让我验证改动"
    Worker 运行测试
    Worker 检查类型错误
    → 输出: 验证报告
  ↓
完成：Coordinator 总结全流程，输出最终结果
```

### 通信机制：XML Protocol

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

Coordinator 看到这个消息，就知道：
- Task "research-1" 完成了
- 发现了 3 个包，附带规模和用途信息

**为什么用 XML 而不是二进制 IPC？** Claude 可以**直接读懂并生成** XML 格式，便于调试和扩展。相比之下，二进制协议只有强类型检查，但牺牲了可读性。

系统提示词文件：`coordinator/coordinatorMode.ts`（约 330 行系统 prompt）

---

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
- 自然的"工作台"抽象——就像白板一样

---

## Agent 内存和记忆

每个 agent 有独立的 memory 层级：

```
~/.claude/memory/
├── MEMORY.md                  # 全局知识库（所有 session 共享）
├── team-memory/
│   ├── shared-findings.md     # 团队共享发现
│   └── progress.md            # 团队进展
├── agent-abc123/
│   ├── MEMORY.md              # Agent 特定知识库
│   └── session-transcript.log # Session 记录
```

**Team Memory Sync**（`services/teamMemorySync/`）确保：
- 所有 agent 可以读取共享的 findings
- 写操作时使用 file lock 防止竞态
- 记忆自动垃圾收集（24 小时清理）

---

## 任务状态机

Agent 和子 agent 通过 `LocalAgentTask` 追踪：

```
pending → running → (completed | failed | killed)
```

任务 ID 格式：类型前缀 + 8 字符随机字符串

| 前缀 | 类型 | 示例 |
|------|------|------|
| `b` | 本地 bash | `b_abc12345` |
| `a` | 本地 agent | `a_def67890` |
| `r` | 远程 agent | `r_ghi13579` |
| `t` | in-process teammate | `t_jkl24680` |
| `d` | dream 内存巩固 | `d_mno35791` |

36 字符字母表确保碰撞概率极低（防暴力猜测）。

---

## Design Decision 专栏

### 为什么 Coordinator 要等 Research 完全结束才进行 Synthesis？

如果不等（流式处理）：
```
Worker 1 报告 → Coordinator 开始 synthesis
Worker 2 报告（晚到）→ Coordinator 需要重新 synthesis
Worker 3 报告（更晚）→ 又得重新来一遍
```

问题：浪费 token 和时间在重复的 synthesis。

如果等待（阶段隔离）：
```
所有 Worker 完成 Research
  ↓
Coordinator 一次性读完所有报告
  ↓
一次 Synthesis，不需要改
  ↓
继续下一阶段
```

权衡：
- 阶段隔离：时间线性增长（4 个阶段顺序执行）
- 流式处理：时间可能更短（如果 worker 速度差异大），但 prompt 复杂度增加

Claude Code 选择**阶段隔离**，因为系统 prompt 清晰度和可预测性更重要。

### 为什么 Coordinator 不执行任何 Tool？

**Coordinator 的职责**：解析问题、制定计划、管理分解、总结结果。
**Worker 的职责**：执行所有具体操作（读文件、改代码、运行命令）。

```
如果 Coordinator 也执行 tool：
  - 难以追踪谁做了什么
  - 权限检查变复杂（Coordinator 的权限 vs Worker 的权限）
  - 无法并行（Coordinator 忙于执行 tool 时无法管理其他 worker）

如果 Coordinator 只决策：
  - 所有 tool 执行都在 Worker 中进行
  - Coordinator 可以并行监管多个 Worker
  - 审计日志清晰
```

---

## 常见误解

**误解 1**："Coordinator 也会执行 tool？"

实际：Coordinator **只** spawn worker 和管理任务。所有的 tool 执行都在 worker 中进行。Coordinator 只看消息、做决策、发指令。

**误解 2**："Worker 之间可以直接通信？"

实际：Worker 通过 **Coordinator 中转**。这样 Coordinator 可以：
- 理解全局状态
- 决定优先级
- 在冲突时仲裁

直接 worker-to-worker 通信会导致难以追踪的依赖和死锁。

**误解 3**："多个 agent 肯定比单个快？"

实际：取决于任务：
- **可并行化**（如搜索 3 个不同目录）：并行快 **3 倍**
- **高度依赖**（后续任务需要前序结果）：并行反而**慢**（overhead 大于并行收益）

Claude Code 的 Coordinator 会选择合适的分解策略，或提示用户"这个任务不适合并行"。

---

## 关键要点

1. **四种 Agent 类型**：local（in-process）、remote（CCR 30min）、forked（子进程）、teammate（同进程同伴）
2. **AsyncLocalStorage 隔离**：子 agent 的全局状态完全独立，互不污染
3. **Coordinator 模式**：4 个阶段（research → synthesis → implementation → verification）
4. **XML Protocol 通信**：Worker 通过 `<task-notification>` 报告，Coordinator 中转
5. **Tengu Scratchpad**：大文件存储，不占 context window
6. **Team Memory**：共享发现和进展，file lock 防竞态

---

## 深入阅读

- `tools/AgentTool/AgentTool.tsx`：Agent spawning 逻辑（~233 KB，最复杂的 tool）
- `tools/AgentTool/forkSubagent.ts`：Fork 实现
- `coordinator/coordinatorMode.ts`：Coordinator system prompt 和阶段逻辑
- `services/teamMemorySync/`：Team memory 同步
- `tasks/LocalAgentTask/`：Agent 任务生命周期

下一步：了解**权限系统**，看看 Agent 怎么知道"我可以执行什么 tool"。
