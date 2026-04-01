---
title: Session Persistence
layout: default
nav_order: 2
parent: Session State
grand_parent: Agentic Design Overview
---

# Session Persistence：保存和恢复

Claude Code 的 session 是文件系统上的一组文件。用户可以保存、加载、共享、和版本控制这些文件。

---

## Session 存储位置

```
~/.claude/sessions/
├── active/
│   └── 2026-04-01-abc-123/
│       ├── metadata.json       # Session 元数据
│       ├── messages.jsonl      # 消息历史（NDJSON 格式）
│       ├── state.json          # 当前 AppState 快照
│       └── checkpoints/        # 中间检查点
│
├── archived/
│   └── 2026-03-15-xyz-789/
│       └── (完成的 session 的存档)
│
└── remote/
    └── server-session-id/      # 远程 session 的缓存
```

---

## Session 文件的内容

### metadata.json

```json
{
  "sessionId": "2026-04-01-abc-123",
  "createdAt": "2026-04-01T13:45:00Z",
  "resumedAt": "2026-04-01T14:30:00Z",
  "userId": "user@example.com",
  "projectDirectory": "/home/user/claude-code",
  "tags": ["feature/agent-loop", "debugging"],
  "status": "active",
  "autoSave": true,
  "totalMessages": 42,
  "totalTokensUsed": 125000
}
```

### messages.jsonl

```jsonl
{"id":"msg-1","role":"user","content":"设计一个agent系统","timestamp":"2026-04-01T13:45:00Z"}
{"id":"msg-2","role":"assistant","content":"以下是我的设计...","timestamp":"2026-04-01T13:45:05Z"}
{"id":"msg-3","role":"user","content":"继续实现第一部分","timestamp":"2026-04-01T13:45:10Z"}
```

关键点：
- NDJSON 格式（每行一个 JSON 对象）
- 不可变：新消息追加，不修改旧消息
- 可流式读取：支持 `tail -f sessions/*/messages.jsonl`

### state.json

```json
{
  "agents": {
    "agent-A": {
      "status": "completed",
      "progress": 100,
      "completedTasks": 5
    }
  },
  "messages": [...],
  "input": {
    "value": "",
    "suggestions": []
  },
  "system": {
    "tokenUsage": 125000,
    "contextPressure": 0.75,
    "compactionCount": 2
  }
}
```

---

## Session ID 的格式

```
格式：<date>-<random-id>
示例：2026-04-01-abc-123-def-456

分解：
  - 日期：2026-04-01（ISO 8601）
  - 随机 ID：abc-123-def-456（36 字母表，8 个字符，低碰撞率）
  
优势：
  - 易于人类辨认（日期）
  - 全局唯一（概率上）
  - 可排序（时间序列）
```

---

## Session 恢复流程

### 自动恢复

```
用户运行 claude-code（无参数）
  ↓
系统检查 ~/.claude/sessions/active/
  ↓
如果有最近的活跃 session：
  → 加载 messages.jsonl（最后 N 条）
  → 加载 metadata.json（session 信息）
  → 加载相关 MEMORY.md
  ↓ 显示："恢复 session abc-123，上次于 14:30 停止"
  ↓
准备好继续工作
```

### 手动选择

```
用户运行 claude-code /session list
  ↓
显示所有 sessions（标记哪个最新）
  
用户运行 claude-code /session resume <id>
  ↓
加载指定 session
```

### 导入外部 Session

```
用户有一个旧的 session 文件：session-old.zip
  ↓
claude-code /session import session-old.zip
  ↓
展开并恢复为新的活跃 session
```

---

## 消息历史的加载策略

### 选择性加载

```
完整的 messages.jsonl 可能很大（MB 级别）
加载全部会很慢，也不必要

策略：
  ① 加载最后 N 条消息（如 50 条）进入 AppState
  ② 更早的消息保留在磁盘上
  ③ 如果用户要查看更早的，再加载
```

### 上下文优化

```
Session 恢复时：

最后 50 条消息 → 加载进 context
  ↓
但这 50 条可能太长（超过 context window 的 10%）
  ↓
自动进行 compaction（总结）
  ↓
[总结] + [最近 20 条] → context
```

---

## Session 的生命周期管理

### Auto-save（自动保存）

```
在活跃 session 中：
  每个 API 响应后，自动保存
  
  步骤：
  ① 新消息追加到 messages.jsonl
  ② 更新 state.json 快照
  ③ 更新 metadata.json（tokenUsage 等）
  
间隔：
  实时保存（不会丢失工作）
```

### Session 归档

```
用户完成一个 session：
  claude-code /session close
  或
  手动运行 claude-code /session archive <id>
  
影响：
  移动从 active/ → archived/
  标记为已完成
  不再自动加载
```

### 清理旧 Session

```
系统自动清理：
  age > 90 天 的 archived sessions
  或
  用户手动运行：claude-code /session cleanup
  
保留策略：
  一些重要 session 可标记为 "pinned"，不会被清理
```

---

## 设计决定专栏

### 为什么用文件存储而不是数据库？

**数据库**（PostgreSQL、SQLite）：
```
优势：查询灵活，支持复杂查询
劣势：需要额外依赖，用户部署复杂，无法版本控制
```

**文件存储**（JSON + JSONL）（当前做法）：
```
优势：
- 简单：无需数据库服务
- 版本控制友好：用户可以 git commit session
- 可转移：session 就是文件夹，可复制、备份、分享
- 人类可读：可用文本编辑器查看/编辑

劣势：
- 查询稍慢（但用户通常不查询，只用最近的）
- 并发写入需要小心（用文件锁解决）
```

成本：换取简单性和用户体验，值得。

### 为什么选择 NDJSON 而不是单个 JSON 数组？

**单个数组 (messages.json)**：
```json
[
  { "id": "msg-1", ... },
  { "id": "msg-2", ... },
  ...（上万条）
]
```
问题：加载很大的文件时，需要一次解析全部（内存爆炸）

**NDJSON (messages.jsonl)**：
```jsonl
{"id":"msg-1",...}
{"id":"msg-2",...}
```
优势：
- 流式读取：一次读一行，内存恒定
- 追加很快：append 而不是重写整个文件
- 兼容 Unix 工具：`tail -f`, `grep`, `jq` 等

---

## 深入阅读

- `remote/RemoteSessionManager.ts` — 多客户端 session 管理
- `tasks/LocalMainSessionTask.ts` — 本地 session 任务跟踪
- `commands/session.ts` — session 命令实现（/session list 等）
- `memdir/memdir.ts` — 与 MEMORY.md 的集成

---

**下一步** → [Bridge & Remote](../bridge-remote/)
