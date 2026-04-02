# Claude Code 的记忆系统 (Memory System)

本篇深度分析 Claude Code 如何通过多级记忆机制实现跨会话的知识累积与个性化适配。

## 1. 记忆系统的分类学

在 Claude Code 源码中，记忆并非单一文件，而是由以下几类子系统构成的架构：

### 1.1 核心分类 (Memory Types)
位于 `src/memdir/memoryTypes.ts`：
- **`user`**：用户画像。如用户的职级、技能背景、沟通偏好。
- **`feedback`**：操作指令纠偏。如“以后不要再给这段代码写测试桩”。
- **`project`**：项目背景。如非代码层面的架构决策、截止日期、业务约束。
- **`reference`**：外部系统引用。如 Linear/Slack 频道、Grafana 看板地址。

### 1.2 存储形态
- **`MEMORY.md`**：作为索引入口。每个条目建议不超过 150 字符，通过 Markdown 链接指向具体的 Topic Files。
- **Topic Files (`*.md`)**：具体的记忆内容，带有结构化的 Frontmatter（Name, Description, Type）。

---

## 2. 长期持久化记忆 (Durable Memory)

### 2.1 个体与团队记忆 (Team Memory)
源码：`src/memdir/teamMemPrompts.ts`
- **Private Memory**：存储于 `~/.claude/projects/<slug>/memory/`，仅对当前用户可见。
- **Team Memory**：存储于 `.../memory/team/`，通过 Git 或其他同步机制在团队成员间共享。
- **作用域决策**：系统会引导模型判断信息应归入 `private` 还是 `team`（如项目规范应设为 team）。

### 2.2 记忆的检索与召回 (Memory Recall)
源码：`src/memdir/findRelevantMemories.ts`
- **按需召回**：系统并不会将整份记忆目录塞入上下文。
- **Side Query**：通过一次小模型 (Haiku) 的辅助查询，从 Manifest 中筛选出最相关的 5 个 Topic Files，以 `relevant_memories` 附件的形式注入主 Turn。

---

## 3. 连续会话记忆 (Kairos Daily Log)

### 3.1 Kairos 模式：追加式日志
源码：`src/memdir/memdir.ts`
当激活 Assistant 模式或 Kairos 特性时，记忆写入路径会从“直接更新索引”切换为“追加日志”：
- **存储路径**：`logs/YYYY/MM/YYYY-MM-DD.md`。
- **Append-only**：模型仅负责按时间戳记录 Bullet points。
- **设计初衷**：长效会话中，频繁重写索引会导致巨大的缓存抖动。

### 3.2 记忆蒸馏 (Dreaming/Consolidation)
源码：`src/services/autoDream/`
- **定期整合**：后台进程或通过 `/dream` 命令，将碎片化的 Daily Logs 蒸馏回结构化的 Topic Files，并更新 `MEMORY.md` 索引。

---

## 4. 记忆系统的行为约束

### 4.1 记忆漂移防治 (`MEMORY_DRIFT_CAVEAT`)
模型被告知：“记忆说 X 存在，不代表 X 现在还存在”。
- **验证原则**：在基于记忆作出假设前，模型必须先通过 `Read/Grep/Git` 验证当前真实状态。

### 4.2 禁止记录的内容 (Negative Constraints)
显式禁止记录可从仓库直接推导的信息：
- 代码结构、目录树、函数分布。
- Git 历史记录、Blame 信息。
- 已经存在于 `CLAUDE.md` 中的内容。

---

## 5. 辅助记忆机制

- **SessionMemory**：位于 `src/services/SessionMemory/`。主要用于当前会话在 Compact 后的“状态连续性”，它包含任务详情、当前工作流、错误修正等。
- **Agent Memory**：子代理（Sub-agents）拥有独立的记忆目录，用于存储特定类型任务的累积经验。
- **`remember` Skill**：允许用户人工审阅自动提取的记忆，进行手动提升 (Promote) 或清理 (Cleanup)。

## 6. 总结
Claude Code 的记忆系统是一个“自动提取 + 周期蒸馏 + 人工治理”的闭环。它通过严格的分类学和检索策略，确保只有真正具备跨会话价值的非代码上下文被保留，从而实现“越用越懂你”的智能化体验。
