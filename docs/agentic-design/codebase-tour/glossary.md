---
title: 术语表
layout: default
nav_order: 5
parent: Codebase Tour
grand_parent: Agentic Design Overview
---

# 术语表

Claude Code 文档中的 40+ 核心术语定义。

---

## A

**AbortController / Abort Signal**
通过 AsyncLocalStorage 传播的取消信号，用来停止 agent 和工具的执行。来自：用户按 Ctrl+C、sibling 工具失败、权限熔断。
→ 详见：[Error Handling → Abort Architecture](../error-handling/abort-architecture.md)

**autoDream**
后台记忆巩固服务，每 24 小时自动运行一次，将 session 消息压缩并提炼到 MEMORY.md。用户无需介入。
→ 详见：[Context & Memory → autoDream](../context-memory/auto-dream.md)

**AppStateStore**
Zustand 状态管理库，是终端 UI 的唯一真实来源。包含所有 UI 数据：当前 agent、消息、输入框内容、进度。
→ 详见：[Terminal UI → State → UI Pipeline](../terminal-ui/state-to-ui.md)

---

## B

**Bridge Mode**
远程控制模式，允许从 claude.ai WebUI 对本地 CLI 进行双向同步。通过 JWT 认证和 WebSocket。
→ 详见：[Bridge & Remote](../bridge-remote/)

**Bootstrap**
Claude Code 启动时的初始化过程（56 KB 代码）。包括环境检测、session 恢复、系统 prompt 组装、GrowthBook flag 加载、MCP 连接。
→ 详见：[Session State → Bootstrap](../session-state/bootstrap.md)

---

## C

**Cache Boundary**
系统 prompt 中的标记（<!-- CACHE_BOUNDARY -->），分割静态可缓存部分和动态不可缓存部分。静态部分命中 Prompt Cache 后，可获得 90% 的 token 折扣。
→ 详见：[Advanced → Prompt Cache](../advanced/prompt-cache.md)

**CCR (Cloud Container Runtime)**
Anthropic 的云容器运行时环境，用于运行 Remote agents。支持 30 分钟长时间运行（ULTRAPLAN）。
→ 详见：[Multi-Agent → Agent Types](../multi-agent/agent-types.md)

**Circuit Breaker**
故障转移机制。权限拒绝 3 次 → 熔断打开，回退到手动确认模式。API 连续失败 N 次 → 返回错误给用户。
→ 详见：[Error Handling → Circuit Breakers](../error-handling/circuit-breakers.md)

**cli/print.ts**
输出漏斗（212 KB），所有终端输出（日志、消息、错误）的统一入口。支持多种输出后端（stdout、HTTP、文件）和 NDJSON 结构化输出。
→ 详见：[Terminal UI → Ink Rendering](../terminal-ui/ink-rendering.md)

**Compaction**
Context window 压缩。三层策略：MicroCompact（快速，删除格式）→ AutoCompact（删除非关键消息） → SnipCompact（极端，删除整个消息块）。
→ 详见：[Context & Memory → Compression](../context-memory/compression.md)

**Context Window**
Claude 模型的输入上下文大小（100K tokens）。system prompt + 消息历史必须在此范围内。用完 → token budget 触发压缩。
→ 详见：[Context & Memory](../context-memory/)

**Coordinator Mode**
多 agent 协调模式。Coordinator agent 指挥多个 worker agent 并行执行研究、综合、实施、验证四个阶段，通过 Tengu Scratchpad 共享中间结果。
→ 详见：[Multi-Agent → Coordinator Mode](../multi-agent/coordinator-mode.md)

---

## D

**Deferred Tool**
标记为 `shouldDefer: true` 的工具，默认不出现在 system prompt。通过 ToolSearchTool 明确搜索后才激活。约 18 个工具被延迟（如 CronCreateTool）。
→ 详见：[Tool System → Deferred Tools](../tool-system/deferred-tools.md)

**Denial Tracking**
权限拒绝追踪。用户连续拒绝 3 次同类工具请求 → 熔断打开，自动降级到 acceptEdits 模式（每次都提示）。
→ 详见：[Permission System → Modes](../permission-system/modes.md)

---

## E

**Error Handling**
四大失败模式的处理：Permission Denials、Tool Execution Failures、API Errors、Resource Exhaustion。核心机制：Abort Signal、Circuit Breaker、Graceful Degradation。
→ 详见：[Error Handling](../error-handling/)

---

## F

**Feature Flag**
功能开关，分为编译期（Bun 的 `feature()` API）和运行期（GrowthBook）。隐藏实验性功能（ULTRAPLAN、KAIROS、CHICAGO）。
→ 详见：[Advanced → Feature Gating](../advanced/feature-gating.md)

---

## G

**Graceful Degradation**
功能不可用时的降级策略。YOLO 模式失败 → 回退到提示模式。Compaction 失败 → 限制新请求大小但继续运行。远程 agent 超时 → 强制杀死清理。
→ 详见：[Error Handling → Recovery](../error-handling/recovery-strategies.md)

**GrowthBook**
A/B 测试和 feature flag 服务，管理运行期功能开关（ULTRAPLAN、KAIROS 等）。在 bootstrap 时初始化。
→ 详见：[Session State → Bootstrap](../session-state/bootstrap.md)

---

## J

**JSONL**
JSON Lines 格式，每行一个 JSON 对象，用于 session 消息历史持久化。支持流式读取和追加，不需要重写整个文件。
→ 详见：[Session State → Persistence](../session-state/persistence.md)

**JWT (JSON Web Token)**
Bridge Mode 中的身份验证方案。客户端在 WebSocket 握手时提交 JWT，服务端验证签名和过期时间。
→ 详见：[Bridge & Remote → Bridge Protocol](../bridge-remote/bridge-protocol.md)

---

## K

**KAIROS**
高级 agent 模式：永驻助手，跨 session 保留 MEMORY.md，agent 知道距上次对话的时间，主动复述历史。
→ 详见：[Advanced → Agent Modes](../advanced/agent-modes.md)

---

## M

**MCP (Model Context Protocol)**
开放标准，允许 AI 工具与外部系统通过标准化接口交互。Claude Code 既是 MCP Server（暴露能力给 IDE 插件）又是 MCP Client（连接外部数据库、代码搜索等）。
→ 详见：[MCP Integration](../mcp/)

**MCPSearch Mode**
当 MCP 工具描述总字数 > context window 的 10% 时激活。使用 ToolSearchTool 按需发现工具，而不是全部放入 system prompt。
→ 详见：[Tool System → MCP Tools](../tool-system/mcp-tools.md)

**memdir / Memory Directory**
`~/.claude/memory/` 下的内存文件系统。包含 MEMORY.md（全局跨 session）和 session 特定的 memory 目录。autoDream 负责更新。
→ 详见：[Context & Memory → Memdir System](../context-memory/memdir-system.md)

---

## P

**Permission Mode**
权限决策方式：YOLO（自动批准低风险）、BYPASS（自动批准所有）、ACCEPTEDITS（提示每次）、MANUAL（总是停止）。降级按 YOLO → 提示 → MANUAL。
→ 详见：[Permission System → Modes](../permission-system/modes.md)

**Prompt Cache**
Claude API 的功能，使 system prompt 的静态部分（工具列表、权限规则）可跨请求复用。缓存命中后 token 费用 90% 折扣，实现 50% 的成本节省。
→ 详见：[Advanced → Prompt Cache](../advanced/prompt-cache.md)

---

## Q

**Query Loop**
核心 agent 循环（query/query.ts）。每轮：发送消息 + system prompt 到 Claude API → 接收流式响应 → 执行工具 → 反馈给 Claude → 下一轮。持续直到 agent 停止。
→ 详见：[Agent Loop](../agent-loop/)

---

## R

**Remote Agent**
在 Cloud Container Runtime (CCR) 中运行的 agent，支持 30 分钟长时间运行（ULTRAPLAN）。与本地 agent 通过 MCP 协议通信。
→ 详见：[Multi-Agent → Agent Types](../multi-agent/agent-types.md)

---

## S

**SnipCompact**
最激进的 compaction 策略：删除整个消息块（通常是古老的用户消息或工具结果）以快速释放空间。代价：丢失历史。
→ 详见：[Context & Memory → Compression](../context-memory/compression.md)

**SkillTool / Skill Commands**
用户定义的命令，Markdown 文件存在 `~/.claude/commands/`。包含 YAML 前置（名称、描述、工具列表）+ prompt 正文。触发后创建专用 agent 执行。
→ 详见：[Slash Commands → Skill Commands](../slash-commands/skill-commands.md)

**Streaming**
工具执行的流式输出模式。工具可以边计算边返回（AsyncGenerator），Claude Code 即时反馈给 agent，加快交互。
→ 详见：[Agent Loop → Streaming](../agent-loop/streaming.md)

**System Reminder**
动态注入 system prompt 顶部的短消息，每轮 query 都被添加。包含：当前日期/时间、工作目录、最近 N 个 slash 命令、用户的偏好设置。
→ 详见：[Agent Loop → Stop Hooks](../agent-loop/stop-hooks.md)

---

## T

**Team Memory**
多 agent 协调模式下的共享记忆。Coordinator 和 workers 可读写 `team-memory/` 目录中的共享文件（带 file lock 防竞态）。
→ 详见：[Multi-Agent → Team Memory](../multi-agent/team-memory.md)

**Tengu Scratchpad**
Coordinator 模式的共享工作台。Worker agents 可在此写中间结果（project structure 分析、测试报告等），Coordinator 读取并综合。
→ 详见：[Multi-Agent → Coordinator Mode](../multi-agent/coordinator-mode.md)

**Token Budget**
Token 使用量追踪和触发机制。达到 80% → AutoCompact，92% → SnipCompact，95% → 拒绝新请求，100% → 返回 413 错误。
→ 详见：[Agent Loop → Token Budget](../agent-loop/token-budget.md)

---

## U

**ULTRAPLAN**
高级 agent 模式：深度规划模式，时限 30 分钟（vs 普通 5 分钟），2 倍 token 预算，专属工具（PlanningTool、ResearchTool、ThinkingTool）。
→ 详见：[Advanced → Agent Modes](../advanced/agent-modes.md)

---

## Y

**YOLO Classifier**
机器学习分类器（"You Only Live Once"），决定 shell 命令是否"足够安全"可自动执行。分类器知道常见命令的风险等级（git diff = safe，git reset = risky）。
→ 详见：[Permission System → Rules](../permission-system/rules.md)

---

## Z

**Zustand**
轻量级状态管理库，用于 AppStateStore。提供 hook 式订阅和选择器模式，性能优于 Redux。
→ 详见：[Terminal UI → State → UI](../terminal-ui/state-to-ui.md)

---

**按字母表返回** → [首页](../index.md)
