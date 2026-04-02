# 用户输入、Slash 命令与队列调度深度分析

在 `claude-code` 中，用户输入并不是直接发送给模型的字符串。它经过了一个包含“输入拦截 -> 语义转换 -> 队列调度 -> 并发控制”的复杂流水线。本章将深入分析这一过程。

## 4.1. 输入调度核心：handlePromptSubmit.ts

`handlePromptSubmit` 是所有用户输入的统一入口。它承担了“输入调度器”的角色，主要解决以下问题：

### 4.1.1. 状态判断与调度策略
当用户提交输入时，调度器会根据 `QueryGuard` 的状态决定策略：
- **空闲**：立即进入 `executeUserInput` 流程。
- **忙碌**（已有 Query 运行）：
    - 判断当前正在运行的工具是否可中断（如 `SleepTool`）。如果是，则触发 `abortController.abort()` 尝试打断当前轮次。
    - 将输入包装为 `QueuedCommand` 放入 `messageQueueManager` 中排队。

### 4.1.2. 附件与引用解析
在进入队列或执行前，调度器会：
- 解析粘贴的文本和图片引用（References）。
- 扩展 `expandPastedTextRefs` 将占位符替换为真实内容。
- 处理图片块，只有在 Prompt 中保留了 `[Image #N]` 占位符的图片才会被保留。

## 4.2. 并发控制与同步闸门：QueryGuard

`QueryGuard`（`src/utils/QueryGuard.ts`）是一个轻量级的同步状态机，用于防止多个异步路径同时启动 Query。

- **三种状态**：`idle` (空闲), `dispatching` (调度中), `running` (运行中)。
- **核心机制**：
    - 在 `processUserInput` 之前通过 `reserve()` 抢占状态，防止异步 gap 期间被再次重入。
    - 结合 React 的 `useSyncExternalStore`，使 UI 能够实时响应系统的“忙碌”状态。

## 4.3. 消息队列管理：messageQueueManager

队列系统不仅仅是简单的 FIFO（先进先出），它具备命令感知的调度能力。

- **QueuedCommand 类型**：区分 `prompt`, `bash`, `slash` 以及 `task-notification` 等。
- **优先级分层**：支持 `now`, `next`, `later` 不同级别的入队。
- **批处理优化**：`queueProcessor` 允许在模型空闲时，将多个排队的 Prompt 合并为一个 Turn 提交，提高交互效率。

## 4.4. 语义转换层：processUserInput

原始文本通过 `processUserInput` 被转换为结构化的消息对象。

### 4.4.1. 模式分发逻辑
- **Bash 模式**：直接调用 `processBashCommand`，通常用于交互式的 shell 操作。
- **Slash 模式**：核心的命令系统入口，调用 `processSlashCommand`。
- **Prompt 模式**：普通的自然语言，但会注入 IDE 选择（IDE Selection）、附件（Attachments）等上下文消息。

### 4.4.2. 前置 Hook 拦截
在消息送往模型前，系统会触发 `UserPromptSubmit` hooks。这些 Hook 可以根据当前输入：
- 注入额外的背景上下文。
- 修改消息内容。
- 直接终止当前 Turn 并返回结果。

## 4.5. Slash 命令调度：processSlashCommand

Slash 命令（以 `/` 开头）是整个扩展体系的生命线。它通过 `getMessagesForSlashCommand` 返回以下丰富的控制元数据：

- **`shouldQuery`**：标识执行完该命令后是否需要继续调用 AI 模型。
- **`allowedTools`**：局部权限覆盖。例如 `/commit` 可以临时锁定模型只能使用特定的 Git 辅助工具。
- **`model` / `effort`**：动态调整当前轮次的模型选择和思考深度。
- **`nextInput`**：支持命令链式调用（Chaining）。

### 4.5.1. 三种执行形态
1. **本地状态突变**：如 `/clear`, `/compact`。直接通过 `setMessages` 修改状态，`shouldQuery` 通常为 `false`。
2. **Local JSX UI**：如 `/config`, `/doctor`。渲染交互式 UI（如模态框），完成后通过 `onDone` 回调返回。
3. **技能扩展 (Skills)**：如 `/commit`, `/review`。本质上是动态生成的 System Prompt，将复杂的指令转换为预置消息后再交给模型。

## 4.6. 后台执行机制：Forked Agents

对于耗时较长的 Slash 命令（标记为 `context: 'fork'`），系统使用 `Context Forking` 机制：
- 创建隔离的子环境。
- 在后台运行独立的 Agent 循环。
- 执行完毕后将结果封装为 `<scheduled-task-result>` 重新进入主队列。

## 4.7. 总结

`claude-code` 的输入系统是一个高度健壮的交互流水线。它通过 `QueryGuard` 解决了异步重入问题，通过 `messageQueueManager` 实现了灵活的任务调度，并利用 `Slash Command` 系统将原本简单的聊天输入框转变成了具备复杂指令控制能力的“会话操作系统”控制台。
