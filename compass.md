# Claude Code 源码泄露：512,000 行代码揭示了什么

**2026 年 3 月 31 日，Anthropic 的 Claude Code 再次因 npm 包中遗留的 source map 文件导致全部源码曝光，~1,900 个 TypeScript 文件、512,000 行代码被完整还原。** 这次泄露揭示了一个令人意外的事实：当今最强大的 AI 编程工具核心架构极其简洁——一个 `while(tool_use)` 循环驱动一切。但在这层简洁之下，是一套精密的工程体系：三层上下文压缩、prompt cache 优化、多智能体协调，以及多个尚未发布的惊人功能——包括永驻后台的 KAIROS 助手、30 分钟深度规划的 ULTRAPLAN、甚至一个电子宠物系统 BUDDY。这是 Claude Code **第二次**因同样原因泄露源码（首次在 2025 年 2 月），暴露了 Anthropic 构建流程中的持续性疏漏。

---

## 泄露全貌：59.8 MB 的 source map 如何掀翻一切

安全研究员 Chaofan Shou（@Fried_rice）在 2026 年 3 月 31 日凌晨发现，npm 包 `@anthropic-ai/claude-code` v2.1.88 中附带了一个 **59.8 MB 的 `cli.js.map` 文件**。这个 source map 文件的 `sourcesContent` 数组中完整保存了每一个原始 TypeScript 源文件，同时引用了 Anthropic 自己 R2 云存储上的一个 ZIP 归档。原因很简单：Claude Code 使用 **Bun 的 bundler** 构建，而 Bun 默认生成 source map，团队未在 `.npmignore` 中排除 `.map` 文件，也未在构建配置中禁用此功能。

泄露规模惊人。**~1,900 个 TypeScript 文件、512,000+ 行严格模式代码**，涵盖了 Claude Code 的全部内部实现。数小时内，多个 GitHub 镜像仓库出现（instructkr/claude-code 获 1,100+ star，Kuberwastaken/claude-code 提供了详尽分析 README，sanbuphy/claude-code-source-code 附带中英文架构文档）。讽刺的是，代码库中包含一个精心设计的「Undercover Mode」，专门防止 Anthropic 内部信息在 git commit 中泄露——而整个源码却因一个构建配置疏忽被打包发布。

Anthropic 官方仓库 github.com/anthropics/claude-code（65,200 star）实际上**不包含应用源码**，仅提供安装指南、CHANGELOG 和插件。真正的核心代码全部在 npm 包内，这次通过 source map 完整暴露。Anthropic 迅速推送了更新移除该文件，但为时已晚。

---

## 核心架构：「简洁得令人意外」的 Agent 循环

多位独立研究者分析后得出一致结论：Claude Code 的核心是一个**极其简洁的 `while(tool_use)` 循环**——内部代号「nO」。没有 critic 模式、没有角色切换、没有复杂的状态机，模型自身的推理能力驱动整个流程：

```
用户输入 → messages[] → Claude API (streaming) → 响应
  stop_reason == "tool_use"?
    是 → 执行工具 → 追加 tool_result → 循环
    否 → 返回文本给用户
```

一个典型任务运行 **5-50 个迭代**。技术栈为 **TypeScript（严格模式）+ Bun 运行时 + React 18 + Ink（终端 UI）+ Yoga（flexbox 布局）+ Zod v4（模式验证）+ Commander.js（CLI 解析）**。值得注意的是，Claude Code **90% 的代码由 Claude 自己编写**——这一数据来自 Anthropic 创始工程师 Boris Cherny 和 Sid Bidasaria 的确认。

### 关键模块一览

| 模块 | 规模 | 功能 |
|------|------|------|
| `QueryEngine.ts` | ~46,000 行 | 核心 LLM API 引擎：流式传输、工具循环、token 追踪 |
| `Tool.ts` | ~29,000 行 | 统一工具接口定义、权限模式、Zod schema |
| `commands.ts` | ~25,000 行 | 斜杠命令注册表与懒执行 |
| `main.tsx` | 785KB | 入口：Commander.js CLI 解析 + React/Ink 渲染 |
| `coordinator/` | — | 多智能体编排系统 |
| `services/compact/` | — | 三层上下文压缩策略 |
| `services/autoDream/` | — | 后台记忆巩固引擎 |

API 调用使用 `@anthropic-ai/sdk` 的 `beta.messages.create` 方法，所有请求默认流式传输。**Sonnet 4/4.5/4.6** 处理核心推理，**Haiku 3.5** 处理轻量任务（配额检查、话题检测、bash 命令安全分类），**Opus 4.6** 用于复杂架构推理和 ULTRAPLAN。

---

## 系统提示词：直接、反谄媚、极度精简

系统提示词由**数十个条件化片段**动态组装，而非单一固定字符串。总计约 **8,000-12,000 tokens**（基础角色 ~2,900 tokens + 工具定义 ~3,000 tokens + CLAUDE.md 内容 + Git 状态注入 + 技能描述）。核心行为指令包括：

- **极度简洁**：「You MUST answer concisely with fewer than 4 lines of text」（回答必须少于 4 行）
- **反谄媚**：「Claude never starts its response by saying a question or idea was good, great, fascinating…」（永远不要以称赞开头）
- **不加注释**：「DO NOT ADD ANY COMMENTS unless asked」
- **不主动 commit**：「NEVER commit changes unless the user explicitly asks」
- **拒绝恶意代码**：即使用户声称是教育目的
- **CLAUDE.md 作为持久记忆**：注入为 user message 而非 system prompt——这是有意设计的上下文处理策略

**系统提醒（System Reminders）**是一个关键工程决策。Claude Code 不仅在系统提示词中给出指令，还在**工具返回结果和用户消息中嵌入指令**。例如，每次工具调用后会注入更新的 TODO 列表状态，在对话开始时注入「不要创建不必要的文件」。研究者 Jannes Klaas 发现，这种方式比仅依赖系统提示词的指令遵从率**显著更高**——因为在长会话（几百步）中，模型容易「遗忘」系统提示中的指令，而嵌入在最近的工具结果中的提醒效果更好。

---

## 40+ 工具系统：原语优于集成

Claude Code 定义了 **40+ 权限门控工具**，但核心设计哲学是**少量强大的原语组合**，而非大量专用集成。14 个核心工具包括：

- **命令行**：BashTool、GlobTool、GrepTool（ripgrep 封装）、LS
- **文件操作**：FileReadTool、FileWriteTool、FileEditTool（精确字符串替换）、MultiEditTool
- **Web**：WebSearchTool（服务端搜索，每次 $0.01）、WebFetchTool
- **编排**：TodoWriteTool（JSON 任务管理）、AgentTool/Task（子智能体生成）
- **推理**：ThinkTool（无副作用的推理日志）

系统提示词明确要求：「Use Read instead of `cat`」「Use Grep instead of `grep` or `rg` via Bash」——专用工具永远优先于 Bash 等价命令，以提高透明度和安全性。

### 三个精妙的工具工程细节

**延迟工具发现（Deferred Tool Discovery）**：~18 个工具标记为 `shouldDefer: true`，隐藏直到通过 `ToolSearchTool` 明确搜索。这使基础 prompt 保持在 **200K tokens 以下**，同时支持弹性发现。当 MCP 工具描述超过上下文窗口的 10% 时，自动切换到 MCPSearch 模式。

**按名称字母排序**：工具注册表按名称排序并不是为了美观——这是一个**关键的 prompt cache 优化**。保持工具顺序跨请求一致，最大化 prompt cache 命中率。Anthropic 工程师确认「Cache Rules Everything Around Me」是核心工程原则。

**流式工具执行**：工具执行在**模型仍在流式输出时即开始**。读只工具可并发执行（最多 10 个），写工具串行执行，通过分区调度减少延迟。

---

## 上下文窗口管理：三层压缩 + 「做梦」系统

上下文窗口管理被 VentureBeat 称为「泄露中对竞争者最有价值的发现」。Claude Code 使用 **~200K tokens 的上下文窗口**（Sonnet 4.0 上为 195,072），采用三层递进压缩策略：

**第一层：MicroCompact（微压缩）**——无需 API 调用，直接在本地编辑缓存内容。基于时间（清除超时工具结果）和大小（截断超标工具输出）。仅压缩 FileRead、Bash、Grep、Glob、WebSearch、WebFetch、FileEdit、FileWrite 的结果，且有 cache-aware 变体保护 prompt cache 完整性。

**第二层：AutoCompact（自动压缩）**——在上下文利用率达 **~92-95%** 时触发。首先剥离旧消息中的图片/文档（替换为 `[image]` 标记），然后按 API 轮次分组消息，调用 Sonnet 4 生成摘要（最多 20K token），用 `CompactBoundaryMessage` 替换原始消息。压缩后重新注入最多 5 个文件 + 技能描述（50K token 文件预算，25K 技能预算）。断路器机制：连续 3 次压缩失败后停止。

**第三层：SnipCompact（激进修剪）**——特性门控，保留「受保护尾部」的历史截断。还有 **Context Collapse**：在 API 返回 413 错误时懒提交阶段化折叠。

**autoDream「做梦」系统**——这是最引人注目的设计之一。一个后台记忆巩固引擎作为 fork 的子智能体运行，有三重触发条件：距上次做梦 ≥24 小时 + ≥5 个会话 + 获取巩固锁。分四个阶段：**Orient → Gather Recent Signal → Consolidate → Prune and Index**，将 MEMORY.md 保持在 200 行 / ~25KB 以下。「做梦」子智能体只有只读 bash 权限——可以查看项目但不能修改，防止维护操作污染主智能体的推理。

---

## 未发布的秘密功能：KAIROS、ULTRAPLAN、BUDDY

源码中大量编译时特性门控（`feature()` from `bun:bundle`）暴露了 **44 个功能标志**，揭示了多个未发布但完整实现的功能：

**KAIROS「永驻 Claude」**——位于 `assistant/` 目录，是一个持久后台守护进程。维护每日追加日志（`~/.claude/.../logs/YYYY/MM/DD.md`），接收周期性 `<tick>` 提示来决定是否主动行动，有 **15 秒阻塞预算**——任何超时操作会被延迟。独占工具包括 SendUserFile、PushNotification、SubscribePR、SleepTool、CronCreateTool。夜间进行记忆巩固「做梦」。

**ULTRAPLAN「远程深度规划」**——将复杂规划卸载到远程 Cloud Container Runtime（CCR）上运行的 **Opus 4.6**，允许最长 **30 分钟**的思考时间。用户在浏览器中审批结果，特殊哨兵值 `__ULTRAPLAN_TELEPORT_LOCAL__` 将结果「传送」回本地终端。

**BUDDY「电子宠物」**——完整的虚拟宠物系统，18 个物种、5 级稀有度（普通 60%、稀有 10%、传奇 1%）、1% 闪光概率、5 项属性（DEBUGGING、PATIENCE、CHAOS、WISDOM、SNARK，范围 0-100）、5 行 12 字符宽的 ASCII 精灵图含动画帧。物种名通过 `String.fromCharCode()` 混淆以防止字符串搜索。使用 **Mulberry32 PRNG**（以 userId 为种子）实现确定性抽卡。预告窗口：**2026 年 4 月 1-7 日**；完整发布门控到 **2026 年 5 月**。

其他门控功能包括：**VOICE_MODE**（语音交互）、**WEB_BROWSER**（通过 Playwright 的浏览器自动化）、**CHICAGO_MCP**（计算机使用/GUI 自动化）、**FORK_SUBAGENT**（智能体分叉）、**ABLATION_BASE**（研究模式，禁用所有安全特性）。

---

## Undercover Mode 与内部代号

泄露中最具争议的发现之一是 **Undercover Mode**（`utils/undercover.ts`）。当 Anthropic 员工（`USER_TYPE === 'ant'`）在公共仓库工作时，系统注入：

> *「You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository. Your commit messages, PR titles, and PR bodies MUST NOT contain ANY Anthropic-internal information. Do not blow your cover. NEVER include: Internal model codenames, unreleased model version numbers, the phrase 'Claude Code' or any mention that you are an AI, Co-Authored-By lines...」*

泄露的内部代号包括：**Tengu**（Claude Code 项目代号，出现在数百个特性标志前缀中）、**Capybara**（Claude 4.6 变体，三个层级：capybara / capybara-fast / capybara-fast[1m]）、**Fennec**（Opus 4.6）、**Numbat**（未发布模型）、**Chicago**（计算机使用功能）。还暴露了 **22 个 Anthropic 私有仓库名**及多个 SDK API 密钥和 GrowthBook 特性门控。

---

## 社区反应与可借鉴的设计思路

开发者社区反应两极分化。Hacker News 上的「dismiss 派」认为「The source code of the slot machine is not relevant to the casino manager」——底层模型才是关键；「务实派」则积极从代码中学习架构模式。Reddit 上有开发者开玩笑说「Claude has awakened and decided to open-source Claude Code」。

**社区高度评价的设计决策**包括：原语优于集成的工具哲学（少量强大原语组合出任何工作流）、三层记忆架构解决「上下文熵」问题、子智能体用于上下文隔离（深度限制为 1 级，Explore 子智能体可能读取 50 个文件约 100K tokens 但只返回 1,500 token 摘要）、以及 prompt cache 优化贯穿整个架构设计。

**受到批评的方面**包括：遥测系统追踪用户骂人频率（「frustration metric」）、React-in-terminal 架构导致**峰值内存 746 MB 且不释放**、Undercover Mode 引发的 AI 透明度伦理争议、以及同样的 source map 泄露在 13 个月内发生了两次。

---

## 结论：简洁的力量与工程的启示

Claude Code 的源码泄露揭示了一个核心洞察：**当底层模型足够强大时，最优架构是最简单的架构**。没有 critic 模式、没有复杂编排框架、没有数据库——一个 `while(tool_use)` 循环加上精心设计的工具和提示词就足以构建最先进的 AI 编程工具。但「简单」不等于「简陋」：prompt cache 优化、三层压缩策略、系统提醒注入、延迟工具发现等细节，展现了对工程边界条件的深刻理解。这套代码最值得学习的不是某个具体实现，而是**一种设计哲学**——Anthropic 工程师称之为「co-evolution design」：harness 被设计为会随模型能力增强而缩减（「The harness is designed to shrink」）。今天的复杂工程围栏，终将被更智能的模型所简化。