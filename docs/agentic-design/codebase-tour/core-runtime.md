---
title: Core Runtime
layout: default
nav_order: 1
parent: Codebase Tour
grand_parent: Agentic Design Overview
---

# Core Runtime：系统的心脏

这一章涵盖 Claude Code 的核心执行层：Query loop、Tool 系统、以及驱动它们的 Service 层。

---

## `entrypoints/` — 入口层

系统支持多种启动方式：

| 子目录/文件 | 用途 |
|-----------|------|
| `cli.tsx` | 标准 CLI 入口，React 组件树的根，feature detection |
| `sdk/` | Anthropic SDK 集成（给程序员用） |
| `mcp.ts` | MCP (Model Context Protocol) 服务器 |
| `init.ts` | 初始化逻辑，首次运行配置 |
| `sandboxTypes.ts` | 沙箱相关的类型定义 |

**关键点**：`cli.tsx` 做 feature flag 检测，决定如何初始化全局状态。

---

## `query/` — Query Loop 配置和支撑

这个目录下的文件都是 `query.ts` 的辅助组件：

| 文件 | 用途 |
|------|------|
| `config.ts` | Query 配置构建器，参数组合 |
| `tokenBudget.ts` | Token budget 追踪和强制执行 |
| `stopHooks.ts` | Stop/interrupt 处理 |
| `deps.ts` | Query 依赖注入 |

**最重要的**：`tokenBudget.ts` 控制着"超限时触发 compaction"的逻辑。

关于 Query loop 的完整架构，请阅读 [Agent Loop 文档](../agent-loop/)。

---

## `tools/` — 所有 Tool 实现（40+ 个）

这个目录有 40 多个子目录，每个代表一个 Tool。核心的几个：

### 必读：

| Tool | 文件 | 用途 |
|------|------|------|
| `AgentTool/` | 子 agent 的产卵和管理 | 核心，递归往下 |
| `FileReadTool/` | 读文件 | 基础 |
| `FileEditTool/` | 编辑文件 | 基础 |
| `FileWriteTool/` | 写文件 | 基础 |
| `BashTool/` | 执行 shell 命令 | 基础 |
| `GlobTool/` | 文件匹配 | 基础 |
| `GrepTool/` | 内容搜索 | 基础 |
| `WebFetchTool/` | 获取网页 | 基础 |
| `AskUserQuestionTool/` | 询问用户 | 权限边界 |
| `TodoWriteTool/` | 写 todo list | 后台任务 |

### `AgentTool/` — 深递归

这是最复杂的 Tool，用来 spawn 子 agent：

```
AgentTool/
├── AgentTool.tsx          [233 KB] 主逻辑，agent 生命周期
├── runAgent.ts            执行 agent 的主函数
├── forkSubagent.ts        fork 一个新 agent
├── resumeAgent.ts         恢复之前的 agent
├── loadAgentsDir.ts       从 ~/.agents/ 加载 agent 定义
├── built-in/              内置 agent
│   ├── generalPurposeAgent.ts
│   ├── planAgent.ts
│   ├── exploreAgent.ts
│   └── ... (其他 5 个)
├── agentMemory.ts         Agent 记忆快照
├── agentColorManager.ts   Agent 的颜色分配（便于输出区分）
└── ... (其他支撑文件)
```

**关键概念**：Agent 可以是 local (in-process) 或 remote (CCR)，使用 `AsyncLocalStorage` 隔离上下文。

关于 Agent 产卵和协调的详细信息，请阅读 [Multi-Agent 文档](../multi-agent/)。

---

## `services/` — 业务逻辑服务

这个目录是 Claude Code 的"引擎室"：

### `tools/` — Tool 执行编排

```
services/tools/
├── StreamingToolExecutor.ts    [核心] 并发 tool 执行，结果缓冲
├── toolOrchestration.ts        Tool 调度和权限检查
├── toolExecution.ts            单个 tool 调用的执行
├── toolHooks.ts                Tool 执行前/后的 hook
└── ...
```

**最重要**：`StreamingToolExecutor.ts` 实现了"并发执行但顺序输出结果"的逻辑。这是 Claude Code 能高效并行工作的关键。

### `autoDream/` — 自动记忆压缩

```
services/autoDream/
├── autoDream.ts                [11 KB] Dream 流程：3 个触发门 + 4 个阶段
├── consolidationPrompt.ts      [核心] 告诉 Claude 怎么压缩 memory
├── consolidationLock.ts        防并发的 lock 机制
└── config.ts                   Dream 参数配置
```

**关键点**：Dream 有三个触发条件门：24h 时间 + 5+ 个 session + 拿到 lock。

关于 Memory 系统的详细设计，请阅读 [Context & Memory 文档](../context-memory/)。

### 其他重要服务

| 目录 | 用途 |
|------|------|
| `api/` | Claude API 调用，token 计算，usage 追踪 |
| `mcp/` | MCP 协议服务器的实现 |
| `analytics/` | 事件上报、GrowthBook feature flags |
| `plugins/` | 插件系统 |

---

## 深入阅读

- `query.ts` — 完整的 Query loop 状态机实现（~1700 行）
- `Tool.ts` — Tool 接口和类型定义
- `tools.ts` — Tool 注册系统
- `services/tools/StreamingToolExecutor.ts` — 并发 Tool 执行核心
