---
title: 高级 Agent 模式
layout: default
nav_order: 3
parent: Advanced Topics
grand_parent: Agentic Design Overview
---

# 高级 Agent 模式

Claude Code 通过 **feature flags** 支持多种高级运行模式，实现不同的工作流优化。

---

## 什么是 Agent Mode？

Agent Mode 是一种特殊的 agent 配置，改变 agent 的行为方式、token 预算、超时限制和功能可用性：

```
普通 session:
  Query Loop
    ↓ single agent
    ↓ 30 分钟超时
    ↓ normal token budget
    ↓ 基础工具集

ULTRAPLAN Mode:
  Query Loop
    ↓ planning agent
    ↓ 30 分钟超时（增强）
    ↓ 2x token budget
    ↓ 深度思考工具集
```

---

## 四大 Agent Mode

### 1. ULTRAPLAN — 深度规划模式（30 分钟）

**用途**：需要深思熟虑、多步骤的复杂任务

**特性**：
- 时限：30 分钟（vs 普通 5 分钟）
- Token 预算：2 倍（vs 普通）
- 专属工具：
  - `PlanningTool` — 生成详细计划
  - `ResearchTool` — 深度研究
  - `ThinkingTool` — 显式推理链
- System Prompt：包含"你有充足的时间和资源"

**激活条件**：
```typescript
// 用户输入包含关键词
"/ultraplan 帮我设计系统架构"

// 或显式要求
const agent = spawnAgent({
  mode: "ultraplan",
  initialMessage: "..."
})
```

**Flow**：
```
用户请求（大型项目）
  ↓
ULTRAPLAN 初始化
  ↓
Agent 说："我需要 30 分钟深度规划"
  ↓
Step 1: 生成多个方案草案（10 分钟）
Step 2: 评估各方案权衡（10 分钟）
Step 3: 选择最优方案，生成详细计划（5 分钟）
Step 4: 准备实施步骤（5 分钟）
  ↓
返回详细计划，用户可选择执行

节省时间：省去反复询问和修改的时间
```

**成本**：由于运行时间和 token 多，成本约为普通 session 的 3-5 倍，但避免了多次迭代。

---

### 2. KAIROS — 永驻助手模式（长期记忆）

**用途**：需要与同一个 agent 的长期交互，agent 记住历史

**特性**：
- Time Awareness：Agent 知道自上次对话以来过了多久
- Memory State：跨 session 保留 MEMORY.md，agent 可以读取并扩展
- Continuity：Agent 会说"上次我们讨论了 X，现在让我们继续..."
- 专属工具：
  - `ReadMemoryTool` — 主动查询旧 memory
  - `UpdateMemoryTool` — 更新关于用户的知识

**激活条件**：
```typescript
// 在 bootstrap 时配置
const KAIROS_MODE = growthBook.getFeatureValue("KAIROS", false)

// 用户启用 KAIROS
/config set mode kairos
```

**System Prompt 补充**：
```
你是用户的永驻 AI 助手。

上次对话：2025-11-20（11 天前）
你们讨论的内容：
  - 用户正在做微服务迁移项目
  - 优先关注数据库分片
  - 有 3 个团队成员参与

建议：
  1. 先回顾上次的进展
  2. 询问有什么新的障碍
  3. 提醒之前的决策上下文
```

**Flow**：
```
用户：我回来了，继续做微服务项目

KAIROS Agent 说：
"欢迎回来！上次（11 天前）我们在规划数据库分片。
我看到你记录了：
  - 已完成：核心数据库 schema 设计
  - 待做：分片键选择
  
让我们从分片键选择继续。有新的进展吗？"
```

**Memory 的角色**：
```
~/.claude/memory/MEMORY.md 内容：

# User Profile

## Project: Microservices Migration
- Status: In Progress (started 2025-11-10)
- Team: 3 members (Alice, Bob, Charlie)
- Current Focus: Database Sharding
- Key Decisions Made:
  - Using MySQL for shards
  - Sharding key: user_id
- Known Issues:
  - Cross-shard transactions problematic
```

---

### 3. BUDDY — 电子宠物模式（交互友好）

**用途**：轻松、日常的对话和任务

**特性**：
- 语气：友好、鼓励性、幽默
- 反馈：更多的 emoji、表情符号、进度条
- 交互：支持自然语言指令（不必"/开头")
- 工具：有趣但强大（如 `/joke`, `/motivate`)

**激活条件**：
```typescript
/config set mode buddy
```

**System Prompt 补充**：
```
你是用户的友好 AI 助手。

行为指南：
  ✨ 使用表情符号和幽默
  🎯 给予清晰的进度反馈
  💪 鼓励用户
  🎭 展示个性

示例：
✅ "太棒了！你完成了一个大任务！🎉"
❌ "Task completed."
```

**Flow**：
```
用户：帮我测试这个代码

BUDDY 说：
🔍 开始检查你的代码...
  ✅ 导入检查
  ⏳ 类型检查...
  ✅ 函数签名正确
  ✅ 边界情况处理完备

结果：所有测试通过！你太棒了 🚀
```

---

### 4. CHICAGO — 计算机使用模式（屏幕控制）

**用途**：需要 agent 看到和操作屏幕的任务

**特性**：
- 屏幕截图支持：Agent 可以`ScreenshotTool` 查看当前屏幕
- 鼠标/键盘：Agent 可以`ClickTool`, `TypeTool` 操作
- OCR：Agent 可以识别屏幕上的文字
- 时限：5 分钟（自动操作限制）

**激活条件**：
```typescript
/config set mode chicago
// 需要授权：允许 agent 访问屏幕
```

**注意**：需要用户明确授权（隐私考虑）。

**Flow**：
```
用户：帮我填这个表单

CHICAGO Agent：
📸 截图当前屏幕
  ↓ 看到表单字段：Name, Email, Phone
  ↓ 识别 "Name" 输入框
  ↓ 点击并输入用户信息
  ↓ 点击 "Submit" 按钮
  ↓ 验证成功消息

✅ 表单提交成功！
```

**安全考量**：
- 不记录屏幕内容（除非明确保存）
- 所有操作可审计
- 用户可随时停止（Ctrl+C）

---

## Feature Flag 控制

所有 Agent Mode 都通过 GrowthBook flags 控制：

```typescript
// bootstrap/state.ts
const growthBook = getGrowthBook()

const ULTRAPLAN_ENABLED = growthBook.getFeatureValue("ULTRAPLAN", false)
const KAIROS_ENABLED = growthBook.getFeatureValue("KAIROS", false)
const BUDDY_ENABLED = growthBook.getFeatureValue("BUDDY", false)
const CHICAGO_ENABLED = growthBook.getFeatureValue("CHICAGO", false)

if (ULTRAPLAN_ENABLED) {
  // 启用 ULTRAPLAN 相关代码
}
```

**GrowthBook 配置示例**：
```json
{
  "features": {
    "ULTRAPLAN": {
      "defaultValue": false,
      "rules": [
        {
          "condition": {
            "version": { "$gte": "0.15.0" }
          },
          "value": true
        }
      ]
    },
    "KAIROS": {
      "defaultValue": false,
      "rules": [
        {
          "condition": {
            "userId": { "$in": ["user123", "user456"] }
          },
          "value": true
        }
      ]
    }
  }
}
```

---

## 模式组合

可以同时启用多个模式：

```
ULTRAPLAN + KAIROS：
  用户："/ultraplan 帮我设计新项目"
  Agent：
    1. 读取 MEMORY.md（KAIROS）
    2. 询问与以前项目的联系
    3. 进行 30 分钟深度规划（ULTRAPLAN）
    4. 更新 MEMORY.md

成本：很高（ULTRAPLAN 的时限 + KAIROS 的记忆操作）
收益：最佳的长期项目设计
```

---

## 选择合适的模式

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 日常编码、小任务 | 默认 | 快速、低成本 |
| 大型系统设计 | ULTRAPLAN | 需要深思 |
| 长期项目持续 | KAIROS | 需要记住历史 |
| 反复调试和测试 | BUDDY + 默认 | 友好的反馈 |
| 自动化重复任务 | CHICAGO | 屏幕操作 |

---

## 深入阅读

### 相关文档

- [Feature Gating](feature-gating.md) — Feature flag 的编译期/运行期隔离
- [Token Budget](../context-memory/compression.md) — ULTRAPLAN 如何管理 token

### 源代码

- `bootstrap/state.ts` — Feature flag 初始化
- `services/modes/` — 各个 mode 的实现
- `coordinator/` — ULTRAPLAN 的多步规划逻辑
- `memdir/` — KAIROS 的记忆管理

---

**下一步** → [Feature Gating](feature-gating.md)
