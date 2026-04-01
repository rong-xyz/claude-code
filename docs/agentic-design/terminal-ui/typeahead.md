---
title: Typeahead 与输入
layout: default
nav_order: 3
parent: Terminal UI
grand_parent: Agentic Design Overview
---

# Typeahead & Input：交互的智能化

`useTypeahead.tsx` 是一个 212 KB 的 React Hook，它提供：
- 斜杠命令补全（如 `/commit`, `/review-pr`）
- 文件路径补全
- 命令历史导航
- 智能排序

---

## Typeahead 的三层

### [1] 命令发现

```
用户输入："/"
  ↓
系统扫描：
  - 内置命令（commit, review-pr, ...）
  - 用户自定义命令（~/.claude/commands/）
  - 延迟发现的技能命令
  ↓
生成建议列表：
  [
    { text: "commit", description: "Git commit workflow" },
    { text: "review-pr", description: "Review GitHub PR" },
    ...
  ]
```

### [2] 模糊匹配和排序

```
用户输入："/rev"
  ↓
候选：[
  "review-pr",       ← 完美匹配前缀
  "revert",          ← 部分匹配
  "everywhere"       ← 包含 "rev"
]
  ↓
排序：
  1. "review-pr"     (前缀匹配)
  2. "revert"        (前缀匹配)
  3. ...
  ↓
显示前 N 条（如 10 条）
```

### [3] 上下文感知

```
不同位置的补全方式：

输入框开头 "/"
  ↓ 补全斜杠命令

输入框中间 "/comm"
  ↓ 补全命令参数（如文件路径）

例如："/edit <file-path>"
  ↓ 补全文件路径，支持相对路径、通配符
```

---

## 文件路径补全

### 基于当前工作目录

```
用户输入："/edit src/co"
  ↓
系统扫描 src/ 目录
  ↓
匹配以 "co" 开头的文件：
  - src/components/
  - src/config.ts
  - src/context.ts
  ↓
显示建议（优先显示目录）
```

### 相对路径解析

```
输入："/edit ../utils/he"
  ↓
解析相对路径："../utils/"
  ↓
扫描 utils/ 目录下匹配 "he" 的文件
  ↓
显示建议：
  - helpers.ts
  - hooks.ts
```

### 特殊路径快捷方式

```
支持快捷方式：
  ~/     → 主目录
  ./     → 当前目录
  ../    → 上级目录
  //     → 项目根目录（git root）
```

---

## 历史导航

### 上/下箭头历史导航

```
用户在输入框内
  ↓
按上箭头：加载前一条历史命令
  ↓ "/commit --amend"（来自 N 条命令前）
  ↓
按上箭头：再上一条
  ↓ "/commit --all"（来自 N+1 条命令前）
  ↓
按下箭头：返回下一条
```

相关 Hook：`useArrowKeyHistory.tsx` (34 KB)

---

## 建议排序算法

### 多维度排序

```
建议列表中，按以下优先级排序：

[1] 完全匹配（用户输入 == 命令名）
      ↓
[2] 前缀匹配（命令名以用户输入开头）
      ↓
[3] 子字符串匹配（用户输入在命令名中间）
      ↓
[4] 模糊匹配（按编辑距离）
      ↓
[5] 频率排序（用户常用的命令靠前）
```

### 频率追踪

```
每次用户执行一个命令：
  记录执行时间戳和频率
  ↓
下次补全时：
  高频命令靠前
  最近用过的靠前
```

---

## 渲染补全窗口

### 补全菜单的显示

```
当有建议时：

Typeahead Menu:
  ├─ > review-pr         (当前选中)
  ├─  revert
  ├─  restore
  └─  (8 more)

  当前选中项用 ">" 标记，彩色高亮
```

### 键盘导航

```
用户在补全菜单中：
  ↑ 上一个建议
  ↓ 下一个建议
  Enter 选择当前建议
  Esc 关闭菜单
  PageUp/PageDown 翻页
```

### 动态更新

```
当用户继续输入时：

"/rev" → 显示 ["review-pr", "revert", ...]
  ↓ 用户继续输入 "i"
"/revi" → 实时更新建议为 ["review-pr", ...]
  ↓ 菜单自动缩小，适应新的匹配数量
```

---

## 延迟加载和性能

### 初始化开销

```
useTypeahead 初始化时：
  扫描所有内置命令（~88 个）
  扫描用户自定义命令（~? 个）
  构建索引（用于快速搜索）
  ↓
如果命令很多（>1000）：
  只加载前 N 条，需要时再加载
```

### 搜索优化

```
使用 Trie 数据结构（前缀树）：
  O(K) 查询时间（K = 输入长度）
  快速前缀匹配
  自动去重
```

---

## 设计决定专栏

### 为什么优先显示目录而不是文件？

**随机排序**：
```
/edit "
  ↓ 结果混合文件和目录
  ↓ 用户需要手动浏览找目录
```

**优先目录**（当前做法）：
```
/edit "
  ↓ 首先显示目录（方便导航）
  ↓ 然后显示目录内的文件
```

优势：大多数情况下，用户是逐层导航（打开目录 → 看目录内文件）。优先显示目录加速这个流程。

### 为什么支持 ~ 和 // 快捷方式？

**总是相对路径**：
```
用户输入："/edit ../../../../../../src/utils"
问题：太冗长，容易出错
```

**支持快捷方式**（当前做法）：
```
/edit ~/projects/app/src/utils
  或
/edit //src/utils  （从项目根目录）

优势：减少输入，清晰易懂
```

---

## 深入阅读

- `hooks/useTypeahead.tsx` — Typeahead 的完整实现（212 KB）
- `hooks/useArrowKeyHistory.tsx` — 历史导航的实现（34 KB）
- `hooks/useGlobalKeybindings.tsx` — 全局键盘快捷键（31 KB）
- `constants/` — 命令列表和描述

---

**下一步** → [Session State](../session-state/)
