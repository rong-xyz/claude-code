---
title: Slash Commands
layout: default
nav_order: 13
parent: Agentic Design Overview
has_children: true
---

# Slash Commands：88 个命令系统

Claude Code 有 88 个内置命令（slash commands）。这些命令通过"/" 前缀调用，用来执行特定任务。

```
/commit              # Git commit workflow
/review-pr <number>  # Review a GitHub PR
/memory-dream        # Trigger autoDream
/session list        # List sessions
...（85 个其他）
```

---

## 为什么有这么多命令？

### 问题：通用 Agent 的限制

```
用户："帮我提交一个 commit"
  ↓
纯 Agent 回答："好的，我来帮你..."
  → 可能走很多弯路（设计流程、找工具、处理细节）
  
带有 /commit 命令的 Agent：
  用户："/ commit"
  ↓ Agent 知道确切流程
  ↓ 直接执行（设计、实现、校验）一气呵成
```

### Slash 命令的优势

1. **领域专家**：每个命令都是特定领域的最佳实践
2. **快速**：无需 agent 从头思考
3. **一致**：同一个命令，每次执行方式相同
4. **可发现**：用户可以列出所有命令

---

## 命令的分类

### 版本控制类（Git）

```
/commit                  # 提交工作流
/commit-push-pr          # 一体化 commit + push + PR
/review-pr               # 审查 PR
/autofix-pr              # 自动修复 PR
/sync-branch             # 同步分支
```

### 内存和学习

```
/memory-dream            # 触发 autoDream 巩固
/memory-add <topic>      # 手动添加 memory
/memory-search <query>   # 搜索 memory
```

### 任务和会话

```
/task create <name>      # 创建 task
/task list               # 列出 tasks
/session list            # 列出 sessions
/session resume <id>     # 恢复旧 session
```

### 配置和系统

```
/config get <key>        # 读取配置
/config set <key> <val>  # 设置配置
/debug                   # 进入调试模式
/help                    # 显示帮助
```

### Agent 管理

```
/agent spawn <type>      # 生成新 agent
/agent list              # 列出活跃 agent
/agent kill <id>         # 杀死 agent
```

### 其他（70+ 个）

```
/web-search <query>      # 网络搜索
/extract-url             # 提取消息中的 URL
/refactor                # 代码重构建议
...
```

---

## 命令的结构

### YAML/Markdown 定义

```
每个命令可以定义为：

/commit:
  name: "Git Commit"
  description: "Smart git commit workflow with AI suggestions"
  usage: "/commit [--amend] [--no-push]"
  examples:
    - "/commit"
    - "/commit --amend"
  systemPrompt: |
    You are a Git commit expert...
  tools:
    - BashTool (for git commands)
    - FileReadTool (for checking staged files)
  parameters:
    amend:
      type: boolean
      description: "Amend to previous commit"
    noPush:
      type: boolean
      description: "Don't push after committing"
```

---

## 命令发现和路由

### Central Router（commands.ts）

```
commands.ts (~25K 行)：

switch (userInput) {
  case /^\/commit\b/:
    return await runCommitCommand(args)
  case /^\/review-pr\b/:
    return await runReviewPRCommand(args)
  case /^\/session\b/:
    return await runSessionCommand(args)
  ...
  default:
    return unknownCommand(userInput)
}
```

### 命令的执行步骤

```
1. 解析命令（/commit --amend）
   → 命令名：commit
   → 参数：{ amend: true }

2. 权限检查
   → 需要 BashTool 权限？
   → 需要 FileEditTool 权限？

3. 加载命令定义
   → 获取 systemPrompt
   → 获取 tools list

4. 创建临时 Agent
   → 用命令的 systemPrompt
   → 装备指定 tools
   → 给 agent 原始用户消息（如 " amend previous commit"）

5. 执行 Agent
   → agent 自主运行，直到完成

6. 返回结果
   → 输出返回给用户
```

---

## 导航

- **[Command Routing](command-routing.md)** — 如何路由和执行命令
- **[Skill Commands](skill-commands.md)** — 用户定义的技能
