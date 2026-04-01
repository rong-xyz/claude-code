---
title: 规则系统与 YOLO 分类器
layout: default
nav_order: 2
parent: Permission System
grand_parent: Agentic Design Overview
---

# 规则系统与 YOLO 分类器

## 规则系统：Glob 模式

用户可以配置规则，告诉 agent 什么是自动允许的。规则语法使用 **Glob 模式**（不是正则表达式）：

```
Bash(git *)               # 允许任何 git 命令
Bash(npm install)         # 只允许 npm install，不允许 npm test
FileEdit(src/*.ts)        # 允许编辑 src/ 下的 .ts 文件
FileRead(/*)              # 允许读任何文件
BashDeny(rm -rf /*)       # 明确拒绝危险命令
```

## 规则分层（优先级）

规则可以来自多个地方，优先级从**高到低**：

1. **CLI 参数**：`--approve-file "FileEdit(src/*.ts)"`
2. **项目配置**：`.claude/settings.json` 中的 `approvals`
3. **用户设置**：`~/.claude/settings.json`
4. **系统默认**：Claude Code 内置的基础规则

下层规则会被上层规则覆盖。

## 为什么用 Glob 而不是正则表达式？

**Glob**：`src/*.ts` → 简单直观
**正则**：`^src/[^/]+\.ts$` → 复杂，容易写错

目标用户是**工程师，但不是安全专家**。Glob 的错误率更低，而且 CLI 工具（如 .gitignore）已经用 Glob，学习曲线平缓。

---

## YOLO 分类器：ML 自动审批

对于**规则未覆盖的操作**，系统用 YOLO 分类器（"You Only Live Once"）决定：

```
输入:
  tool_name: "BashTool"
  command: "npm test"
  context: 当前任务是"修复 bug"

分类器评估:
  - 命令类型（npm 相对安全）
  - 模式匹配（test 是常见操作，不是破坏性）
  - 上下文（"修复 bug" 任务中运行 npm test 很合理）

结论: "这个操作很像是开发者会做的正常事，批准"
```

分类器基于启发式规则和一些 ML 训练，考虑：
- **Tool 类型**（BashTool 比 FileReadTool 高风险）
- **具体命令**（`npm test` 比 `rm -rf` 安全）
- **当前上下文**（"修复 bug" vs "浏览网页"）

## YOLO 的风险等级

```
LOW
  → 自动允许（不问用户）
  → 例：FileReadTool, GlobTool, GrepTool, WebFetchTool
  → 典型：ls, cat, grep

MEDIUM
  → 取决于 mode（auto mode 用分类器，default mode 问用户）
  → 例：FileEditTool, BashTool (git 命令)
  → 典型：npm install, git commit

HIGH
  → 一般不自动允许（需要用户确认）
  → 例：BashTool (rm -rf), PowerShellTool, REPLTool
  → 典型：rm -rf /, curl | bash, eval()
```

---

## 为什么需要 ML 分类器？

如果只用规则：
```
允许所有 Bash 命令?
  → 太宽，不安全（用户可能 rm -rf）

只允许 git/npm 命令?
  → 太窄，不便利（sudo docker pull 被拒）
```

ML 分类器的优势：
```
可以说"这个命令看起来是在开发工作流的一部分，
而不是破坏性操作，所以允许"

如果出错（分类有假阴性）：
  系统还有：
  - Denial tracking（连续拒绝降级）
  - Protected files（黑名单）
  - Circuit breaker（防反复）
```

所以 ML 不是唯一防线，而是**多层防御中的一层**。

---

## 深入阅读

- `utils/permissions/permissionRuleParser.ts`：规则解析
- `utils/permissions/permissionsLoader.ts`：规则加载
- `utils/permissions/yoloClassifier.ts`：YOLO 分类器实现（52 KB）
