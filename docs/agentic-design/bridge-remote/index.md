---
title: Bridge & Remote
layout: default
nav_order: 12
parent: Agentic Design Overview
has_children: true
---

# Bridge & Remote：多客户端架构

Claude Code 可以通过多种方式被控制：

1. **Local CLI**：用户在终端直接交互
2. **Remote CLI**：通过 WebSocket 从远端控制
3. **WebUI**（claude.ai）：从 claude.ai 的 WebUI 远程控制 CLI
4. **SDK**：编程方式调用

---

## 什么是 Bridge Mode？

BRIDGE_MODE 允许从 claude.ai WebUI 对 CLI 进行 **远程控制** 和 **双向同步**。

```
用户在 claude.ai WebUI 上
  ↓ 输入消息
  ↓ WebUI 通过 JWT + WebSocket 连接到本地 Claude Code CLI
  ↓ CLI 执行工作
  ↓ CLI 输出流回 WebUI
  ↓ 用户在 WebUI 上看到实时进度
```

---

## 多客户端架构

```
Claude Code 作为服务端：

主进程（main session）
  ├── 本地 CLI 客户端
  │   └── stdin/stdout
  │
  ├── 远程客户端 1
  │   └── WebSocket → 远端计算机 A
  │
  ├── 远程客户端 2
  │   └── WebSocket → 远端计算机 B
  │
  └── WebUI 客户端
      └── HTTPS + JWT → claude.ai
```

所有客户端共享相同的 session 和 message history。

---

## 核心概念

### JWT Token（JSON Web Token）

```
Bridge 使用 JWT 进行身份验证：

┌─────────────────────┐
│ JWT Token           │
├─────────────────────┤
│ Header:  typ, alg   │
│ Payload: user, exp  │
│ Sig:     secret     │
└─────────────────────┘

client 在 WebSocket 握手时提交 JWT
server 验证签名和过期时间
```

### Trusted Device（受信设备）

```
用户在一个新设备上运行 Claude Code
  ↓
系统生成设备 ID
  ↓
首次连接时，需要用户在 claude.ai 上批准
  ↓ "信任此设备？"
  ↓ 批准后，后续连接不需要再批准
```

### Message Fanout（消息扇出）

```
一个消息来自 client A
  ↓ 服务端处理
  ↓ 结果需要发送给：
    - Client A（原始请求者）
    - Client B, C, D（其他连接的客户端）
    - WebUI（对应用户看到实时更新）
```

---

## 关键文件

| 文件 | 用途 |
|------|------|
| `bridge/bridgeApi.ts` | 与 claude.ai 的通信 |
| `bridge/jwtUtils.ts` | JWT 令牌生成和验证 |
| `bridge/trustedDevice.ts` | 设备信任管理 |
| `remote/RemoteSessionManager.ts` | 远程 session 生命周期 |
| `remote/SessionsWebSocket.ts` | WebSocket 连接管理 |

---

## 导航

- **[Bridge Protocol](bridge-protocol.md)** — JWT 握手和消息协议
- **[Multi-Client Architecture](multi-client.md)** — 多客户端的并发和同步

---

**下一步** → [Bridge Protocol](bridge-protocol.md)
