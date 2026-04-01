---
title: Bridge Protocol
layout: default
nav_order: 1
parent: Bridge & Remote
grand_parent: Agentic Design Overview
---

# Bridge Protocol：JWT 握手和消息交换

---

## 初始连接（握手）

### WebSocket 握手

```
客户端（WebUI）向本地 CLI 发起 WebSocket 连接：

ws://localhost:3000/bridge?token=<JWT_TOKEN>&deviceId=abc-123

服务端验证：
  ① 检查 JWT 签名（用 secret 验证）
  ② 检查 exp（过期时间）
  ③ 检查 deviceId 是否被信任
```

### JWT Payload 结构

```json
{
  "sub": "user-id",              // Subject（用户 ID）
  "aud": "claude-code-cli",       // Audience（目标应用）
  "iat": 1700000000,              // Issued at（发行时间）
  "exp": 1700003600,              // Expiration（过期时间，1 小时后）
  "deviceId": "abc-123-def-456",  // 设备 ID
  "scope": ["read", "write", "execute"]  // 权限范围
}
```

### 设备信任流程

```
新设备首次连接：

1. 客户端提交 JWT + deviceId
2. 服务端检查 deviceId 是否已信任
   ↓ 已信任 → 允许连接
   ↓ 未信任 → 返回 DEVICE_NOT_TRUSTED 错误
3. 客户端捕获错误
   ↓ 提示用户在 claude.ai 上批准
4. 用户在 claude.ai 点击 "信任此设备"
5. claude.ai 向 CLI 服务器发送 approve 请求
   ↓ 将 deviceId 加入白名单
6. 客户端重新连接
   ↓ 这次成功
```

---

## 消息协议

### 客户端 → 服务端

```json
{
  "type": "query",
  "id": "req-123",
  "sessionId": "abc-def-456",
  "messages": [
    { "role": "user", "content": "帮我写一个 function" }
  ],
  "options": {
    "stream": true,
    "temperature": 0.7
  }
}
```

### 服务端 → 客户端

```json
{
  "type": "message",
  "id": "req-123",
  "delta": "function ",
  "index": 0
}

{
  "type": "message",
  "id": "req-123",
  "delta": "hello() {",
  "index": 1
}

...（流式传输）

{
  "type": "done",
  "id": "req-123",
  "totalTokens": 245
}
```

### 错误消息

```json
{
  "type": "error",
  "id": "req-123",
  "code": "DEVICE_NOT_TRUSTED",
  "message": "Device abc-123 is not trusted. Please approve in claude.ai"
}
```

---

## WebSocket 生命周期

### 连接和认证

```
[客户端]                    [服务端]
    |
    | WebSocket 握手
    |──────────────────────→|
    |                       | 验证 JWT
    |                       | 检查 deviceId
    |                       |
    |←──────── 200 OK ──────|
    | (连接建立)
```

### 消息交换

```
[客户端]                    [服务端]
    |
    | { type: "query", ... }
    |──────────────────────→|
    |                       | 处理请求
    |                       | 调用 Claude API
    |                       |
    |←── { type: "message" } |
    | (流式响应)
    |←── { type: "message" } |
    |←── { type: "done" }   |
```

### 心跳（Heartbeat）

```
每 30 秒：

[服务端]                    [客户端]
    |
    | { type: "ping" }
    |──────────────────────→|
    |                       | 收到 ping
    |                       |
    |←─── { type: "pong" } ──|
    | (连接活跃)
```

### 断开连接

```
客户端关闭连接
  ↓
服务端清理该客户端的状态
  ↓
其他客户端继续运行（共享 session）
```

---

## 数据安全

### TLS/HTTPS

```
在 bridge mode 下：
  WebUI ←──TLS──→ 本地 CLI
  
认证方式：
  ① JWT 验证身份
  ② TLS 加密传输
  ③ 设备 ID 白名单
```

### 令牌有效期

```
JWT 有效期：1 小时
  ↓ 快过期时，WebUI 自动刷新（获取新 token）
  ↓ 无缝续期，用户无感
  
如果 token 过期且无法刷新：
  → 提示用户重新认证
  → 点击 "Sign in again"
```

### 权限范围（Scope）

```
token 中的 scope 字段限制客户端能做什么：

scope: ["read", "write"]
  → 可以读消息、写消息
  → 不能执行 shell 命令（需要 "execute" 权限）

scope: ["read"]
  → 只读，不能发送请求
```

---

## 设计决定专栏

### 为什么用 JWT 而不是 session cookie？

**Session Cookie**：
```
问题：
- 跨域困难（WebUI 和 local CLI 不同域）
- 难以在多个客户端中使用
- 服务端需要维护 session store
```

**JWT**（当前做法）：
```
优势：
- 无状态：令牌本身包含所有信息
- 跨域友好：可在 Authorization header 中传输
- 多客户端：同一个 token 可在多个设备使用
- 支持过期：token 有 exp，自动失效
```

---

## 深入阅读

- `bridge/bridgeApi.ts` — Bridge API 实现
- `bridge/jwtUtils.ts` — JWT 生成和验证
- `bridge/trustedDevice.ts` — 设备信任管理
- `remote/SessionsWebSocket.ts` — WebSocket 服务器
