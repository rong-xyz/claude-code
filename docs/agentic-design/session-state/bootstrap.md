---
title: Bootstrap & Cold Start
layout: default
nav_order: 1
parent: Session 状态
grand_parent: Agentic Design Overview
---

# Bootstrap & Cold Start：系统启动的关键时刻

`bootstrap/state.ts` (56 KB) 是应用启动时的"入口点"，它初始化所有全局状态。

---

## 启动顺序

### 阶段 1：环境检测（0–100ms）

```
① 读取环境变量和配置文件
   $HOME/.claude/config.json
   $CLAUDE_CODE_MODE (如 KAIROS 模式)
   
② 检测运行环境
   - Ant user?（内部用户）
   - Public user?（公开用户）
   - Subscription level? (free/pro/max)
   
③ 加载 Feature Flags
   从 GrowthBook 获取运行时 flags
   tengu_new_memory_v2: true/false?
   chicago_beta: true/false?
```

### 阶段 2：会话初始化（100–500ms）

```
① 检查是否有上次的 session
   ~/.claude/sessions/
   最近的 session ID？
   
② 如果有：加载历史
   消息历史（最后 N 条）
   Memory.md（相关内容）
   任务状态
   
③ 如果没有：创建新 session
   生成 sessionId
   初始化空消息列表
   创建新 Memory.md（如果需要）
```

### 阶段 3：系统提示组装（500–800ms）

```
① 加载 Base System Prompt
   constants/system.ts (~2900 tokens)
   
② 添加模块化部分
   ├─ Tools 定义（所有 40+ 工具）
   ├─ Safeguards 指导（安全约束）
   ├─ Beta features 说明
   ├─ 特定模式的指导（coordinator, KAIROS 等）
   └─ 用户自定义提示词
   
③ 标记 Cache Boundary
   <cache-control>
     type: "ephemeral"
   </cache-control>
   ← 这行之前的内容可被 API 缓存
```

System prompt 总规模：~8000–12000 tokens

---

## 全局状态的初始值

```typescript
globalState = {
  // Session 信息
  sessionId: "abc-123",
  createdAt: "2026-04-01T13:45:00Z",
  
  // 用户信息
  userId: "user@example.com",
  userType: "public",  // or "ant"
  subscriptionLevel: "max",  // free, pro, max
  
  // Feature flags
  featureFlags: {
    "tengu_scratch": true,
    "chicago_beta": false,
    ...
  },
  
  // 权限模式
  permissionMode: "auto",  // default, auto, acceptEdits, bypass, dontAsk
  
  // 其他
  workingDirectory: "/home/user/project",
  kairosModeEnabled: false
}
```

---

## 性能优化

### Lazy Loading（延迟加载）

某些初始化可以推迟：

```
优先级：
  ① 必须立刻初始化：sessionId, permissionMode, base system prompt
  ② 可以延迟到首次使用：Tool 定义、GrowthBook flags
  ③ 可以后台加载：用户自定义技能、extended memory
```

### 缓存层

```
第一次启动：扫描 ~/.claude/ → 300ms
第二次启动：使用缓存 → 50ms
```

缓存包括：
- Session 列表
- 用户自定义命令
- 上次的权限规则

---

## Cold Start vs Warm Start

### Cold Start（首次启动）

```
系统启动 → 创建新 session → 空消息列表
  ↓ 用户输入第一个请求
  ↓ 系统提示 + 用户消息 → API
```

首次调用 API 时，发送完整的系统提示和请求。

### Warm Start（恢复 Session）

```
系统启动 → 检查到旧 session → 加载历史
  ↓ 用户输入新请求
  ↓ [历史消息缓存] + [新请求] + [系统提示缓存] → API
```

系统提示已被 API 缓存，只需发送新请求，节省 tokens。

---

## 特殊模式的启动

### KAIROS 模式（永驻助手）

```
bootstrap/state.ts 检测到：
  $KAIROS_MODE = 1
  
初始化：
  - 后台工作线程
  - 消息队列
  - 长期记忆加载
  ↓
Claude 进入"always-on"状态，后台监听任务
```

### Coordinator 模式

```
bootstrap/state.ts 检测到：
  feature("COORDINATOR_MODE") == true
  
初始化：
  - Coordinator 系统提示加载
  - 4 阶段工作流（research → synthesis → implementation → verification）
  - 任务队列
```

---

## 设计决定专栏

### 为什么要在 bootstrap 阶段加载 Feature Flags？

**在 API 时才获取**：
```
第一个 API 调用时获取 feature flags
问题：如果 flag 影响系统提示（如 coordinator mode），太晚了
```

**在 bootstrap 时获取**（当前做法）：
```
启动时立即从 GrowthBook 获取
  ↓ 决定要不要加载 coordinator 系统提示
  ↓ 决定权限模式
```

成本：启动慢 ~200ms（网络请求），但保证了正确的初始化。

### 为什么要标记 System Prompt 的 Cache Boundary？

**不标记**：
```
每个 API 调用都发送完整系统提示（~8K tokens）
100 个请求 → 800K tokens 浪费
成本 ≈ $8 / 100 requests
```

**标记 Cache Boundary**（当前做法）：
```
第一个请求：发送完整系统提示（API 缓存）
后续 99 个请求：直接用缓存，只发送新的部分
节省 = 8K * 99 = 792K tokens ≈ $8 / 100 requests
→ 50% 成本节省
```

---

## 深入阅读

- `bootstrap/state.ts` — 完整的启动逻辑（56 KB）
- `entrypoints/cli.tsx` — CLI 入口，调用 bootstrap
- `constants/system.ts` — Base system prompt
- `utils/settings/` — 配置加载逻辑

---

**下一步** → [Session Persistence](persistence.md)
