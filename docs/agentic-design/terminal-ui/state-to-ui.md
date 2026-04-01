---
title: 状态 → UI 流水线
layout: default
nav_order: 2
parent: 终端 UI
grand_parent: Agentic Design Overview
---

# State → UI Pipeline：单一真实来源

Claude Code 的 UI 是"状态驱动"的：所有显示内容都来自 AppStateStore，没有独立的 UI 状态。

---

## AppStateStore 的职责

### 全局状态树

```typescript
AppState = {
  agents: {
    "agent-A": {
      status: "running",
      message: "Reading file...",
      progress: 45,
      taskCount: 3,
      completedTaskCount: 1
    },
    "agent-B": {
      status: "idle",
      // ...
    }
  },
  
  messages: [
    { id: "msg-1", text: "Starting...", type: "info" },
    { id: "msg-2", text: "✓ File read", type: "success" }
  ],
  
  input: {
    value: "/com",  // 当前输入框内容
    suggestions: ["commit", "config"],  // 自动补全建议
    selectionIndex: 0
  },
  
  system: {
    tokenUsage: 45000,
    maxTokens: 100000,
    contextPressure: 0.45
  }
}
```

### 状态变化的流程

```
Agent 产生事件：
  "agent-A 完成了任务 1"
    ↓ 调用 AppState 的 setter
    ↓
AppState.agents["agent-A"].completedTaskCount = 2
    ↓ Zustand 检测到变化
    ↓ 通知所有订阅者（React 组件）
    ↓
React 重新渲染涉及的组件
    ↓
Ink 更新终端显示
```

---

## 选择器（Selectors）

### 为什么需要选择器？

直接访问 AppState：
```typescript
// 组件代码
<AgentProgressLine progress={appState.agents[props.agentId].progress} />

// 问题：
// 1. 每次 AppState 任何地方变化，都会重新渲染
// 2. 组件耦合到 AppState 的结构
// 3. 重构 AppState 时需要改所有组件
```

使用选择器：
```typescript
// selectors.ts
const selectAgentProgress = (state: AppState, agentId: string) => 
  state.agents[agentId].progress

// 组件代码
<AgentProgressLine progress={selectAgentProgress(appState, props.agentId)} />

// 优势：
// 1. 选择器是"订阅合约"——只在这个数据变化时重渲
// 2. 结构变化只需改选择器，不需改组件
// 3. 可以做计算优化（memoization）
```

### 常见选择器

| 选择器 | 返回值 | 用途 |
|--------|--------|------|
| `selectAllAgents(state)` | 所有 agent 列表 | AgentList 组件 |
| `selectAgentProgress(state, id)` | 单个 agent 进度 | AgentProgressLine 组件 |
| `selectMessages(state)` | 消息历史 | MessagePane 组件 |
| `selectInputValue(state)` | 输入框内容 | InputField 组件 |
| `selectInputSuggestions(state)` | 补全建议 | Typeahead 组件 |

---

## 组件树结构

```
<App>
  {/* 所有 agent 的进度 */}
  <AgentList>
    {agents.map(agent => (
      <AgentProgressLine
        agentId={agent.id}
        progress={selectAgentProgress(state, agent.id)}
        status={selectAgentStatus(state, agent.id)}
      />
    ))}
  </AgentList>
  
  {/* 消息历史 */}
  <MessagePane messages={selectMessages(state)} />
  
  {/* 输入框 + 建议 */}
  <InputField
    value={selectInputValue(state)}
    suggestions={selectInputSuggestions(state)}
  />
  
  {/* 系统信息 */}
  <SystemBar
    tokenUsage={selectTokenUsage(state)}
    contextPressure={selectContextPressure(state)}
  />
</App>
```

### 每个组件的职责

| 组件 | 输入 | 输出 |
|------|------|------|
| `AgentProgressLine` | agentId, progress | 进度条字符串 |
| `MessagePane` | messages 数组 | 彩色消息列表 |
| `InputField` | value, suggestions | 用户输入内容 |
| `SystemBar` | tokenUsage, maxTokens | 内存压力指示 |

---

## 更新路径（Update Path）

### 触发更新的来源

```
[1] Query Loop 进展
      update state: messages.add("Tool executing...")
      ↓
[2] Tool 完成
      update state: agents[id].completedTaskCount++
      ↓
[3] API 响应
      update state: tokenUsage = ...
      ↓
[4] 用户输入
      update state: input.value = ...
```

### 批量更新

```typescript
// 多个更新打包成一个事务
AppState.transaction(() => {
  state.agents["A"].progress = 50
  state.agents["A"].lastMessage = "Halfway there"
  state.system.tokenUsage += 500
})
// 只触发一次组件重渲，而不是三次
```

---

## 性能考虑

### 选择器的 Memoization

```typescript
// 不好的做法（每次都重新计算）
selectAgentProgress(state, id)
  // 如果依赖 state 的深层对象，会导致组件频繁重渲

// 好的做法
const selectAgentProgress = (state, id) => {
  return state.agents[id]?.progress ?? 0
}
// 使用 Zustand 的 shallow 比较
useShallow(selectAgentProgress)  // 只在值真的变化时更新
```

### 列表渲染优化

```typescript
<AgentList>
  {agents.map(agent => (
    <AgentProgressLine
      key={agent.id}
      agentId={agent.id}
      // 只传 id，组件内部用选择器获取详情
    />
  ))}
</AgentList>

// 这样，当某个 agent 的信息变化时，
// 只有那个 AgentProgressLine 重渲，其他的不动
```

---

## 设计决定专栏

### 为什么要有单一 AppStateStore，而不是每个组件一个本地状态？

**组件本地状态**：
```
AgentProgressLine 有自己的 progress 状态
MessagePane 有自己的 messages 状态

问题 1：多个组件需要同步（如：用户编辑消息 → 需要同步到 MessagePane）
问题 2：状态不一致（一个组件显示 progress=45，另一个显示 40）
问题 3：调试困难（需要查看多个组件的状态）
```

**单一真实来源**（当前做法）：
```
所有状态在 AppStateStore
  ↓ 任何组件需要状态，都从这里读
  ↓ 组件只是"视图"，不存储数据
优势：
- 数据一致性有保障
- 时间旅行调试（可以回放状态变化）
- 易于持久化（save/restore state）
```

成本：所有组件都要关联到 store，但这是值得的。

---

## 深入阅读

- `state/AppStateStore.ts` — Zustand 状态存储实现（21 KB）
- `state/selectors.ts` — 所有选择器定义
- `state/onChangeAppState.ts` — 状态变化的 handler
- `components/` — React 组件实现

---

**下一步** → [Typeahead & Input](typeahead.md)
