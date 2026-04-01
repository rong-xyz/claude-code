---
title: Feature Gating
layout: default
nav_order: 8
parent: Agentic Design
---

# Feature Gating：内部功能和公开功能的隔离

## 为什么需要 Feature Gating？

Claude Code 是一个复杂系统，有很多功能：

- 一些已发布、稳定、所有用户可用（如 FileEditTool）
- 一些还在测试、只给内部用户（"ant" 用户）（如 KAIROS）
- 一些只给特定高级用户（Max / Pro 订阅）（如计算机使用）
- 一些通过实验 A/B 测试（如新 memory 系统）

如果不做隔离，代码会变得一团糟：

```typescript
// 不好的做法
if (user.isInternal) {
  // KAIROS 模式逻辑
  launchKAIROS()
} else if (user.isMax) {
  // 高级功能
  enableComputerUse()
} else if (featureFlags.has('experimental_search')) {
  // A/B 测试
  enableNewSearch()
} else {
  // 默认行为
  launchDefault()
}
// ... 这样嵌套太深，难以维护
```

Claude Code 的解决方案：**编译期和运行期双层 gating**。

---

## 编译期 Gating：Bun 的 `feature()` API

Bun（JavaScript 运行时）提供了一个编译期特性检测 API：

```typescript
if (feature("KAIROS")) {
  // 这段代码只在 KAIROS 构建中出现
  // 其他构建完全不包含这代码（编译时删除）
} else {
  // 默认行为
}
```

### 工作原理

```
源代码（编译前）：
  if (feature("KAIROS")) {
    launchKAIROS()
  } else {
    launchDefault()
  }
      ↓ Bun 编译（--feature KAIROS=1）
  if (true) {
    launchKAIROS()  // 只保留这段
  } else {
    // 被 dead-code-elimination 删除
  }
      ↓ 死代码消除
  launchKAIROS()  // 最终的可执行代码

---

编译为公开构建（--feature KAIROS=0）：
  if (false) {
    launchKAIROS()
  } else {
    // 只保留这段
    launchDefault()
  }
      ↓
  launchDefault()
```

### 优势

1. **二进制体积小**：内部功能完全不包含在公开二进制中
2. **安全**：用户不能"逆向工程"解锁内部功能（代码不存在于二进制）
3. **性能**：不需要运行时检查这些条件（编译时就决定了）
4. **成本**：公开二进制更小，CDN 传输更便宜

相关代码散布在整个 codebase 中，比如：
- `entrypoints/cli.tsx`
- `bootstrap/state.ts`
- `query.ts`

---

## Compile-Time Feature 列表

Claude Code 在构建时支持这些 feature flag：

| Flag | 启用时 | 作用 | 类型 |
|------|--------|------|------|
| `KAIROS` / `PROACTIVE` | 启动 KAIROS 模式 | 全天候 assistant，后台守护进程 | 内部 |
| `BRIDGE_MODE` | 启用 claude.ai 远程控制 | 在 Web UI 中控制 CLI，双向同步 | 内部测试 |
| `VOICE_MODE` | 启用语音输入 | 用麦克风说指令，而不是打字 | 测试中 |
| `DAEMON` | 启用 daemon 模式 | 后台持久进程，长时间运行 | 测试中 |
| `COORDINATOR_MODE` | 启用 coordinator 多 agent | 复杂任务分解，Research/Synthesis/Implementation/Verification | 测试中 |
| `BUDDY` | 启用 Tamagotchi 伴侣系统 | 养一个宠物 AI，5 级稀有度，闪光概率 1% | 内部娱乐 |
| `WORKFLOW_SCRIPTS` | 启用工作流脚本 | 自动化重复任务，YAML 定义 | 测试中 |
| `NATIVE_CLIENT_ATTESTATION` | 启用本地证明 | 设备信任验证，硬件安全模块支持 | 企业 |
| `TRANSCRIPT_CLASSIFIER` | 启用 AFK 自动模式 | 离开电脑时自动工作（检测闲置） | 测试中 |
| `HISTORY_SNIP` | 启用历史压缩优化 | Context 超限时激进修剪，保留受保护尾部 | 稳定 |
| `EXPERIMENTAL_SKILL_SEARCH` | 启用技能搜索（实验） | 发现新 skill 命令，向量搜索 | 实验 |
| `ABLATION_BASELINE` | 科学研究 baseline | 论文用，禁用所有安全特性和优化 | 研究 |
| `CHICAGO` / `WEB_BROWSER` | 计算机使用功能 | GUI 自动化，通过 Playwright | 高级用户 |

默认只编译：`KAIROS=0, BRIDGE_MODE=0, VOICE_MODE=0, ...`（全是 0，最小二进制）。

内部构建时：编译脚本设置 `--feature KAIROS=1 --feature BRIDGE_MODE=1 --feature COORDINATOR_MODE=1 ...`

相关文件：构建脚本（如 `scripts/build.sh`），或 `bunfig.toml`

---

## 运行期 Gating：GrowthBook Feature Flags

编译期 feature 决定"可能性"（代码是否在二进制中），运行期 flags 决定"激活"（是否使用该代码）。

系统启动时，从 GrowthBook（特性管理平台）获取动态配置：

```typescript
// 启动时
const flags = await growthbook.getFeatures({
  userId: currentUser.id,
  organization: currentUser.org,
  environment: "production"
})

// 现在可以做运行期决策
if (flags.has('tengu_scratch')) {
  // 启用共享 scratchpad（coordinator 模式）
  enableScratchpad()
}

if (flags.has('tengu_amber_flint')) {
  // 启用 in-process teammate swarms
  enableTeammates()
}

if (flags.has('tengu_penguins_off')) {
  // 禁用某个实验（rollback）
  disableExperiment()
}

// 根据订阅级别动态启用
if (user.subscription === "max" && flags.has('chicago_beta')) {
  enableComputerUse()
}
```

### Flag 命名约定

Runtime flags 以 **`tengu_`** 开头（"tengu" 是 Claude Code 的内部代号）：

```
tengu_scratch              # 共享 scratchpad（为 coordinator）
tengu_amber_flint          # In-process teammate
tengu_onyx_plover          # 某个实验
tengu_penguins_off         # 禁用什么功能（灰度发布时用）
tengu_new_memory_v2        # A/B 测试新 memory 系统
chicago_beta               # 计算机使用 beta
```

这样容易区分：
- **编译期**：`KAIROS`、`VOICE_MODE`（大写）
- **运行期**：`tengu_kairos_v2`、`tengu_voice_v3`（小写 + 版本号）

相关文件：
- 初始化：`bootstrap/state.ts`（56 KB）
- GrowthBook 集成：`services/analytics/`

---

## 两层系统的协作

### 场景 1：Compile-Time Only（最常见）

```
新功能 KAIROS（always-on Claude）
  ↓
指定构建目标
  ├─ 构建 ant（内部）：--feature KAIROS=1
  │   → KAIROS 代码被编译进去
  │   → 二进制大小 +5 MB
  │
  └─ 构建 public（公开）：--feature KAIROS=0
      → KAIROS 代码被 dead-code-elimination 删除
      → 二进制中完全没有 KAIROS 的痕迹
      → 二进制大小正常

结果：公开版本的用户无法用任何方法激活 KAIROS
```

### 场景 2：Compile-Time + Runtime（用于 A/B 测试）

```
新的 memory 系统（已编译进所有版本）
  ↓
运行时
  ├─ 50% 的用户在 tengu_new_memory=true 组
  │   → 使用新系统
  │   → 收集性能数据、用户反馈
  │
  └─ 50% 的用户在 tengu_new_memory=false 组
      → 使用旧系统
      → 收集对照组数据
      ↓
1 周后：新系统的 P95 延迟 +2%，但用户满意度 +15%
→ 全量发布（所有用户切换到新系统）
```

### 场景 3：运行期特定用户

```
高级功能（编译期编译进去）
  ↓
运行时按用户身份激活
  ├─ Max 订阅用户 → 启用计算机使用（chicago_beta）
  ├─ Pro 订阅用户 → 启用高级 memory（tengu_memory_pro）
  └─ 免费用户 → 基础功能

这些都在同一个二进制中，只是权限不同
```

相关文件：`services/policyLimits/`（根据订阅级别应用 policy）

---

## 对 System Prompt 的影响

Feature gating 会改变 system prompt 的内容：

```typescript
const systemPrompt = [
  BASE_PROMPT,

  // 编译期条件（如果编译时启用才包含）
  feature("BRIDGE_MODE") ? BRIDGE_MODE_INSTRUCTIONS : "",
  feature("VOICE_MODE") ? VOICE_INSTRUCTIONS : "",

  // 运行期条件（运行时检查）
  flags.get("tengu_coordinator") ? COORDINATOR_INSTRUCTIONS : "",
  flags.get("chicago_beta") ? COMPUTER_USE_INSTRUCTIONS : "",

  // API beta 特性（可协商启用）
  BETAS_NEGOTIATED_WITH_API,  // 如 "interleaved_thinking"
]
```

这也影响 **prompt cache**：

```
缓存键 = MD5(system_prompt_prefix)

如果 flag 变化，cache key 变化，旧 cache 失效

所以：高频变化的 runtime flag 应该放在 cache boundary 之后
（动态部分），而不是之前（静态部分）
```

---

## API Beta 协商

Claude API 定期发布新特性（beta），Claude Code 需要：

1. **请求启用**这些 beta
2. **如果启用**，在 system prompt 中告诉 Claude（这个 API 支持什么新功能）

### 示例：Interleaved Thinking Beta

```typescript
// constants/betas.ts
const BETAS_REQUESTED = [
  "interleaved-thinking",    // 支持在回复中穿插思考
  "structured-outputs",       // 支持返回结构化 JSON
  "context-1m",              // 100 万 token context
  "web-search",              // 网络搜索能力
  "extended-thinking",       // 深度思考模式（最多 32K token 思考）
]

// API 返回确认
const BETAS_ENABLED = [
  "interleaved-thinking",    // ✓ 已启用
  // "structured-outputs",   // ✗ 暂无权限
  "context-1m",              // ✓ 已启用
  "web-search",              // ✓ 已启用
  // "extended-thinking"     // ✗ 还没权限
]

// System prompt 中动态加入
You have access to the following beta features:
  - Interleaved thinking: You can output <thinking> blocks interleaved with text
  - Context 1M: You can use up to 1,000,000 tokens
  - Web search: You can call the WebSearchTool for live information
```

### Beta Negotiation 流程

```
初始化时：
  → 列出 BETAS_REQUESTED
  → 调用 Claude API，请求这些 beta
  ← API 返回 BETAS_ENABLED（已启用的列表）
  → 在 system prompt 中告诉 Claude 哪些 beta 可用

运行时：
  → Claude 可能会使用这些 beta 特性
  → 如果使用了未启用的特性，API 返回错误
  → 系统记录并禁用该特性，重试
```

相关文件：`constants/betas.ts`

---

## 内部代号（Internal Codenames）

源码泄露中暴露了多个内部代号，这些通常被隐藏：

| 代号 | 含义 | 用途 |
|------|------|------|
| `Tengu` | Claude Code 主项目代号 | 出现在 feature flag 前缀、内部文档 |
| `Capybara` | Claude 4.6 变体 | 三个层级：capybara / capybara-fast / capybara-fast[1m] |
| `Fennec` | Opus 4.6 的内部名称 | 用于复杂推理任务 |
| `Numbat` | 未发布模型 | 实验阶段 |
| `Chicago` | 计算机使用功能代号 | 也称 `WEB_BROWSER` |
| `KAIROS` | 永驻 Claude assistant | 后台守护进程 |
| `ULTRAPLAN` | 深度规划引擎 | 最多 30 分钟思考时间 |
| `BUDDY` | 电子宠物系统 | 18 个物种，5 级稀有度 |

**Undercover Mode** (`utils/undercover.ts`) 确保当 Anthropic 员工在公开仓库工作时，不会在 commit message 中泄露这些代号。

```typescript
if (USER_TYPE === 'ant' && IS_PUBLIC_REPO) {
  // 注入提醒
  console.warn("You are operating UNDERCOVER. DO NOT include:")
  console.warn("- Internal model codenames (Capybara, Fennec)")
  console.warn("- Unreleased features (KAIROS, ULTRAPLAN)")
  console.warn("- Feature flag names (tengu_*)")
  console.warn("- Internal project codename (Tengu)")
}
```

---

## Design Decision 专栏

### 为什么同时用编译期和运行期？

**只用编译期**：
- 缺点：无法做快速的 A/B 测试（需要重新编译）
- 缺点：无法根据用户身份动态启用功能

**只用运行期**：
- 缺点：内部功能可能被反编译/逆向工程
- 缺点：内部代码暴露在公开二进制中（体积大）
- 缺点：启动时需要网络请求拿 flags（慢）

**两者结合**（Claude Code 的方式）：
```
编译期决定"可能性"（物理隔离内部代码）
  ↓
运行期决定"激活"（灵活的 A/B 测试）
  ↓
最安全、最灵活、最快
```

### 为什么 Bun 的 `feature()` 而不是其他工具？

Bun 的 `feature()` 是编译期指令，会进行 dead code elimination：

```
其他工具（如 rollup 的条件编译）：
  需要额外的 webpack 插件
  编译配置复杂
  可能有漏网之鱼（某些代码没被删除）

Bun:
  原生支持（不需要插件）
  编译快（Bun 本身编译快）
  输出体积最小（彻底删除死代码）
  集成在 bundler 中（无额外配置）
```

---

## 常见误解

**误解 1**："Runtime flag 可以改变编译期行为？"

实际：不能。Runtime flag 只能在编译期已包含的代码中做选择。

```typescript
// 如果编译时 --feature KAIROS=0，这段代码被完全删除
if (feature("KAIROS")) {
  launchKAIROS()
}

// 运行时即使 growthbook.flags.get("enable_kairos") == true，也没用
// KAIROS 代码根本不存在于二进制中
```

**误解 2**："Feature flag 对性能有开销？"

实际：
- **编译期 feature**：没有开销（代码级别选择）
- **运行期 flag**：有极小开销（map lookup，<1ms）

可忽略不计。

**误解 3**："所有用户都能看到内部代码？"

实际：不能。公开二进制中编译时被删除的代码，用户看不到。只有 ant（内部）用户的二进制才包含 KAIROS、BRIDGE_MODE 等代码。

**误解 4**："Beta feature 会自动启用？"

实际：不会。需要 Anthropic 在 GrowthBook 中明确启用该用户的 beta flag。

---

## 关键要点

1. **编译期 feature**：Bun 的 `feature()` API，dead code elimination，对二进制体积和安全性的保障
2. **运行期 flags**：GrowthBook，动态启用特性，支持 A/B 测试和灰度发布
3. **两层协作**：编译决定可能性，运行时决定激活
4. **System prompt 影响**：不同 flag 组合 → 不同 system prompt → 不同 cache key
5. **API Beta**：协商启用新的 Claude API 特性，在 system prompt 中告诉 Claude
6. **内部代号隐藏**：Undercover Mode 防止 Anthropic 员工在公开仓库泄露内部信息

---

## 深入阅读

- `bootstrap/state.ts`：运行时 flag 初始化（56 KB）
- `constants/betas.ts`：Beta feature 列表
- `constants/system.ts` 和 `constants/systemPromptSections.ts`：System prompt 组装
- `entrypoints/cli.tsx`：编译期条件编译示例
- `utils/undercover.ts`：Undercover Mode 实现
- `services/analytics/`：GrowthBook 集成

---

## 后记

这 6 篇文档覆盖了 Claude Code 最核心的 agentic 设计决策。还有很多其他话题（Bridge 模式、Voice 输入、Plugin 系统、MCP 集成等），但这些是最重要的基础。

如果你想进一步了解，建议：

1. **读完这 6 篇**（你现在正在做的）
2. **打开源代码**，对照本文档的路线图逐个浏览：
   - `query.ts`（~1700 行，Query loop 核心）
   - `Tool.ts`（Tool interface 定义）
   - `tools/AgentTool/AgentTool.tsx`（~233 KB，最复杂的 tool）
3. **运行 Claude Code**，用 `--verbose` flag 看内部日志
4. **探索具体 tool 实现**（如 `tools/BashTool/`、`tools/FileEditTool/`）

### 学习进阶路线图

```
基础（你是这里）：
  [1] Agent Loop
  [2] Tool System
  [3] Multi-Agent Coordination
  [4] Permission System
  [5] Context & Memory
  [6] Feature Gating

中级（理解完整系统）：
  → Query 执行（query.ts 详读）
  → Permission 决策流程（52 KB 的权限代码）
  → Compaction 算法（context 管理）
  → Multi-agent 隔离（AsyncLocalStorage）

高级（扩展和优化）：
  → 添加新 tool（实现 Tool interface）
  → 调整权限规则（rules.json）
  → 优化 prompt cache（system prompt 分层）
  → 实现自定义 MCP server（外部工具集成）
```

祝学习愉快！
