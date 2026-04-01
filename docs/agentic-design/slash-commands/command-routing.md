---
title: Command Routing
layout: default
nav_order: 1
parent: Slash Commands
grand_parent: Agentic Design Overview
---

# Command Routing：命令的分派和执行

`commands.ts` (~25K 行) 是一个中央路由器，负责将用户输入映射到正确的命令处理程序。

---

## 命令解析

### 输入示例

```
用户输入："/commit --amend --no-verify"

解析结果：
{
  command: "commit",
  flags: {
    amend: true,
    noVerify: true
  },
  args: []
}

用户输入："/review-pr 123"

解析结果：
{
  command: "review-pr",
  args: ["123"],
  flags: {}
}
```

### 解析算法

```typescript
function parseCommand(input: string) {
  const match = input.match(/^\/(\w+)(.*)$/)
  if (!match) return null
  
  const [, cmd, rest] = match
  const flags = rest.match(/--(\w+)/g) || []
  const args = rest.match(/[^\s-]+/g) || []
  
  return { command: cmd, flags, args }
}
```

---

## 路由表（Routing Table）

### 命令到处理程序的映射

```javascript
const commandHandlers = {
  "commit": require("./commit.ts"),
  "commit-push-pr": require("./commit-push-pr.ts"),
  "review-pr": require("./review-pr.ts"),
  "session": require("./session.ts"),
  "memory": require("./memory.ts"),
  "task": require("./task.ts"),
  "agent": require("./agent.ts"),
  // ... 80+ 更多
}
```

### 动态加载（延迟加载）

```
某些命令（如 /web-search）不是立刻加载，而是延迟加载：

用户输入 "/commit"
  ↓ 加载 ./commit.ts
  ↓ 执行

用户没有输入 "/web-search"
  ↓ ./web-search.ts 不加载
  ↓ 节省内存
```

---

## 命令处理程序的接口

### Handler 函数签名

```typescript
interface CommandHandler {
  (options: {
    args: string[],
    flags: Record<string, boolean>,
    appState: AppState,
    api: APIClient,
    tools: ToolRegistry,
    agent: Agent,
    output: OutputFunnel
  }): Promise<CommandResult>
}
```

### 例子：commit 命令

```typescript
export const commitCommand: CommandHandler = async (options) => {
  const { args, flags, appState, agent, output } = options
  
  // 1. 获取 staged files
  const stagedFiles = await agent.runTool("BashTool", {
    command: "git diff --staged --name-only"
  })
  
  // 2. 提示 agent 生成 commit message
  const message = await agent.query({
    systemPrompt: COMMIT_SYSTEM_PROMPT,
    userMessage: `Generate a commit message for these files: ${stagedFiles}`
  })
  
  // 3. 执行 commit
  if (flags.amend) {
    await agent.runTool("BashTool", {
      command: `git commit --amend -m "${message}"`
    })
  } else {
    await agent.runTool("BashTool", {
      command: `git commit -m "${message}"`
    })
  }
  
  output.log(`✓ Committed: ${message}`)
  
  return {
    status: "success",
    message: message
  }
}
```

---

## 出错处理

### 命令不存在

```
用户输入："/foo"
  ↓ 路由表中找不到 "foo"
  ↓ 返回：
    "Unknown command: /foo"
    "Did you mean: /focus?"
```

使用 Levenshtein 距离做模糊匹配。

### 权限不足

```
用户输入："/commit" （但权限模式是 "bypass"）
  ↓
Handler 尝试调用 BashTool
  ↓ 权限检查失败
  ↓
返回：
  "Permission denied: BashTool requires approval"
```

### 参数错误

```
用户输入："/review-pr abc"  （abc 不是有效的 PR 号）
  ↓
Handler 验证参数
  ↓ 无效
  ↓
返回：
  "Invalid PR number: abc"
  "Usage: /review-pr <number>"
```

---

## 命令的输出格式

### 两种输出模式

#### 1. 人类可读（默认）

```
$ claude-code
> /commit

✓ Committed: feat: add error handling layer

Staged files:
  - error-handling/index.md
  - error-handling/abort-architecture.md
  - error-handling/circuit-breakers.md
  - error-handling/recovery-strategies.md

Commit message:
  feat: add error handling layer (Phase 3.1)
  
  - Document abort architecture and sibling abort
  - Add circuit breaker patterns for denial tracking
  - Include recovery strategies for all layers
```

#### 2. 机器可读（NDJSON）

```
$ claude-code --format=ndjson
{"type":"command_start","command":"commit","timestamp":"..."}
{"type":"tool_output","tool":"BashTool","output":"On branch main\n..."}
{"type":"command_result","status":"success","message":"feat: add error handling"}
{"type":"command_end","command":"commit","totalTime":5234}
```

---

## 特殊命令：/help

### 列出所有命令

```
> /help

Available commands:
  /commit                  Git commit workflow
  /review-pr <number>      Review a GitHub PR
  /session list            List sessions
  /memory-dream            Trigger memory consolidation
  ...

Type '/help <command>' for details about a specific command.
```

### 显示特定命令帮助

```
> /help commit

/commit [--amend] [--no-push]

Smart git commit workflow with AI suggestions.

Options:
  --amend               Amend to previous commit
  --no-push             Don't push after committing

Examples:
  /commit
  /commit --amend
  /commit --no-push
```

---

## 设计决定专栏

### 为什么要有 88 个命令而不是让 Agent 自己决定？

**纯 Agent 方法**：
```
用户："帮我审查 PR 123"
Agent："好的，我会分析代码，审查设计..."
  ↓ Agent 设计自己的流程（可能低效）
  ↓ 5 分钟后完成
```

**带有 /review-pr 命令**：
```
用户："/review-pr 123"
Agent："我知道 /review-pr 的流程，直接执行..."
  ↓ 30 秒完成（已优化的流程）
```

成本：维护 88 个命令很复杂，但好处值得。

### 为什么支持 NDJSON 输出？

**纯文本**：
```
无法被程序解析
只能给人看
```

**NDJSON**：
```
程序可以 parse 每一行 JSON
用于：
- IDE 集成（JetBrains, VS Code）
- 自动化脚本
- 数据分析
```

---

## 深入阅读

- `commands.ts` — 中央路由器（~25K 行）
- `commands/` — 所有命令的实现
- `cli/handlers/` — 输出处理程序
