---
title: Tool System
layout: default
nav_order: 4
parent: Agentic Design Overview
has_children: true
---

# Tool 系统：Agent 影响世界的手段

## 为什么需要 Tool？

如果 Claude 只能回复文字，它就只是个聊天机器人。要让 Claude **执行动作**（读文件、改代码、运行命令），需要一个 **Tool 接口**。

Tool 系统解决的问题：
1. 如何告诉 Claude "有哪些动作可用"？
2. 如何验证 Claude 的请求是否合法（权限检查）？
3. 如何让多个 tool 并发执行，但结果保持有序？
4. 如何在 tool 执行失败时恢复？

---

## 工具分类：40+ 个工具

Claude Code 提供了 40+ 个工具，按类别分为三组：

### 文件与 Shell 工具

| Tool | 功能 | 安全级别 |
|------|------|------|
| `BashTool` / `PowerShellTool` | Shell 命令执行，含 `dangerousPatterns` 检查 | 高风险 |
| `FileReadTool` | 带缓存的文件读取（`readFileState`） | 低风险 |
| `FileWriteTool` | 文件写入，含路径安全验证 | 中风险 |
| `FileEditTool` | 精确字符串替换（找到一处替换，防止歧义） | 中风险 |
| `MultiEditTool` | 批量文件编辑 | 中风险 |
| `GlobTool` | 文件名模式匹配（含 ignore 规则） | 低风险 |
| `GrepTool` | 内容搜索（ripgrep 封装，内存高效） | 低风险 |
| `REPLTool` | 持久状态的交互求值环境（变量跨调用保持） | 高风险 |
| `NotebookEditTool` | Jupyter notebook 编辑（保留 JSON metadata） | 中风险 |

### Agent 与任务工具

| Tool | 功能 |
|------|------|
| `AgentTool` | Spawn 子 agent，支持 local / remote / forked |
| `TaskCreateTool` | 创建异步任务（36 字符字母表，防暴力猜测） |
| `TaskUpdateTool` / `TaskGetTool` / `TaskListTool` | 任务状态管理 |
| `TaskStopTool` | 优雅终止任务 |
| `TeamCreateTool` / `TeamDeleteTool` | Agent 团队管理 |
| `SendMessageTool` | Agent 间通信（邮箱模型） |
| `SkillTool` | 执行 markdown 定义的技能命令 |

### MCP、Web 与工具类

| Tool | 功能 |
|------|------|
| `MCPTool` | 代理调用外部 MCP server（动态刷新） |
| `ListMcpResourcesTool` | 枚举 MCP 可用资源 |
| `WebSearchTool` | 网络搜索（服务端执行，每次约 $0.01） |
| `WebFetchTool` | 网页内容获取 |
| `LSPTool` | Language Server Protocol 集成，代码导航 |
| `ConfigTool` | 运行时修改 agent 配置 |
| `TodoWriteTool` | JSON 格式任务列表管理 |
| `ScheduleCronTool` | 定时重复任务调度 |
| `SleepTool` | 等待异步操作完成 |
| `ThinkTool` | 无副作用的推理记录（不执行任何操作） |

系统提示词明确要求：「Use Read instead of `cat`」「Use Grep instead of `grep` or `rg` via Bash」——**专用 tool 永远优先于 Bash 等价命令**，以提高透明度和安全性。

---

## 三个精妙的工程细节

### 1. 延迟工具发现（Deferred Tool Discovery）

约 **18 个工具**标记为 `shouldDefer: true`，在默认情况下**不出现在模型的工具列表中**，隐藏直到通过 `ToolSearchTool` 明确搜索才激活。

```typescript
// 模型调用 ToolSearchTool 来发现被延迟的工具
Agent: "我需要搜索一个管理 cron 任务的工具..."
Tool Call: ToolSearchTool { query: "schedule recurring" }
→ 返回: ["ScheduleCronTool", "CronCreateTool", ...]
→ 这些工具现在可以被调用了
```

**为什么这样设计？**
- 所有工具同时放入 system prompt 会消耗约 200K tokens，直接超出部分模型的上下文窗口
- 延迟发现让**基础 prompt 保持精简**，同时支持弹性扩展
- 当 MCP 工具描述超过上下文窗口的 10% 时，系统自动切换到 MCPSearch 模式

### 2. 按名称字母排序（Prompt Cache 优化）

工具注册表按名称**字母顺序排序**——这不是为了美观，而是一个**关键的 prompt cache 优化**。

```
工具列表顺序：
  AgentTool
  BashTool
  ConfigTool
  FileEditTool
  FileReadTool
  ...（字母序）

每次请求这个顺序都相同 → Claude API 的 prompt cache 命中率最大化
```

Anthropic 工程师确认的工程原则：**"Cache Rules Everything Around Me"**（缓存统治一切）。保持工具顺序在跨请求间一致，是减少 API 成本的核心手段之一。

### 3. 流式工具执行（Streaming Tool Execution）

工具执行在**模型仍在 streaming 输出时即开始**，无需等待整条消息结束：

```
模型 streaming 输出：
  "让我先读取文件..."
  [tool_use block 开始: FileReadTool]  ← 这里立即触发执行，不等后续
  "同时我也需要搜索..."
  [tool_use block 开始: GrepTool]      ← 这里也立即触发

FileReadTool 和 GrepTool 并发执行（都是读操作）
结果按调用顺序返回给模型
```

- **读操作**：最多 **10 个并发**执行
- **写操作**：串行执行（防止冲突）
- 通过**分区调度**（read partition / write partition）减少整体延迟

---

## Safe vs Unsafe Tool 分类

| 类型 | 典型工具 | 特点 |
|------|---------|------|
| 只读（低风险） | `FileReadTool`, `GlobTool`, `GrepTool`, `WebFetchTool` | 无副作用，auto mode 下自动执行 |
| 写操作（中风险） | `FileEditTool`, `FileWriteTool`, `BashTool`（git 命令） | 需规则匹配或 YOLO 分类器 |
| 高风险 | `BashTool`（任意命令）, `AgentTool`, `REPLTool` | 通常需要用户确认 |

---

## 关键要点

1. **Tool interface** 是 agent 能力的合约：名称、Zod schema、执行函数、权限标记
2. **延迟工具发现**：~18 个工具默认隐藏，按需搜索激活，保持 prompt 精简
3. **字母排序**不是美观要求，是 prompt cache 优化的关键
4. **流式并发执行**：读操作最多 10 个并发，写操作串行，结果保序返回
5. **Sibling Abort** 快速失败：一个 tool 报错立即取消兄弟 tool
6. **`buildTool` 工厂**确保所有 tool 类型安全和行为一致

---

## 深入阅读

- [Tool Interface 与执行 Pipeline](tool-interface.md)
- [Tool 注册表与权限分类](tool-registry.md)
- [延迟工具发现机制](deferred-tools.md)
- [MCP Tools 集成](mcp-tools.md)

下一步：了解 **Tool Interface** 的定义，看看如何统一定义所有工具的接口。
