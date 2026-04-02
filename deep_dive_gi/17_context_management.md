# Claude Code 的上下文管理 (Context Management)

本篇深入分析 Claude Code 如何通过多级治理流水线解决长会话中的 Token 膨胀、Prompt Cache 命中与上下文溢出问题。

## 1. 上下文管理的阶梯体系

Claude Code 并非简单地截断历史，而是构建了一个按需触发、从轻到重的梯度治理体系：

1. **Tool Result Budget**：限制单条工具结果的尺寸。
2. **Snip**：轻量级裁剪。直接删除中间的历史消息，通过 UUID 重新链接消息链。
3. **Microcompact**：细粒度清理工具结果（重点详述见下文）。
4. **Context Collapse**：投影视图管理。通过类似 Git Commit Log 的机制维护会话的“压缩投影”。
5. **Autocompact**：重型压缩。在 Token 接近限制时触发全文总结。

---

## 2. Microcompact：微观治理的双重路径

位于 `src/services/compact/microCompact.ts`，这是最频繁执行的清理机制：

### 2.1 基于时间的清理 (Time-based Microcompact)
- **触发条件**：当两次消息之间的间隔超过阈值（如 5 分钟），系统判定 Prompt Cache 已经“冷却”。
- **清理动作**：直接修改本地消息，将旧的 `tool_result.content` 替换为 `[Old tool result content cleared]`。
- **目的**：既然缓存已冷，不如彻底瘦身再重新加载。

### 2.2 基于缓存的清理 (Cached Microcompact)
- **核心技术**：使用 `cache_edits` 和 `cache_reference` API。
- **清理动作**：**不修改**本地消息内容。在 API 请求层通过 ID 指定要删除的工具调用块。
- **目的**：在不破坏服务端 Prompt Cache 前缀的前提下，动态移除不需要的历史结果块。

---

## 3. Autocompact：宏观治理的自适应逻辑

位于 `src/services/compact/autoCompact.ts`：

### 3.1 触发阈值
公式：`EffectiveContextWindow - 13,000 Tokens`。
- 系统预留约 1.3 万 Token 的缓冲区，防止在处理长任务时突然溢出。

### 3.2 Compaction 路径决策
当触发 Autocompact 时，系统会优先尝试：
1. **Session Memory Compaction** (快路径)：
   - 源码：`sessionMemoryCompact.ts`。
   - 逻辑：直接利用 `SessionMemory` 已经提取出的结构化 Notes，将最近的 N 条消息拼接到其后，快速构建一个“后压缩视图”。
2. **Traditional Compaction** (回退路径)：
   - 逻辑：启动一个专门的 `compact` 代理，通过 `compactConversation` 生成全文摘要。

---

## 4. 关键概念：Compact Boundary

- **隔离历史**：在会话数组中插入 `compact_boundary_marker`。
- **模型视图**：模型只能看到 Boundary 之后的消息。
- **UI Scrollback**：REPL 界面依然保留完整历史，确保用户体验连贯。
- **持久化**：Boundary 状态会写入 Transcript，确保会话恢复 (Resume) 时上下文依然准确。

---

## 5. 失败恢复：Reactive Compact

如果上述预防措施全部失效（如 API 报 413 错误），系统会进入 `reactiveCompact.ts`：
- **溢出恢复**：紧急截断更多历史，或强制执行最高强度的 Compaction。
- **断路器 (Circuit Breaker)**：如果连续 Compaction 失败（如连续 3 次），系统将停止尝试，防止浪费 API 调用。

---

## 6. 核心源码锚点

- `src/query.ts`：查询主循环，定义了治理流水线的执行顺序。
- `src/services/compact/microCompact.ts`：工具结果清理逻辑。
- `src/services/compact/autoCompact.ts`：自动化压缩调度器。
- `src/services/compact/sessionMemoryCompact.ts`：基于 Session Memory 的快路径压缩实现。
- `src/utils/sessionStorage.ts`：处理 Boundary 与 Snip 在磁盘上的持久化。

## 7. 总结
Claude Code 的上下文管理是一个“动态重计算”的过程。它不仅仅是在节省 Token，更是在不断地为模型筛选出“当前任务最相关的信息”，同时利用缓存编辑技术在性能与成本之间取得最优平衡。
