---
title: Permission Modes 与决策 Pipeline
layout: default
nav_order: 1
parent: Permission System
grand_parent: Agentic Design Overview
---

# Permission Modes 与决策 Pipeline

## Permission Mode 详解

### default（默认）
- 每个操作都提示用户
- 最安全但最慢
- 适合不信任 agent 的场景

### auto（推荐）
- ML 分类器自动审批低风险和明显安全的中风险操作
- 高风险操作仍会提示用户
- 平衡安全和便利

### acceptEdits（开发模式）
- 自动接受文件编辑（`FileEditTool`、`FileWriteTool`）
- 其他操作仍然检查
- 适合代码编辑工作流

### bypass（内部模式）
- 完全自动，不提示用户
- 所有操作都通过
- 仅用于 CI/CD 和内部测试

### dontAsk / yolo（最安全）
- 拒绝所有 unsafe 工具（高风险操作）
- 只允许读操作和已规则允许的操作
- 适合沙箱环境

---

## Permission 决策 Pipeline：五层防线

```
[Layer 1] Mode 检查
  ↓
  若 mode == "bypass" → ✓ 允许
  若 mode == "dontAsk" → 检查是否 safe tool
    ├─ 是（FileRead、Glob、Grep 等）→ ✓ 允许
    └─ 否 → ✗ 拒绝
  否则继续下层
  ↓
[Layer 2] Always Allow / Always Deny 规则
  ↓
  检查用户配置的规则列表
  ├─ 匹配 Always Allow 规则 → ✓ 允许
  ├─ 匹配 Always Deny 规则 → ✗ 拒绝
  └─ 无匹配 → 继续下层
  ↓
[Layer 3] 风险分类
  ↓
  根据 tool 类型和操作内容分类为 LOW / MEDIUM / HIGH
  ├─ LOW → ✓ 自动允许
  ├─ MEDIUM → 继续下层
  └─ HIGH → 继续下层
  ↓
[Layer 4] YOLO 分类器
  ↓
  对于 MEDIUM 风险：
  ├─ 如果命令看起来"明显安全" → ✓ 自动批准
  └─ 否则 → 继续下层
  ↓
[Layer 5] 最后手段：用户提示
  ↓
  用户做最终决定: ✓ 允许 / ✗ 拒绝 / 📝 修改规则
```

---

## Risk Level（风险等级）

### LOW
- 自动允许（不问用户）
- 例：FileReadTool, GlobTool, GrepTool, WebFetchTool
- 典型：ls, cat, grep

### MEDIUM
- 取决于 mode（auto mode 用分类器，default mode 问用户）
- 例：FileEditTool, BashTool (git 命令)
- 典型：npm install, git commit

### HIGH
- 一般不自动允许（需要用户确认）
- 例：BashTool (rm -rf), PowerShellTool, REPLTool
- 典型：rm -rf /, curl | bash, eval()

---

## Denial Tracking：Circuit Breaker 机制

如果用户连续拒绝 agent 多次，系统会自动**降级权限模式**：

```
用户拒绝:
  1. "BashTool"       ← Denial #1
  2. "FileEditTool"   ← Denial #2
  3. "BashTool"       ← Denial #3

连续 3 次拒绝触发!
  ↓
系统自动切换到 "default" mode
后续所有操作都要问用户
  ↓
这样防止：
  - Agent 持续尝试被拒的操作
  - 用户一直看到提示烦不胜烦
```

---

## 深入阅读

- `utils/permissions/PermissionMode.ts`：Mode 定义
- `utils/permissions/permissions.ts`：完整决策逻辑（52 KB）
- `utils/permissions/yoloClassifier.ts`：YOLO 分类器实现（52 KB）
