---
title: Claude Code Documentation
layout: home
nav_order: 1
---

# Claude Code 文档

欢迎来到 Claude Code 的官方文档。Claude Code 是 Anthropic 的官方 AI 编程 CLI 工具，通过交互式 agent 接口实现强大的软件工程任务。

## 学习资源

本文档提供深入的指南，帮助理解 Claude Code 的架构和 agentic 系统设计原理。

### [Agentic Design 学习路径]({{ site.baseurl }}/agentic-design/)

一份全面的学习资料，涵盖 Claude Code 最核心的设计决策：

- **[00: 代码库全景]({{ site.baseurl }}/agentic-design/00-codebase-tour)** — 代码库结构导游，从顶层到最深处
- **[01: Agent Loop]({{ site.baseurl }}/agentic-design/01-agent-loop)** — 核心 Query Loop，`while(tool_use)` 的艺术
- **[02: Tool 系统]({{ site.baseurl }}/agentic-design/02-tool-system)** — 40+ 工具的统一接口设计
- **[03: 多 Agent 协调]({{ site.baseurl }}/agentic-design/03-multi-agent-coordination)** — 分解复杂任务，四种 agent 类型
- **[04: Permission 系统]({{ site.baseurl }}/agentic-design/04-permission-system)** — 五层防线的安全沙箱
- **[05: Context & Memory]({{ site.baseurl }}/agentic-design/05-context-and-memory)** — 上下文窗口和记忆管理
- **[06: Feature Gating]({{ site.baseurl }}/agentic-design/06-feature-gating)** — 内部功能的编译期/运行期隔离

## 快速开始

从 [Agentic Design 学习路径]({{ site.baseurl }}/agentic-design/) 开始，获得完整的学习概览和推荐阅读顺序。

---

**最新更新**（2026 年 4 月）：基于 Claude Code 源码泄露分析，所有文档已大幅扩充。详见 [GitHub 仓库](https://github.com/rong-xyz/claude-code)。
