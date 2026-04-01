---
title: 终端 UI
layout: default
nav_order: 10
parent: Agentic Design Overview
has_children: true
---

# Terminal UI：声明式终端渲染

Claude Code 是一个 CLI，但它的终端 UI 不是简单的"打印文本"，而是用 **React + Ink** 实现的完整组件系统。这使得终端中可以有：

- 实时进度条
- 交互式菜单和对话框
- 着色的日志输出
- 自动排版和换行

## 为什么选择 React + Ink？

### 问题：原始 readline

```javascript
// 传统 CLI（readline 库）
console.log("工作进度：0%")
// 用户输入
await getUserInput("> ")
console.log("工作进度：50%")
console.log("工作进度：100%")

// 问题：
// 1. 输出会被混乱的日志淹没
// 2. 进度条无法更新（只能追加新行）
// 3. 多个并行 agent 的输出互相干扰
```

### 方案：React 组件树

```javascript
// React + Ink （当前做法）
<App>
  <AgentProgress agentId="A" />
  <AgentProgress agentId="B" />
  <InputField />
  <OutputPane />
</App>

// 优势：
// 1. 状态驱动：AppState 变化 → 组件重新渲染
// 2. 组件隔离：每个 agent 有自己的进度组件
// 3. 响应式排版：终端宽度变化时自动调整
```

---

## 架构概览

```
AppStateStore (Zustand)
    ↓ state changes
    ↓
React Component Tree (Ink)
    ├── AgentProgressLine
    ├── MessagePane
    ├── InputField
    └── ...
    ↓
cli/print.ts (output funnel)
    ↓
Terminal Rendering
```

三层分离：
1. **State** — 用 Zustand 管理全局状态
2. **Components** — React 组件描述 UI 逻辑
3. **Rendering** — cli/print.ts 负责最终输出

---

## 关键概念

### 声明式 vs 命令式

```javascript
// 命令式（传统 readline）
term.moveCursor(0, -5)      // 移到第 5 行
term.clearLine()            // 清空
term.write("进度：50%")     // 写入

// 声明式（React + Ink）
<ProgressBar progress={50} />
// 框架负责所有细节
```

### Ink 的渲染模型

```
每个状态变化：

AppState 更新（如：progress: 0% → 50%）
  ↓
React 重新渲染所有组件
  ↓
Ink 计算新的 VTree（虚拟终端树）
  ↓
与前一个 VTree 对比（diff）
  ↓
只发送差异给终端（最小化重绘）
```

---

## 三个关键系统

### 1. AppStateStore — 单一真实来源

所有 UI 数据都来自这里：
- 当前 agent 状态
- 消息历史
- 输入框内容
- 进度信息

### 2. cli/print.ts — 输出漏斗

所有文本输出（日志、消息、错误）都通过这里。这使得：
- 可以统一格式化
- 可以支持多种输出后端（stdout、HTTP、文件）
- 可以生成 NDJSON 结构化输出

### 3. useTypeahead — 交互智能化

useTypeahead Hook 提供：
- 斜杠命令自动补全
- 文件路径补全
- 命令历史导航

---

## 导航

- **[Ink Rendering Engine](ink-rendering.md)** — cli/print.ts 如何工作
- **[State → UI Pipeline](state-to-ui.md)** — AppStateStore 如何驱动 UI
- **[Typeahead & Input](typeahead.md)** — 交互和补全系统

---

**下一步** → [Ink Rendering](ink-rendering.md)
