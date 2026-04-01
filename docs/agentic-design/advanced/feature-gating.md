---
title: Feature Gating
layout: default
nav_order: 1
parent: 高级主题
grand_parent: Agentic Design Overview
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

---

## 两层系统的协作

### 场景 1：Compile-Time Only（最常见）

```
新功能 KAIROS（always-on Claude）
  ↓ 指定构建目标
  ├─ 构建 ant（内部）：--feature KAIROS=1 → 编译进去
  └─ 构建 public（公开）：--feature KAIROS=0 → 删除代码

结果：公开版本的用户无法用任何方法激活 KAIROS
```

### 场景 2：Compile-Time + Runtime（用于 A/B 测试）

```
新的 memory 系统（已编译进所有版本）
  ↓ 运行时
  ├─ 50% 的用户在 tengu_new_memory=true 组
  └─ 50% 的用户在 tengu_new_memory=false 组
      ↓ 1 周后收集数据、分析
      ↓ 全量发布到所有用户
```

### 场景 3：运行期特定用户

```
高级功能（编译期编译进去）
  ↓ 运行时按用户身份激活
  ├─ Max 订阅用户 → 启用计算机使用（chicago_beta）
  ├─ Pro 订阅用户 → 启用高级 memory（tengu_memory_pro）
  └─ 免费用户 → 基础功能

都在同一个二进制中，只是权限不同
```

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

---

## Design Decision 专栏

### 为什么同时用编译期和运行期？

**只用编译期**：缺点是无法快速 A/B 测试，无法动态启用功能。
**只用运行期**：缺点是内部功能可能被逆向工程，代码暴露在二进制中。
**两者结合**：编译期物理隔离 + 运行期灵活激活 = 最安全、最灵活、最快。

---

## 关键要点

1. **编译期 feature**：Bun 的 `feature()` API，dead code elimination
2. **运行期 flags**：GrowthBook，动态启用特性，支持 A/B 测试
3. **两层协作**：编译决定可能性，运行时决定激活
4. **内部代号隐藏**：Undercover Mode 防止泄露

---

## 深入阅读

### 相关文档

- [Bootstrap & Cold Start](../session-state/bootstrap.md) — GrowthBook 在 bootstrap 时初始化
- [Token Compression](../context-memory/compression.md) — HISTORY_SNIP 功能标志控制压缩行为

### 源代码

- `bootstrap/state.ts`：运行时 flag 初始化（56 KB）
- `constants/betas.ts`：Beta feature 列表
- `entrypoints/cli.tsx`：编译期条件编译示例
- `utils/undercover.ts`：Undercover Mode 实现
- `services/analytics/`：GrowthBook 集成
