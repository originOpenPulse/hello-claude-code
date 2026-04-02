# `QueryEngine` 与 SDK/非交互路径

本篇分析 `QueryEngine` 类，它是 Claude Code 脱离 REPL 交互界面、在 headless/SDK 模式下运行的核心编排器。

## 1. `QueryEngine` 的定位

**代码位置**: `claude-code/src/QueryEngine.ts`

`QueryEngine` 将交互式 `ask()` 函数中的核心逻辑解构并封装进类中。它不仅是一次性的函数调用，而是**面向一个非交互会话的执行控制器**。

它维护的持久状态包括：
- `mutableMessages`: 长期保存的消息历史。
- `readFileState`: 文件内容缓存。
- `totalUsage`: 累计 Token 使用量。
- `permissionDenials`: 累计权限拒绝记录。

## 2. `submitMessage()`: headless 对话的主入口

**代码位置**: `claude-code/src/QueryEngine.ts:209-1047`

这是 SDK 模式下的“核心循环”。每一轮调用模拟了 REPL 的输入处理链路，但去除了 UI，改为生成 `AsyncGenerator<SDKMessage>`。

### 2.1 构造执行上下文
在进入 `query` 内核前，`QueryEngine` 必须重建一个等效的运行时环境：
- **权限包装 (Wrapped canUseTool)**: 在 SDK 模式下，权限拒绝不仅会终止执行，还必须被显式捕获并计入 `permission_denials` 数组返回给 SDK 调用方。
- **System Prompt 组装**: 调用 `fetchSystemPromptParts` 聚合默认提示词、自定义提示词、MCP 指示以及 **Memory Mechanics Prompt**（后者在显式配置内存路径时注入，告知模型如何操作 `MEMORY.md`）。

### 2.2 输入语义的复用
关键点：`QueryEngine` 并没有为 SDK 单独实现一套逻辑，而是复用了 REPL 所使用的 `processUserInput(...)`。
- 这意味着：Slash Commands（如 `/compact`）、文件附件、技能/插件注入在 SDK 模式下依然生效。

## 3. 恢复性设计：提前持久化 Transcript

**代码位置**: `claude-code/src/QueryEngine.ts:436-463`

这是源码中一个非常深刻的细节：
```typescript
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages)
  if (isBareMode()) {
    void transcriptPromise
  } else {
    await transcriptPromise
    // 必要时 EAGER_FLUSH
  }
}
```
**设计哲学**: 如果在 API 响应返回前进程被意外杀掉，如果没有提前写入 Transcript，下一次 `--resume` 将找不到该会话。`QueryEngine` 确保用户消息在发送给 API 之前就已经落盘，极大地提高了长效对话的容错能力。

## 4. `system_init`: 核心能力的透明化

在第一轮对话开始前，`QueryEngine` 会通过 `yield buildSystemInitMessage(...)` 输出当前会话的能力快照：
- 已加载的 Tools / MCP Clients / Skills / Plugins。
- 当前权限模式 (`PermissionMode`)。
- 主循环模型 (`mainLoopModel`)。
这使得 SDK 消费方能够感知环境能力，决定后续行为。

## 5. 终止条件管理 (Safety Guardrails)

在 headless 环境中，系统必须能自主判断何时停止。`QueryEngine` 额外管理以下边界条件：
- **Max Turns**: 达到硬性轮次上限（通过 `max_turns_reached` 附件发出信号）。
- **Max Budget USD**: 达到费用配额上限。
- **Structured Output Retries**: 如果开启了 `jsonSchema` 约束，系统会对无效的结构化输出进行重试，缺省上限为 5 次。

## 6. QueryEngine 与 `query()` 的关系

- `query()`: 无状态的消息状态机内核，负责 API 调用和工具循环。
- `QueryEngine`: 负责**会话状态维护**、**SDK 协议翻译**（将内部 Message 归一化为 `SDKMessage`）以及**生命周期钩子管理**。

## 7. 总结

`QueryEngine` 的设计体现了 Claude Code 极高的架构质量：**逻辑内核（query）与表现层（REPL/SDK）解耦**。无论 UI 如何变化，对话的语义、权限系统和上下文治理逻辑始终保持绝对一致。
