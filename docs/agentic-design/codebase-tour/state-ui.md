---
title: 状态与 UI
layout: default
nav_order: 2
parent: 代码库全景
grand_parent: Agentic Design Overview
---

# State & UI：前端和交互层

这一章涵盖 Claude Code 的 React 状态管理、UI 组件、终端渲染、以及用户命令系统。

---

## `state/` — React 状态管理

```
state/
├── AppState.tsx                React 状态类型定义
├── AppStateStore.ts            [21 KB] Zustand-like 状态存储
├── onChangeAppState.ts         状态变化的 handler
├── selectors.ts                状态选择器
├── store.ts                    存储初始化
└── teammateViewHelpers.ts      Teammate 视图辅助
```

**模式**：这是 Claude Code 的全局状态树，UI 所有数据都来自这里。

---

## `hooks/` — 80+ 个 React Hook

重要的：

| Hook | 用途 |
|------|------|
| `useCanUseTool.tsx` | [40 KB] 权限检查 Hook（频繁调用） |
| `useGlobalKeybindings.tsx` | [31 KB] 全局键盘快捷键 |
| `useTypeahead.tsx` | [212 KB] 自动补全建议 |
| `useVoice.ts` | [45 KB] 语音输入集成 |
| `useInboxPoller.ts` | [34 KB] 后台轮询消息 |
| `useArrowKeyHistory.tsx` | [34 KB] 历史导航（上/下箭头） |
| `useReplBridge.ts` | [115 KB] REPL 集成 |

**学习策略**：跳过大部分 hook，只深入 `useCanUseTool` 和 `useTypeahead`。

---

## `bootstrap/state.ts` — 全局启动状态

```
bootstrap/
└── state.ts                    [56 KB]
```

内容：
- 当前 session ID 和持久化 flag
- 用户类型（ant/public）
- Feature flag 缓存
- KAIROS mode 检测
- 全局配置状态

**作用**：在 App 启动的最早时刻初始化所有全局状态。

---

## `cli/` — 终端输出和格式化

```
cli/
├── print.ts                    [212 KB] 终端渲染引擎
├── structuredIO.ts            NDJSON 和结构化输出
├── handlers/                   命令特定的输出 handler
├── transports/                 多种输出后端（stdout, HTTP 等）
├── remoteIO.ts                远程 IO
└── ... (其他)
```

**关键**：`print.ts` 是整个 Claude Code 的"输出黑洞"，所有文本最终都通过这里。这里的优化对用户体验有直接影响。

---

## `context/` — React Context Provider

```
context/
├── QueuedMessageContext.tsx    消息队列
├── mailbox.tsx                Agent 邮箱（inter-agent 消息）
├── modalContext.tsx            Modal 对话框
├── notifications.tsx           通知系统
├── overlayContext.tsx          浮层上下文
├── promptOverlayContext.tsx    提示浮层
├── stats.tsx                   统计数据
└── voice.tsx                   语音上下文
```

**作用**：提供 React component tree 范围内的全局上下文。

---

## `components/` — React UI 组件

```
components/
├── App.tsx                     顶层 app 组件
├── AgentProgressLine.tsx       Agent 进度显示
├── BaseTextInput.tsx           文本输入基础
├── ... (100+ 个其他组件)
```

**特点**：使用 Ink 库在终端中渲染 React 组件。这允许 Claude Code 在纯终端中实现复杂的 UI（如按钮、输入框、模态对话框）。

---

## `commands/` — Slash 命令（88 个）

```
commands/
├── commit.ts / commit-push-pr.ts    Git 相关命令
├── agent.ts                         Agent 管理命令
├── task.ts, tasks.ts                任务管理
├── config.ts                        配置命令
├── memory.ts, memory-dream.ts       Memory 管理
├── session.ts                       Session 管理
├── autofix-pr.ts                    PR 自动修复
├── ... (80+ 个其他)
```

**特点**：每个 command 都可以定制输出格式，支持 structured output (NDJSON)。这使得 Claude Code 可以被其他工具和脚本调用。

**核心路由**：在 `commands.ts` 中维护所有命令的中央注册表。

---

## 深入阅读

- `state/AppStateStore.ts` — 全局状态存储的实现
- `cli/print.ts` — 终端渲染引擎（212 KB 的复杂系统）
- `hooks/useCanUseTool.tsx` — 权限检查的 Hook 实现
- `components/App.tsx` — 顶层应用组件
