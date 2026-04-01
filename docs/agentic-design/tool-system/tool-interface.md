---
title: Tool Interface 与执行 Pipeline
layout: default
nav_order: 1
parent: Tool System
grand_parent: Agentic Design Overview
---

# Tool Interface 与执行 Pipeline

## 核心概念：Tool Interface

每个 Tool 必须满足统一接口（`Tool.ts`）：

```typescript
interface Tool<T extends ZodSchema> {
  name: string                    // 模型识别 tool 的名称，如 "FileReadTool"
  description: string             // 模型理解 tool 用途（出现在 system prompt 中）
  inputSchema: T                  // Zod schema，模型生成的参数必须符合
  call(input: z.infer<T>, ctx: ToolUseContext): AsyncGenerator<ToolResult>
  isSafe?: boolean                // 是否"安全"（影响权限决策）
  shouldDefer?: boolean           // 是否延迟发现（见下文）
  internalMetadata?: { ... }      // 内部使用，不会发给模型
}
```

**Tool Execution Context（`ToolUseContext`）** 提供给每个 tool：
- `readFileState`：文件系统缓存，避免重复读取
- `getAppState`：全局 React 状态访问
- `setToolJSX`：向终端 UI 渲染自定义组件
- `mcpClients`：外部 MCP 协议连接
- `abortController`：取消信号

---

## Tool 执行 Pipeline

从"模型调用 tool"到"返回结果"的完整流程：

```
Query Loop 收到 tool_use block
  │
  ├─ 1. 输入验证（Zod schema）
  │     - 确保模型没有传入格式错误的参数
  │     - 路径、字符串类型等全部检查
  │
  ├─ 2. 权限检查（utils/permissions/）
  │     - Mode 检查（bypass / default / auto / dontAsk）
  │     - Always Allow / Always Deny 规则匹配
  │     - YOLO 分类器自动审批
  │     - 如果拒绝 → 返回 permission_denied error
  │
  ├─ 3. 执行 Tool
  │     - 调用 tool.call(input, ctx)
  │     - 得到 AsyncGenerator
  │     - 流式 yield ToolProgressData（实时显示进度）
  │
  ├─ 4. 收集 / 截断结果
  │     - 大输出通过 ContentReplacementState 截断
  │     - 或存为附件，不直接放入 context window
  │
  └─ 5. 发送给 Claude
        - 组合为 tool_result block
        - 追加到 messages 数组
        - 继续 Query Loop
```

---

## 为什么 Tool 返回 `AsyncGenerator` 而不是 `Promise<string>`？

不好的设计：
```typescript
call(input): Promise<string> {
  const result = await readFile(input.path)
  return result  // 10 MB 的代码文件？用户一直等
}
```

更好的设计：
```typescript
call(input): AsyncGenerator<ToolResult> {
  yield { type: "progress", message: "Reading..." }
  const content = await readFile(input.path)
  yield { type: "content", data: content }
  yield { type: "complete", lines: 100 }
}
```

Generator 把**长时间操作的透明度**提升为一级特性——UI 立即显示进度，用户不会以为卡住了。

---

## 进度渲染与 Streaming

Tool 执行时会 emit 类型化的进度数据，实时显示在终端：

| 进度类型 | 场景 |
|---------|------|
| `BashProgress` | Shell 命令的 stdout / stderr 流式输出 |
| `MCPProgress` | 远程 tool 调用状态 |
| `WebSearchProgress` | 搜索词 + 结果数量 |
| `REPLToolProgress` | 代码求值的中间输出 |

大输出通过 `ContentReplacementState` 截断或转为附件，避免撑爆 context window。

---

## 深入阅读

- `Tool.ts`：Tool interface 定义
- `services/tools/StreamingToolExecutor.ts`：并发执行编排
- `utils/permissions/permissions.ts`：权限检查
