---
title: Tool System
layout: default
nav_order: 4
parent: Agentic Design
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

## 核心概念：Tool Interface

每个 Tool 必须满足统一接口（`Tool.ts`）：

```typescript
interface Tool<T extends ZodSchema> {
  name: string                    // 模型识别 tool 的名称，如 "FileReadTool"
  description: string             // 模型理解 tool 用途（出现在 system prompt 中）
  inputSchema: T                  // Zod schema，模型生成的参数必须符合
  call(input: z.infer<T>, ctx: ToolUseContext): AsyncGenerator<ToolResult>
  isSafe?: boolean                // 是否"安全"（影响权限决策）
  shouldDefer?: boolean           // 是否延迟发现（见下文）
  internalMetadata?: { ... }      // 内部使用，不会发给模型
}
```

**Tool Execution Context（`ToolUseContext`）** 提供给每个 tool：
- `readFileState`：文件系统缓存，避免重复读取
- `getAppState`：全局 React 状态访问
- `setToolJSX`：向终端 UI 渲染自定义组件
- `mcpClients`：外部 MCP 协议连接
- `abortController`：取消信号

---

## 工具分类：40+ 个工具

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

## Tool 执行 Pipeline

从"模型调用 tool"到"返回结果"的完整流程：

```
Query Loop 收到 tool_use block
  │
  ├─ 1. 输入验证（Zod schema）
  │     - 确保模型没有传入格式错误的参数
  │     - 路径、字符串类型等全部检查
  │
  ├─ 2. 权限检查（utils/permissions/）
  │     - Mode 检查（bypass / default / auto / dontAsk）
  │     - Always Allow / Always Deny 规则匹配
  │     - YOLO 分类器自动审批
  │     - 如果拒绝 → 返回 permission_denied error
  │
  ├─ 3. 执行 Tool
  │     - 调用 tool.call(input, ctx)
  │     - 得到 AsyncGenerator
  │     - 流式 yield ToolProgressData（实时显示进度）
  │
  ├─ 4. 收集 / 截断结果
  │     - 大输出通过 ContentReplacementState 截断
  │     - 或存为附件，不直接放入 context window
  │
  └─ 5. 发送给 Claude
        - 组合为 tool_result block
        - 追加到 messages 数组
        - 继续 Query Loop
```

---

## 并发执行：多个 Tool 同时运行

一条 Claude 消息中可能包含多个 `tool_use` block：

```
Claude:
  1. 读文件 A（tool_use id=1）
  2. 读文件 B（tool_use id=2）
  3. 读文件 C（tool_use id=3）
```

这三个 tool 可以**并发执行**，但有几个复杂性：

### In-Order Emission（保序输出）

虽然执行可能乱序，但**结果必须按调用顺序返回**：

```
调用顺序: tool_1, tool_2, tool_3
执行时序:         tool_2 ✓（快速）
        tool_1 ✓（较慢）
                         tool_3 ✓

返回顺序: tool_1 结果 → tool_2 结果 → tool_3 结果
```

实现方式：结果缓冲 + 有序 emit：

```typescript
const results = new Map<number, ToolResult>()
let nextIdToEmit = 0

for each completed tool {
  results.set(tool.id, tool.result)
  while (results.has(nextIdToEmit)) {
    yield results.get(nextIdToEmit)
    nextIdToEmit++
  }
}
```

### Sibling Abort（快速失败）

如果一个 tool 失败，其**兄弟 tool 会被立即取消**：

```
tool_1 → 运行中
tool_2 → 抛异常 ✗
tool_3 → 运行中

↓ tool_2 失败时立即 abort tool_1 和 tool_3
↓ 整个批次失败，告知 Claude 出了问题
```

---

## Tool 注册与过滤

在 `tools.ts` 中，所有 tool 通过 `buildTool` 工厂函数创建，注册到中央注册表：

```typescript
const ALL_TOOLS: Tool[] = [
  AgentTool,
  BashTool,
  FileReadTool,
  FileEditTool,
  ...
]
```

然后根据用户权限和配置进行过滤：

```typescript
const PUBLIC_TOOLS = ALL_TOOLS.filter(t => !t.internalOnly)
const ANT_ONLY_TOOLS = [REPLTool, ConfigTool, ...]
```

**`buildTool` 工厂模式**保证：所有 tool 都有一致的类型安全和默认行为。

---

## Safe vs Unsafe Tool 分类

| 类型 | 典型工具 | 特点 |
|------|---------|------|
| 只读（低风险） | `FileReadTool`, `GlobTool`, `GrepTool`, `WebFetchTool` | 无副作用，auto mode 下自动执行 |
| 写操作（中风险） | `FileEditTool`, `FileWriteTool`, `BashTool`（git 命令） | 需规则匹配或 YOLO 分类器 |
| 高风险 | `BashTool`（任意命令）, `AgentTool`, `REPLTool` | 通常需要用户确认 |

---

## 进度渲染与 Streaming

Tool 执行时会 emit 类型化的进度数据，实时显示在终端：

| 进度类型 | 场景 |
|---------|------|
| `BashProgress` | Shell 命令的 stdout / stderr 流式输出 |
| `MCPProgress` | 远程 tool 调用状态 |
| `WebSearchProgress` | 搜索词 + 结果数量 |
| `REPLToolProgress` | 代码求值的中间输出 |

大输出通过 `ContentReplacementState` 截断或转为附件，避免撑爆 context window。

---

## Design Decision 专栏

### 为什么 Tool 返回 `AsyncGenerator` 而不是 `Promise<string>`？

不好的设计：
```typescript
call(input): Promise<string> {
  const result = await readFile(input.path)
  return result  // 10 MB 的代码文件？用户一直等
}
```

更好的设计：
```typescript
call(input): AsyncGenerator<ToolResult> {
  yield { type: "progress", message: "Reading..." }
  const content = await readFile(input.path)
  yield { type: "content", data: content }
  yield { type: "complete", lines: 100 }
}
```

Generator 把**长时间操作的透明度**提升为一级特性——UI 立即显示进度，用户不会以为卡住了。

### 为什么 FileEditTool 用精确字符串匹配而不是行号？

行号方式的问题：
```
第 42 行插入代码
→ 但 Claude 读文件时是第 42 行，执行时已被编辑成第 45 行了
→ 插到错误位置
```

精确字符串匹配：
```typescript
// 找到这段文本（只允许出现一次），替换为新文本
{ old_text: "function foo() {", new_text: "function bar() {" }
```

精确匹配保证了**编辑的确定性**——如果 old_text 出现多次，操作失败，强迫 Claude 提供更精确的上下文。

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

- `Tool.ts`：Tool interface 定义
- `tools.ts`：Tool 注册和过滤
- `services/tools/StreamingToolExecutor.ts`：并发执行编排
- `utils/permissions/permissions.ts`：权限检查
- `tools/AgentTool/AgentTool.tsx`：最复杂的 tool（~233 KB）

下一步：去了解**多 Agent 协调**，看看当 Claude 想要 spawn 一个子 agent 时，系统如何支持这个。
