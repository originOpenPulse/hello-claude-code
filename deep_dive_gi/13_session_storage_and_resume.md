# 会话存储与恢复机制：Transcript 持久化、中断检测与 `resume` 语义

本篇深入拆解 Claude Code 如何将内存中的会话状态持久化到磁盘，以及在执行 `/resume` 或 `--continue` 时，如何通过复杂的算法重建会话链并恢复完整的运行时上下文。

## 1. 核心设计：Append-only 会话日志 (Transcript)

Claude Code 的会话存储（Transcript）不仅仅是聊天记录，而是一个 **追加式的会话事件日志 (Append-only Session Event Log)**。

### 1.1 存储内容
关键文件：`src/utils/sessionStorage.ts`

Transcript 文件（`.jsonl`）中记录的内容远超消息本身：
- **Transcript Messages**: 用户 (user)、助手 (assistant)、附件 (attachment) 和系统 (system) 消息。
- **元数据 (Metadata)**: 会话标题、标签、Agent 名称/颜色、模式 (mode)。
- **状态快照 (Snapshots)**: 工作区状态 (worktree state)、文件历史快照 (file history)、归属快照 (attribution)。
- **控制逻辑**: 内容替换记录 (content replacements)、上下文折叠提交 (context collapse commits/snapshots)。

### 1.2 延迟落盘 (Lazy Materialization)
为了避免产生大量的空会话文件，系统采用了延迟创建策略：
- **`materializeSessionFile()`**: 只有当首个真实的 user/assistant 消息产生时，才会真正创建 `.jsonl` 文件。在此之前，元数据和待定条目仅在内存中缓存。
- **排除非核心消息**: `isTranscriptMessage()` 定义了哪些条目属于主链。例如，`progress`（进度条）消息被明确排除在持久化主链之外，以防止在恢复时造成逻辑分支。

## 2. 路径管理与项目解耦

Transcript 的路径不仅取决于 `sessionId`，还深度耦合了 `sessionProjectDir`。

### 2.1 路径推导逻辑
关键代码：`getTranscriptPath()` / `getTranscriptPathForSession()`

- **项目目录优先**: 路径通过 `getSessionProjectDir()` 获取，这确保了在切换 branch 或使用 worktree 操作时，`resume` 看到的路径与真实落盘位置保持一致。
- **子代理独立存储**: 子代理（Subagents）的 transcript 存储在主会话目录下的 `subagents/` 子目录中，如 `session/<id>/subagents/agent-<id>.jsonl`。

## 3. 会话恢复算法：从 JSONL 到 Live Runtime

`resume` 操作不是简单的文件读取，而是一个涉及“预扫描 -> 链重建 -> 状态回灌”的复杂过程。

### 3.1 消息链重建 (Chain Rebuilding)
关键代码：`src/utils/sessionStorage.ts: buildConversationChain()`

恢复的核心不是文件物理顺序，而是 **`parentUuid` 链**：
- **消息 DAG**: 每一条消息都记录了 `uuid` 和 `parentUuid`。恢复算法会扫描文件，根据这对指针重建消息的有向无环图 (DAG)。
- **Legacy Bridging**: 早期版本可能将 `progress` 写入了链中。`loadTranscriptFile` 包含逻辑来识别 `isLegacyProgressEntry` 并跨过它们重新连接真实消息。
- **边界处理**: `compact boundary` 会通过 `logicalParentUuid` 保持逻辑上的连续性。

### 3.2 中断检测与修复 (Interrupt Detection)
关键代码：`src/utils/conversationRecovery.ts: deserializeMessagesWithInterruptDetection()`

系统假设进程可能在任何时刻被杀掉，因此在反序列化时会主动“消毒”：
- **清理不完整状态**: 过滤掉未解析的 `tool_uses`、孤立的 `thinking-only` 助手消息以及纯空白消息。
- **状态迁移**: 自动将旧版的附件类型（如 `new_file`）迁移到当前的 `file` 结构。
- **中断分类**: 检测并标记 `interrupted_prompt` 或 `interrupted_turn`。如果检测到上一次会话在回复中途被强杀，系统可能会注入一条 meta message 以便模型继续生成。

## 4. 状态回灌 (State Re-hydration)

一旦消息链重建完成，`sessionRestore.ts` 负责将结果重新注入全局状态 `AppState`。

### 4.1 恢复面 (Recovery Facets)
关键代码：`src/utils/sessionRestore.ts`

- **工作区恢复**: `restoreWorktreeForResume()` 恢复到正确的 `cwd`。
- **文件历史**: `copyFileHistoryForResume()` 重新挂载文件的修改历史。
- **Agent 配置**: `restoreAgentFromSession()` 恢复 Agent 类型、模型覆盖 (model override) 和任务描述。
- **上下文管理**: 重新加载上下文折叠 (context collapse) 状态。

## 5. 关键源码锚点

| 功能模块 | 关键文件/函数 | 核心逻辑 |
| :--- | :--- | :--- |
| **存储定义** | `sessionStorage.ts: isTranscriptMessage` | 定义持久化主链的边界 |
| **延迟创建** | `sessionStorage.ts: materializeSessionFile` | 减少孤儿会话文件的产生 |
| **路径推导** | `sessionStorage.ts: getTranscriptPath` | 与 `sessionProjectDir` 强绑定 |
| **消息反序列化** | `conversationRecovery.ts: deserializeMessages` | 中断检测与数据迁移 |
| **链重建算法** | `sessionStorage.ts: loadTranscriptFile` | 预扫描、大文件裁剪与父子链连接 |
| **状态回灌** | `sessionRestore.ts: restoreSessionStateFromLog` | 将磁盘状态映射回 `AppState` |

## 6. 总结

Claude Code 的会话恢复体系是一个三层架构：
1. **持久化层 (`sessionStorage.ts`)**: 负责高效、可靠地将事件流写入追加式日志。
2. **逻辑恢复层 (`conversationRecovery.ts`)**: 负责将原始日志转换回逻辑一致的消息序列，并处理异常中断。
3. **运行时挂载层 (`sessionRestore.ts`)**: 负责将会话语义重新注入到 Live Session 中，使交互无缝延续。
