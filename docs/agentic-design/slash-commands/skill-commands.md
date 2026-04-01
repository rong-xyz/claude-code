---
title: Skill Commands
layout: default
nav_order: 2
parent: Slash Commands
grand_parent: Agentic Design Overview
---

# Skill Commands：用户定义的技能

除了 88 个内置命令，Claude Code 还支持用户创建自定义技能（Skill Commands）。

---

## 什么是 Skill？

Skill 是用户在 Markdown 中定义的自定义 slash 命令。用户可以创建领域特定的工作流。

### 例子：用户自定义的 /deploy 技能

```markdown
---
name: deploy
description: Deploy the app to production with safety checks
usage: /deploy [--env=prod|staging] [--dry-run]
tools:
  - BashTool
  - WebFetchTool
---

# Deploy Workflow

You are a deployment expert. Your task:

1. Check deployment readiness:
   - Run tests
   - Verify all commits pushed
   - Check CI status

2. Deploy:
   - Tag the release
   - Push to origin
   - Monitor logs

3. Verify:
   - Health check
   - Smoke tests
   - Alert on issues

Safety: Always ask for confirmation before production deploy.
```

### 位置

```
~/.claude/commands/
├── deploy.md
├── generate-docs.md
├── refactor-utils.md
└── ... (用户的其他技能)
```

---

## 技能的发现和加载

### 启动时扫描

```
claude-code 启动
  ↓
扫描 ~/.claude/commands/
  ↓
发现 deploy.md, generate-docs.md, ...
  ↓
注册为可用命令
  ↓
用户可以运行 /deploy 等
```

### Typeahead 集成

```
用户输入："/"
  ↓
系统显示：
  /commit (built-in)
  /review-pr (built-in)
  /deploy (user skill) ← 用不同颜色标记
  /generate-docs (user skill)
  ...
```

---

## 创建自定义技能

### 步骤 1：创建 Markdown 文件

```bash
cat > ~/.claude/commands/my-skill.md <<'EOF'
---
name: my-skill
description: My custom workflow
tools:
  - BashTool
  - FileEditTool
---

# My Custom Skill

You are an expert in X. When the user runs /my-skill:

1. Step 1: ...
2. Step 2: ...
3. Step 3: ...

Always be careful about Y and Z.
EOF
```

### 步骤 2：测试

```bash
claude-code /my-skill
```

### 步骤 3：共享

```bash
# 导出为 .zip
claude-code /skill export my-skill

# 发给别人
# 别人导入：
claude-code /skill import my-skill.zip
```

---

## 技能的 Frontmatter 格式

```yaml
---
name: deploy                           # 命令名
description: Deploy with safety        # 描述
usage: /deploy [--env=prod|staging]   # 使用方式
tools:                                # 需要的工具
  - BashTool
  - WebFetchTool
parameters:                           # 可选参数定义
  env:
    type: string
    choices: [prod, staging]
    default: staging
  dryRun:
    type: boolean
    default: false
version: 1.0                          # 技能版本
author: john                          # 作者（可选）
tags: [deployment, safety]            # 标签
---
```

---

## SkillTool：技能的执行者

### SkillTool 的职责

```
当用户运行 /deploy 时：

1. SkillTool 接收命令
   → 命令名：deploy
   → 参数：{ env: "prod" }

2. 读取 ~/.claude/commands/deploy.md
   → 解析 frontmatter（元数据）
   → 提取 systemPrompt（Markdown 正文）

3. 创建临时 Agent
   → 系统提示 = deploy.md 的内容
   → 装备的工具 = tools 字段列出的

4. 运行 Agent
   → 用户消息："/deploy --env=prod"
   → Agent 自主执行工作流

5. 返回结果
   → 给用户输出
```

---

## 技能的版本管理

### 技能版本

```
version 字段允许技能更新：

当前版本：deploy.md (v1.0)
  
用户的 ~/.claude/commands/deploy.md (v1.0)
  
社区贡献者更新为 v1.1
  ↓
用户可以运行：/skill update deploy
  ↓ 更新到 v1.1（或提示用户选择）
```

### 兼容性

```
v1.0 的技能在 v2.0 的 Claude Code 上可能不兼容
  ↓
Claude Code 检查 version 字段
  ↓
如果 version < minVersion：警告用户
  ↓
用户选择：升级技能 / 降级 Claude Code / 继续使用
```

---

## 技能的权限

### 权限声明

```
技能的 frontmatter 中：

tools:
  - BashTool      # 需要执行任意命令
  - FileEditTool  # 需要编辑文件
  - WebFetchTool  # 需要网络访问
```

### 权限检查

```
用户运行 /deploy
  ↓
系统检查：deploy.md 需要什么权限？
  → BashTool, FileEditTool, WebFetchTool

用户的权限模式是 "auto"
  → 这些权限都自动允许 → 继续执行

用户的权限模式是 "acceptEdits"
  → 提示用户确认 → 用户同意后执行
```

---

## 技能库和社区

### 本地技能库

```
~/.claude/commands/
  用户本地的技能集合
  完全私密
  完全可控
```

### 社区技能库（假设存在）

```
# 发布技能
claude-code /skill publish deploy.md

# 发现技能
claude-code /skill search deploy

# 安装技能
claude-code /skill install community/deploy
```

---

## 设计决定专栏

### 为什么用 Markdown 而不是 YAML 或 JSON？

**YAML**：
```yaml
name: deploy
description: ...
systemPrompt: |
  You are a deployment expert...
  Step 1: Check...
  Step 2: Deploy...
```

**Markdown**（当前做法）：
```markdown
---
name: deploy
description: ...
---

# Deploy Workflow

You are a deployment expert. Your task:

1. Check...
2. Deploy...
```

优势：
- 更易读（可以有标题、列表、代码块）
- 更易写（用户熟悉 Markdown）
- 支持格式化（如代码示例、强调）

### 为什么技能的权限要显式声明？

**隐式**：
```
用户运行 /deploy
系统尝试执行，失败时才发现权限问题
```

**显式声明**（当前做法）：
```
deploy.md 声明需要 BashTool
启动时检查权限
用户知道会发生什么
```

优势：
- 安全（不会惊喜地要求权限）
- 透明（用户能看到技能要做什么）
- 可拒绝（用户知道风险，可以拒绝运行）

---

## 深入阅读

### 相关文档

- [Tool Registry](../tool-system/tool-registry.md) — SkillTool 的注册
- [Agent Types](../multi-agent/agent-types.md) — skill 触发的是哪种 agent 类型

### 源代码

- `tools/SkillTool/` — SkillTool 的实现
- `commands/skill.ts` — /skill 命令本身
- `~/.claude/commands/` — 用户技能存储（用户本地）

---

**下一步** → [Feature Gating](../advanced/feature-gating.md)
