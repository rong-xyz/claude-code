---
title: Permission System
layout: default
nav_order: 6
parent: Agentic Design Overview
has_children: true
---

# Permission 系统：Agent 的沙箱

## 为什么需要权限管控？

想象 Claude 可以执行任意 bash 命令。用户说"帮我看看网站"，Claude 可能：
- 合法地执行 `curl https://example.com`
- 也可能执行 `rm -rf /` ← 灾难

**权限系统的目标**：
1. 阻止意外的危险操作
2. 提示用户"你确定吗？"
3. 在保留便利的同时维护安全

Claude Code 的 permission 系统是一个**五层防线**的多层防御机制。

---

## 核心概念：Permission Mode

系统支持多种权限模式：

| Mode | 特点 | 场景 | 审批速度 |
|------|------|------|---------|
| `default` | 每次都问用户 | 不信任 agent，想完全控制 | 很慢（用户等待） |
| `auto` | ML 分类器自动审批 | 信任 agent，想要流畅体验 | 快（自动） |
| `acceptEdits` | 自动接受文件编辑（危险操作仍问） | 开发工作流 | 中等 |
| `bypass` | 完全自动，不问 | 内部测试、自动化流程 | 极快 |
| `dontAsk` / `yolo` | 拒绝所有非低风险操作 | 沙箱环境，最安全 | 快（直接拒绝） |

启动方式：
```bash
# 每次都问
claude-code --permission-mode default

# 自动审批（推荐）
claude-code --permission-mode auto

# 最安全
claude-code --permission-mode dontAsk
```

---

## Permission 决策 Pipeline：五层防线

当 agent 试图执行一个 tool 时，系统会问："我应该允许吗？"

```
Agent 调用: BashTool { command: "npm test" }
  │
  ├─ [Layer 1] Mode 检查
  │      if mode == "bypass" → ✓ 允许，直接返回
  │      if mode == "dontAsk" → ✗ 拒绝 unsafe tool，返回
  │
  ├─ [Layer 2] Always Allow / Always Deny 规则
  │      检查用户配置的规则列表
  │      如: "Bash(git *)" → 允许 git 命令
  │         "Bash(npm *)" → 允许 npm 命令
  │      我们的命令 "npm test" 匹配 "npm *" → ✓ 允许
  │
  ├─ [Layer 3] 风险分类
  │      规则匹配 → 风险等级为 LOW
  │      如果没匹配到规则，继续下一层
  │
  ├─ [Layer 4] YOLO 分类器
  │      对于 MEDIUM 风险的操作：
  │      - 如果命令看起来"明显安全"（开发模式），自动批准
  │      - 否则需要用户确认
  │      对于 HIGH 风险的操作：
  │      - 一般不自动允许，转到第 5 层
  │
  └─ [Layer 5] 最后手段：提示用户
         用户选择: ✓ 允许 / ✗ 拒绝 / 📝 改规则
```

---

## 关键要点

1. **五层防线**：Mode → Rules → Risk Classification → YOLO → User Prompt
2. **Permission mode**：`default`（每次问）、`auto`（ML 自动）、`bypass`（完全自动）、`dontAsk`（最安全）
3. **规则系统**：Glob 模式，分层来源（CLI > project > user > default）
4. **YOLO 分类器**：ML 自动批准 MEDIUM 风险操作，不需要用户每次确认
5. **Denial tracking**：连续 3 次拒绝自动降级权限，防止烦人的重复提示
6. **路径安全**：多层防御（normalization、URL decode、Unicode clean）
7. **Protected files**：黑名单，永远要再次确认
8. **Sandbox adapter**：高风险操作可以隔离执行

---

## 深入阅读

- [Permission Modes 详解](modes.md)
- [规则系统与 YOLO 分类器](rules.md)
- [路径安全与沙箱](sandbox.md)

核心文件：
- `utils/permissions/permissions.ts`（52 KB）
- `utils/permissions/yoloClassifier.ts`（52 KB）
- `utils/permissions/pathValidation.ts`（62 KB）
