---
title: Multi-Client Architecture
layout: default
nav_order: 2
parent: Bridge & Remote
grand_parent: Agentic Design Overview
---

# Multi-Client Architecture：多客户端同步

同一个 Claude Code 实例可以有多个客户端连接（WebUI、本地 CLI、远程 CLI）。它们都共享相同的 session、message history 和权限状态。

---

## 客户端类型

### Local CLI

```
用户在本地终端运行：
  $ claude-code
  
特点：
  - stdin/stdout 通信
  - 唯一（只有一个）
  - 最高权限（无需 JWT）
```

### Remote CLI

```
用户在远端机器运行：
  $ claude-code --remote localhost:3000
  
特点：
  - WebSocket 通信
  - 可以有多个（多个远端机器）
  - 需要 JWT + device approval
```

### WebUI（claude.ai）

```
用户在 claude.ai 网站上，使用"控制 Claude Code CLI"功能
  
特点：
  - HTTPS + WebSocket
  - 通过 claude.ai 中继
  - 需要强认证（用户账户）
```

### SDK（程序化）

```
程序通过 SDK 调用：
  const code = new ClaudeCode({ mode: 'sdk' })
  const result = await code.query(messages)
  
特点：
  - 进程内通信
  - 自动认证（共享进程）
  - 用于集成和自动化
```

---

## 消息扇出（Message Fanout）

### 场景：User 输入来自 Client A

```
[Local CLI]  [Remote CLI]  [WebUI]
    │            │            │
    │            └────────┬────┘
    │                     │
    ├─────────────┬───────┘
                  │
            ┌─────▼────┐
            │  Session │
            └────┬─────┘
                 │
     ┌───────────┼───────────┐
     │           │           │
     ▼           ▼           ▼
 Message    Message    Message
  Store    Broadcast   Sync

① User 在 WebUI 上输入 "/commit"
② 消息发到 RemoteSessionManager
③ 服务端处理，更新 message history
④ 广播到所有客户端：
   - 本地 CLI 更新 AppState
   - 其他远程 CLI 接收消息
   - WebUI 看到新消息
```

### Conflict Resolution（冲突解决）

```
如果多个客户端同时输入（竞争条件）：

LocalCLI: "git commit"
RemoteCLI: "git push"  (几乎同时)

解决：
  ① 每条消息有 timestamp
  ② 服务端按 timestamp 排序
  ③ message history 是单调的（不会重新排序）
  ④ 所有客户端看到相同顺序
```

---

## 权限和隔离

### 权限委托

```
假设用户的权限模式是 "auto"（自动执行）

LocalCLI
  → 权限模式：auto
  → 可以自动执行任何工具
  ↓
RemoteCLI
  → 权限模式：降级为 "acceptEdits"（需要确认）
  → 为什么降级？因为远端不能被信任
  ↓
WebUI
  → 权限模式：降级为 "dontAsk"（完全自动，但记录）
  → 通过 claude.ai 的中继，有额外验证
```

这是一个**信任链**的实现。

### 操作隔离

```
RemoteCLI 执行一个命令
  ↓
影响只限于该 session
  ↓
不会：
  - 修改用户文件系统上的其他项目
  - 访问其他用户的数据
  - 跨 session 污染（memory/state）
```

---

## 远程 Session 管理（RemoteSessionManager）

### Session 的创建

```
客户端连接时：

① 检查是否有现存活跃 session
   ↓ 有 → 加入（share 该 session）
   ↓ 无 → 创建新 session

② 为客户端分配唯一 clientId
   clientId = UUID()

③ 记录：client A 已加入 session XYZ

④ 从 session 加载消息历史，发给客户端
```

### Session 的销毁

```
所有客户端都断开时：

① RemoteSessionManager 检测到：no clients
② 开始倒计时（5 分钟）
③ 如果倒计时期间有新客户端：重置倒计时
④ 如果倒计时结束：销毁 session
   - 保存到 session archive
   - 释放内存资源
```

---

## 网络拓扑

### 本地模式（单机）

```
┌──────────────┐
│ Local CLI    │
│ (stdin/out)  │
│              │
│ Query Loop   │
│ Tool Exec    │
│ AppState     │
└──────────────┘
```

### 远程模式（分布式）

```
┌─────────────┐         ┌───────────────────┐
│  Remote CLI │         │   Claude Code     │
│  (Client A) │────────→│   (Server)        │
└─────────────┘   WS    │                   │
                        │  RemoteSession    │
┌─────────────┐         │  Manager          │
│  Remote CLI │         │                   │
│  (Client B) │────────→│  Message History  │
└─────────────┘   WS    │  AppState Store   │
                        │  Query Loop       │
┌─────────────┐         │  Tool Exec        │
│  WebUI      │         │                   │
│ (claude.ai) │────────→│                   │
└─────────────┘   HTTPS │                   │
                        └───────────────────┘
```

---

## 设计决定专栏

### 为什么远程客户端的权限模式要降级？

**不降级**：
```
LocalCLI: 权限模式 "auto"
RemoteCLI: 权限模式也是 "auto"

问题：
- RemoteCLI 可能被攻击者控制（网络被截获）
- 即使有 JWT，网络端点也可能被伪造
- 用户没有物理确认（与 local CLI 不同）
```

**降级**（当前做法）：
```
LocalCLI: "auto"（用户物理在场）
RemoteCLI: "acceptEdits"（降级，需要确认）
WebUI: "dontAsk"（完全自动，但有 claude.ai 的签名）

权衡：
- LocalCLI 可以快速自动执行
- RemoteCLI 仍然可用，但加了提示（防止误操作）
- WebUI 通过中继，增加信任
```

### 为什么要有倒计时销毁 Session？

**立即销毁**：
```
最后一个客户端断开
  ↓ 立即销毁 session
  ↓ 用户重新连接：需要重新加载历史
  ↓ 浪费时间
```

**倒计时销毁**（当前做法）：
```
最后一个客户端断开
  ↓ 等待 5 分钟
  ↓ 如果用户在 5 分钟内重新连接：seamless 恢复
  ↓ 如果 5 分钟后还没人连：销毁，释放内存
```

优势：大多数情况用户会立即重新连接，避免重新加载。

---

## 深入阅读

### 相关文档

- [Session Persistence](../session-state/persistence.md) — 客户端断开后的 session 保留
- [Permission Modes](../permission-system/modes.md) — 权限在 bridge 中的降级

### 源代码

- `remote/RemoteSessionManager.ts` — 远程 session 管理
- `remote/SessionsWebSocket.ts` — WebSocket 服务器实现
- `remote/remotePermissionBridge.ts` — 权限同步
- `remote/sdkMessageAdapter.ts` — SDK 消息适配

---

**下一步** → [Slash Commands](../slash-commands/)
