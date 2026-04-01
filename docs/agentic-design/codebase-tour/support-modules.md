---
title: 支持模块
layout: default
nav_order: 3
parent: Codebase Tour
grand_parent: Agentic Design Overview
---

# Support Modules：基础设施和工具

这一章涵盖支撑 Claude Code 运行的各种基础设施：权限系统、多 agent 协调、后台任务管理、用户内存、以及与外部系统的集成。

---

## `utils/permissions/` — 权限系统（深递归）

这是整个 agent sandbox 的防线，共 20+ 个文件，总计 300+ KB：

```
utils/permissions/
├── permissions.ts              [52 KB] 核心决策逻辑
├── yoloClassifier.ts           [52 KB] ML 自动审批分类器
├── filesystem.ts               [62 KB] 文件系统安全规则
├── PermissionMode.ts           Permission mode 枚举
├── denialTracking.ts           追踪拒绝次数，circuit breaker
├── pathValidation.ts           路径安全（Unicode normalization 等）
├── permissionRuleParser.ts     解析 "Bash(git *)" 这样的规则
├── classifierDecision.ts       分类器决策包装
├── bashClassifier.ts           Bash 命令的特殊规则
├── permissionExplainer.ts      用 LLM 解释权限决定
└── ... (其他辅助)
```

**核心 pipeline**：mode → rules → YOLO classifier → prompt user

详细的权限系统设计，请阅读 [Permission System 文档](../permission-system/)。

---

## `utils/` — 其他公用工具（部分列举）

```
utils/
├── permissions/                [见上面的深递归]
├── messages/
│   ├── mappers.ts             SDK 消息格式转换
│   ├── systemInit.ts          系统初始化消息构建
│   └── ...
├── settings/                   设置加载和应用
├── queryHelpers.ts             Query 辅助函数
├── undercover.ts               隐藏内部代号（Capybara, Tengu 等）
├── abortController.ts          Abort signal 管理
├── agentContext.ts             Agent 上下文 getter
├── api.ts                       API 工具函数
├── hooks/                       React hooks （不是本目录）
├── ... (数十个其他)
```

**学习重点**：`permissions/` 最复杂，其次是 `messages/` 和 `settings/`。

---

## `coordinator/` — 多 Agent 协调

```
coordinator/
└── coordinatorMode.ts          [19 KB] 唯一的文件，包含完整 coordinator 系统提示

功能：启用 CLAUDE_CODE_COORDINATOR_MODE=1 时激活，引入 4 个阶段：
  research → synthesis → implementation → verification
```

**关键点**：Worker 通过 `<task-notification>` XML 向 coordinator 报告进度。

更多关于多 agent 协调的内容，请阅读 [Multi-Agent 文档](../multi-agent/)。

---

## `tasks/` — 后台任务管理

```
tasks/
├── LocalAgentTask/             本地 agent 任务追踪
├── RemoteAgentTask/            远程 agent 任务追踪（CCR）
├── LocalMainSessionTask.ts     主 session 的任务
├── LocalShellTask/             Shell 任务（tmux/iTerm2）
├── InProcessTeammateTask/      In-process teammate 任务
├── DreamTask/                  Memory dream 任务
├── stopTask.ts                 停止任务的逻辑
├── types.ts                    Task 的类型定义
└── pillLabel.ts                任务标签（UI 展示）
```

**核心概念**：每个后台任务都有生命周期追踪。

---

## `memdir/` — 用户 Memory 管理

```
memdir/
├── memdir.ts                   [21 KB] 核心 memory 目录管理
├── memoryScan.ts              扫描 memory 文件
├── memoryTypes.ts             Memory 文件的类型
├── memoryAge.ts               Memory 的年龄计算
├── paths.ts                    Memory 文件路径管理
├── teamMemPaths.ts            Team memory 路径（multi-agent）
└── findRelevantMemories.ts    查找相关 memory
```

**用途**：管理 `~/.claude/memory/` 目录，存储用户的持久化知识。

关于 Memory 系统和 autoDream，请阅读 [Context & Memory 文档](../context-memory/)。

---

## `constant/` — 常量和系统提示

```
constant/ (也被称为 constants/)
├── system.ts                   Base system prompt
├── systemPromptSections.ts    System prompt 的可组装部分
├── cyberRiskInstruction.ts    安全指导（Safeguards team 维护）
├── betas.ts                    API beta features 列表
├── messages.ts                 常见消息文本
├── prompts.ts                  各种 prompt 模板
└── ... (其他常量)
```

**关键**：System prompt 是模块化的，不同 feature 会添加不同的部分。

---

## `bridge/` — 与 claude.ai 的 JWT 连接

```
bridge/
├── bridgeApi.ts               与 claude.ai 的通信
├── bridgeMessaging.ts         消息协议
├── remoteBridgeCore.ts        远程 bridge 核心
├── createSession.ts           创建远程 session
├── jwtUtils.ts                JWT 令牌工具
├── trustedDevice.ts           受信设备管理
└── ... (10+ 个其他)
```

**用途**：BRIDGE_MODE 时，CLI 可以通过 claude.ai 的 WebUI 远程控制。

---

## `remote/` — 远程 Session 管理

```
remote/
├── RemoteSessionManager.ts    远程 session 生命周期
├── SessionsWebSocket.ts       WebSocket 连接管理
├── remotePermissionBridge.ts  权限同步
├── sdkMessageAdapter.ts       SDK 消息适配
└── ... (其他)
```

**用途**：支持多个远程客户端同时连接到同一个服务。

---

## `entrypoints/sdk/` — SDK 集成

```
entrypoints/sdk/
├── agentSdkTypes.ts           给外部程序使用的类型
└── (其他 SDK 特定代码)
```

**用途**：让程序员可以用代码调用 Claude Code，而不只是 CLI。

---

## 递归粒度停止条件

以下类型的文件不再递归深入：

- **单功能工具函数** (<100 行，做一件明确的事)：如 `array.ts`, `api.ts` 的某些导出
- **简单 enum/type 定义**：如 `types.ts`, `*.d.ts`
- **单个小的 React Hook**：如 `useAfterFirstRender.ts`
- **Utility 映射和适配器**：如 message format converter

这些文件的学习价值在于"知道它存在"，而不是理解其内部细节。

---

## 深入阅读

- `utils/permissions/permissions.ts` — 权限决策的核心逻辑
- `utils/permissions/yoloClassifier.ts` — ML 分类器实现
- `memdir/findRelevantMemories.ts` — Memory 相关性搜索
- `coordinator/coordinatorMode.ts` — Coordinator 系统提示
