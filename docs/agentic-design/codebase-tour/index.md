---
title: Codebase Tour
layout: default
nav_order: 2
parent: Agentic Design Overview
has_children: true
---

# 代码库全景：从顶层到最深处

> 这篇文档是一个**递归的导游**。我们从顶层文件开始，逐层深入每个重要的目录，直到粒度太细而没有学习价值为止。每个章节都注明关键文件，你可以随时打开源代码对照。

---

## 顶层：核心文件

在 `/home/user/claude-code/` 的根目录下，以下文件是系统的脊梁：

| 文件 | 行数 | 用途 |
|------|------|------|
| `main.tsx` | ~4700 | CLI 入口，React + Ink 终端渲染器 |
| `query.ts` | ~1700 | Query loop 状态机的完整实现 |
| `QueryEngine.ts` | 核心 | Query 执行的编排引擎 |
| `Tool.ts` | 核心 | Tool 的类型定义和接口规范 |
| `tools.ts` | 核心 | Tool 注册与初始化 |
| `context.ts` | 核心 | System prompt 和 context 的组装 |
| `commands.ts` | ~700 | 所有 slash command 的中央路由 |

**学习路线**：`main.tsx` → `query.ts` → `Tool.ts` 可以了解 agent loop 的完整流程。

---

## 学习路线建议

1. **快速扫描**：从本文开始，看一眼代码库全景。
2. **焦点学习**：根据你的兴趣选择深入：
   - 想理解 agent loop? → `query.ts`, `QueryEngine.ts`, 然后阅读 [Agent Loop 文档](../agent-loop/)
   - 想理解 tool 系统? → `Tool.ts`, `tools/`, `services/tools/`, 然后阅读 [Tool 系统文档](../tool-system/)
   - 想理解权限? → `utils/permissions/`, 然后阅读 [Permission System 文档](../permission-system/)
   - 想理解多 agent? → `tools/AgentTool/`, `coordinator/`, 然后阅读 [Multi-Agent 文档](../multi-agent/)
3. **对照原文**：打开相应的源文件，参照下面各章节的路线图逐个浏览。

---

## 核心数据流

```
User Input
    ↓
entrypoints/cli.tsx (React 初始化)
    ↓
query.ts (Query Loop: QUERY → TOOL_USE → RESULT)
    ↓
Tool.ts & services/tools/ (Tool 执行)
    ↓
utils/permissions/ (权限检查) + BashTool/FileEditTool/... (实际执行)
    ↓
queryHelpers + messages (结果处理)
    ↓
state/ (状态更新)
    ↓
components/ + cli/print.ts (UI 渲染)
    ↓
User sees output
```

---

## 关键文件大小排名

```
Top 15 largest files (最值得读):

1. query.ts                 ~1700 行
2. services/tools/StreamingToolExecutor.ts
3. utils/permissions/permissions.ts    ~52 KB
4. utils/permissions/yoloClassifier.ts ~52 KB
5. utils/permissions/filesystem.ts     ~62 KB
6. hooks/useTypeahead.tsx             ~212 KB
7. cli/print.ts                       ~212 KB
8. tools/AgentTool/AgentTool.tsx      ~233 KB
9. bootstrap/state.ts                 ~56 KB
10. state/AppStateStore.ts             ~21 KB
11. services/autoDream/autoDream.ts    ~11 KB
12. coordinator/coordinatorMode.ts     ~19 KB
13. hooks/useCanUseTool.tsx            ~40 KB
14. hooks/useGlobalKeybindings.tsx     ~31 KB
15. hooks/useInboxPoller.ts            ~34 KB
```

先读前 5 个，会 cover 80% 的系统复杂性。

---

## 导航说明

- **[Core Runtime](core-runtime.md)** — Query loop、Tool 系统、Service 层的核心实现
- **[State & UI](state-ui.md)** — React 状态管理、UI 组件、终端渲染、slash 命令
- **[Support Modules](support-modules.md)** — 权限系统、工具函数、coordinator、后台任务、记忆管理
