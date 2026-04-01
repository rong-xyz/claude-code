---
title: 端到端数据流
layout: default
nav_order: 4
parent: Codebase Tour
grand_parent: Agentic Design Overview
---

# 端到端数据流

本页追踪两条典型请求的完整路径，将文档中的 13 个独立章节串联成一张完整的地图。

---

## Flow 1：用户输入 `/commit`

从键盘按键到 git commit 完成，经过所有系统层：

```
① 用户在终端按下 "/" 键
② 用户输入 "commit" 并回车
```

### 第一层：Terminal UI — 捕获输入

**文档**：[Terminal UI](../terminal-ui/)

```
useTypeahead Hook 监听键盘事件
  ↓ 检测到 "/" 前缀 → 激活命令补全模式
  ↓ 用户输入 "co" → fuzzy match → 候选：/commit, /config
  ↓ 用户回车确认 "/commit"
  ↓ AppStateStore 更新：{ inputValue: "/commit" }
  ↓ cli/print.ts 将输入显示在终端
```

### 第二层：Slash Commands — 路由到命令

**文档**：[Slash Commands](../slash-commands/)

```
commands.ts 中央路由器接收 "/commit"
  ↓ 正则匹配：/^\/commit\b/ → 命中
  ↓ 加载命令定义：
      systemPrompt: "You are a Git commit expert..."
      tools: [BashTool, FileReadTool]
  ↓ 创建临时 agent，装备命令的 systemPrompt 和 tools
```

### 第三层：Agent Loop — 进入 query 循环

**文档**：[Agent Loop](../agent-loop/)

```
query() 函数开始：
  ↓ 组装请求：
      systemPrompt（静态 + 命令 prompt）
      tools（BashTool, FileReadTool）
      messages（当前 session 历史）
  ↓ 发送到 Claude API（带 cache_control 标头）
  ↓ 开始流式接收响应
  ↓ System Reminder 在每轮开始注入（当前时间、工作目录等）
```

### 第四层：Tool System — 并发执行工具

**文档**：[Tool System](../tool-system/)

Claude 决定调用多个工具：

```
Claude 输出：
  <tool_use>FileReadTool { path: "src/*.ts" }</tool_use>
  <tool_use>BashTool { command: "git diff --staged" }</tool_use>

StreamingToolExecutor 并发执行：
  ↓ FileReadTool 开始（读取文件）
  ↓ BashTool 开始（git diff）
  ↓ 两者并行运行
```

### 第五层：Permission System — 每次调用前检查

**文档**：[Permission System](../permission-system/)

```
BashTool 请求执行 "git diff --staged"：
  ↓ 查询规则库：是否有匹配规则？
      "git diff" 是只读命令 → 规则：safe
  ↓ isSafe? → true（无副作用）
  ↓ 自动批准，无需用户确认

BashTool 请求执行 "git commit -m '...'"：
  ↓ 规则库：无精确匹配
  ↓ YOLO 分类器：判断是否"安全"
      → "git commit" 被分类为中风险
  ↓ 弹出权限提示：
      "Claude 想要运行：git commit -m 'feat: add feature'
       允许 / 拒绝 / 总是允许"
```

### 第六层：Error Handling — 处理失败

**文档**：[Error Handling](../error-handling/)

```
假设 git commit 失败（没有 staged 文件）：
  ↓ BashTool 返回 exit code 1
  ↓ 错误结果送回 agent
  ↓ Agent 分析："没有 staged 文件"
  ↓ Agent 决定先运行 "git add -A"
  ↓ 新一轮权限检查 → 用户批准
  ↓ git add 成功 → 再次尝试 git commit
```

### 第七层：Context Memory — 监控 token 用量

**文档**：[Context & Memory](../context-memory/)

```
每次 API 响应后，token budget 更新：
  ↓ 当前使用量：42% → 正常
  ↓ 如果到 80%：触发 AutoCompact
  ↓ 压缩旧消息，保留关键内容
  ↓ 继续 query loop
```

### 第八层：Terminal UI — 流式显示输出

**文档**：[Terminal UI → Ink Rendering](../terminal-ui/ink-rendering.md)

```
Claude 的每个 token 到达：
  ↓ cli/print.ts 格式化（着色、缩进）
  ↓ Ink 重新渲染受影响的组件
  ↓ 只更新差异部分（最小化终端重绘）
  ↓ 用户实时看到输出流
```

### 第九层：Session State — 持久化历史

**文档**：[Session State](../session-state/)

```
query 完成后：
  ↓ 本轮消息（用户 + assistant + tool results）
  ↓ 追加到 ~/.claude/sessions/<session-id>.jsonl
  ↓ 每行一个 JSON 对象（JSONL 格式）
  ↓ 下次 session 启动时可恢复
```

### 第十层：Bridge Remote — 同步到其他客户端

**文档**：[Bridge & Remote](../bridge-remote/)

```
如果用户同时在 claude.ai WebUI 上查看：
  ↓ 每条输出消息通过 WebSocket 扇出
  ↓ RemoteSessionManager 广播给所有连接的客户端
  ↓ WebUI 实时显示 git commit 进度
```

### 完整链路图

```
键盘输入 /commit
  │
  ▼
[Terminal UI] ──用户输入──▶ [Slash Commands] ──命令定义──▶ [Agent Loop]
                                                                │
                                              ◀──工具调用请求──┘
                                              │
                              ┌───────────────┴───────────────┐
                              ▼                               ▼
                         [Tool System]               [Permission System]
                         并发执行工具                  每次调用前检查
                              │                               │
                              └───────────────┬───────────────┘
                                              │
                              ┌───────────────┴───────────────┐
                              ▼                               ▼
                        [Error Handling]            [Context Memory]
                         失败恢复/重试               token 监控/压缩
                              │                               │
                              └───────────────┬───────────────┘
                                              ▼
                                        [Agent Loop]
                                        继续下一轮
                                              │
                              ┌───────────────┴───────────────┐
                              ▼                               ▼
                       [Terminal UI]                  [Session State]
                       流式输出渲染                    持久化 JSONL
                              │                               │
                              └───────────────┬───────────────┘
                                              ▼
                                       [Bridge Remote]
                                       广播给所有客户端
```

---

## Flow 2：多 Agent Coordinator 任务

追踪 `"帮我重构这个 monorepo"` 通过 Coordinator 模式的路径：

**文档**：[Multi-Agent](../multi-agent/)，[Context & Memory](../context-memory/)，[Error Handling](../error-handling/)

```
用户：/ultraplan 帮我重构这个 monorepo
  │
  ▼
[Slash Commands] 路由到 ULTRAPLAN 命令
  ↓ 创建 Coordinator agent（30 分钟时限）
  │
  ▼
[Agent Loop - Coordinator]
  ↓ Coordinator 制定研究计划
  ↓ 生成 3 个 Worker 任务
  │
  ├──────────────────────────────┐
  ▼                              ▼
[Agent Loop - Worker 1]   [Agent Loop - Worker 2]
探索 package.json          探索 src/ 目录
  │                              │
  ▼                              ▼
[Tool System] FileReadTool  [Tool System] GlobTool
  │                              │
  ▼                              ▼
[Permission System]         [Permission System]
只读 → 自动批准              只读 → 自动批准
  │                              │
  └──────────────┬───────────────┘
                 ▼
         Workers 写入 Tengu Scratchpad
         Coordinator 等待所有 Worker 完成
                 │
                 ▼
         [Context Memory] 检查 token 用量
         如果 > 80%：压缩 Worker 报告
                 │
                 ▼
         [Agent Loop - Coordinator] Synthesis 阶段
         一次性读完所有 Worker 报告
         生成重构计划
                 │
                 ▼
         实施阶段：新一批 Workers 并行执行修改
         [Error Handling] 任一 Worker 失败 → sibling abort
                 │
                 ▼
         验证阶段：运行测试
                 │
                 ▼
         [Session State] 完整过程持久化
         [Bridge Remote] 实时同步到 WebUI
```

---

## 关键设计原则

通过以上两条 Flow 可以看到 Claude Code 的三个核心设计原则：

1. **每一层都可以独立失败，不影响其他层**
   - Tool System 失败 → Error Handling 处理，不崩溃整个 session
   - Context Memory 压缩 → 对 Agent Loop 透明

2. **状态集中管理，数据单向流动**
   - AppStateStore 是 UI 的唯一真实来源
   - JSONL session 文件是历史的唯一真实来源

3. **扩展点都是标准接口**
   - 新工具：实现 `Tool<T>` 接口
   - 新命令：添加到 commands.ts 路由表
   - 新 MCP 服务器：注册到 `.mcp.json`

---

## 深入阅读

每个步骤对应的文档章节：

| 层 | 文档 |
|---|---|
| 终端输入 | [Terminal UI → Typeahead](../terminal-ui/typeahead.md) |
| 命令路由 | [Slash Commands → Command Routing](../slash-commands/command-routing.md) |
| Query Loop | [Agent Loop](../agent-loop/) |
| 工具执行 | [Tool System → Tool Interface](../tool-system/tool-interface.md) |
| 权限检查 | [Permission System → Modes](../permission-system/modes.md) |
| 错误恢复 | [Error Handling → Recovery](../error-handling/recovery-strategies.md) |
| Token 管理 | [Context Memory → Compression](../context-memory/compression.md) |
| 输出渲染 | [Terminal UI → Ink Rendering](../terminal-ui/ink-rendering.md) |
| 历史持久化 | [Session State → Persistence](../session-state/persistence.md) |
| 多客户端同步 | [Bridge Remote → Multi-Client](../bridge-remote/multi-client.md) |
| MCP 工具 | [MCP Integration](../mcp/) |
