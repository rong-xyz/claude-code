---
title: 路径安全与沙箱隔离
layout: default
nav_order: 3
parent: Permission System
grand_parent: Agentic Design Overview
---

# 路径安全与沙箱隔离

## 路径安全：防止路径遍历攻击

即使规则写得再好，攻击者也可能用**特殊编码**绕过：

```
规则: FileEdit(src/*.ts)  ← 只允许 src/ 目录

攻击尝试:
  path: "src/../../../etc/passwd"              ← 目录遍历
  path: "src%2f..%2f..%2fetc%2fpasswd"        ← URL 编码
  path: "src/\u202e../../../etc/passwd"       ← Unicode 隐藏字符
```

系统有多层防御：

### 1. Path Normalization（路径规范化）

```
"src/../../../etc/passwd"
  ↓ 规范化
"/etc/passwd"
  ↓ 检查是否在 src/ 内?
否 → 拒绝
```

### 2. URL Decoding（URL 解码）

```
"src%2f..%2f..%2fetc"
  ↓ 解码
"src/../../etc"
  ↓ 规范化
"/etc"
  ↓ 拒绝
```

### 3. Unicode Normalization（Unicode 规范化）

```
隐藏字符被检测和清理
```

相关文件：`utils/permissions/pathValidation.ts`（62 KB）

---

## Protected Files：敏感文件黑名单

某些文件**永远不会被自动编辑**，即使通过了所有权限检查：

```
.gitconfig              ← Git 配置，修改可能破坏工作流
.bashrc, .zshrc         ← Shell 配置，修改可能卡住终端
.claude/settings.json   ← Claude Code 自己的配置
.mcp.json               ← MCP 配置，影响工具可用性
.env, .env.local        ← 环境变量，可能含密钥
```

这些文件即使通过了权限检查，也会再次提示用户确认。

---

## Sandbox Adapter：隔离执行环境

对于操作**风险较高但必要**的情况，系统可以：
- 将执行重定向到**受限环境**（Docker 容器、沙箱）
- 防止主机系统被破坏
- 同时允许 agent 功能继续

```
用户说："我需要测试一个未知的脚本"
  ↓
Agent 请求执行 script.sh
  ↓
权限检查：这个脚本未验证，HIGH 风险
  ↓
系统说："我可以在沙箱中运行这个"
  ↓
在容器内执行 script.sh
  ↓
如果脚本做坏事（删文件、改配置），只影响沙箱
  ↓
Agent 看到执行结果，继续下一步
```

---

## Bypass Mode：内部和特殊环境

一个 `isBypassPermissionsModeAvailable` flag 允许特殊环境（CI/CD、内部测试）在没有交互提示的情况下运行，同时维护审计日志：

```
环境: CI/CD pipeline
  ↓
Agent 执行大量 tool（构建、测试、部署）
  ↓
所有操作自动进行（没有用户提示）
  ↓
但所有操作都被记录到 audit log
  ↓
运维人员可以后审（如果出问题，追踪是哪步）
```

---

## 常见误解

**误解 1**："Protected files 名单是硬编码的？"

实际：可以通过配置扩展：
```json
{
  "protectedFiles": [".gitconfig", ".bashrc", ".my-precious-file"]
}
```

**误解 2**："权限检查会很慢？"

实际：
- Layer 1-3（mode、规则、分类）都是 O(1) 或 O(n) 快速操作
- YOLO 分类器是专门优化的轻量级模型
- 权限检查通常 <10ms

**误解 3**："所有用户都能看到内部代码？"

实际：不能。公开二进制中编译时被删除的代码，用户看不到。

---

## 深入阅读

- `utils/permissions/pathValidation.ts`：路径安全（62 KB）
- `utils/permissions/denialTracking.ts`：Denial tracking 逻辑
- `utils/permissions/permissionExplainer.ts`：可解释的决策
