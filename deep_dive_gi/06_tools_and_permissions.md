# 06. 工具系统与权限机制深度解析

在 `claude-code` 架构中，工具（Tools）是模型实现外部交互的核心途径。本篇将拆解工具的协议定义、权限判定策略以及执行编排机制。

## 1. 工具协议：`Tool.ts` 与 `ToolUseContext`

`claude-code` 对工具的定义并非简单的函数映射，而是一套高度标准化的协议接口（DSL）。

### 1.1 `ToolUseContext`：工具的运行时宇宙
在 `src/Tool.ts` 中，`ToolUseContext` 承载了工具执行时所需的全部上下文。这不仅包含参数输入，还包含：
- **AppState**: 读写应用的全局状态。
- **McpClients**: 调用已连接的 MCP 服务器的能力。
- **Messages**: 访问对话历史的能力。
- **AbortController**: 用于中断长时间运行的任务（如 Bash 脚本）。

这种设计使得一个工具可以成为“跨领域”的操作。例如，一个工具在执行时可以查询 `readFileCache` 缓存，或者向终端推送 `Notification` 通知。

### 1.2 `ToolPermissionContext`：多维权限环境
权限判定不是一个布尔值，而是一个由 `ToolPermissionContext` 描述的策略环境：
- `mode`: 当前所处的权限模式（如 `default`, `plan`）。
- `alwaysAllowRules` / `alwaysDenyRules`: 用户设置的静态白名单/黑名单。
- `shouldAvoidPermissionPrompts`: 针对非交互环境（如后台脚本）的自动静默策略。

## 2. 工具池装配：`tools.ts`

在每一轮对话开始前，系统会动态组装（Assemble）当前可见的工具池：
1. **基础工具 (Base Tools)**: 加载内建的 `BashTool`, `FileEditTool`, `GlobTool` 等。
2. **MCP 工具**: 将动态连接的 MCP 服务器暴露的功能转化为标准的 `Tool` 对象。
3. **Skill/Agent 工具**: 注入用户自定义的技能（Skills）或触发子代理（Agents）的能力。

## 3. 权限护城河：`useCanUseTool`

这是在工具真正执行（`tool.call()`）之前的核心关卡。权限决策链如下：

1. **静态规则判定**: 匹配 `alwaysAllow` 或 `alwaysDeny` 的预设规则。
2. **自动化校验 (Automated Checks)**: 某些低风险工具或通过 `Classifier`（AI 分类器）自动判定为安全的调用将静默批准。
3. **交互式决策 (Interactive Prompt)**: 当上述规则无法覆盖时，CLI 会弹出一个交互式对话框，向用户显示工具名称、输入参数及潜在影响（如破坏性修改文件），等待人工授权。

## 4. 工具执行编排 (Orchestration)

`claude-code` 在处理模型输出的多个 `tool_use` 块时，采用了精细的并发策略（`src/services/tools/toolOrchestration.ts`）。

### 4.1 并发安全分区 (Concurrency Partitioning)
系统会通过 `partitionToolCalls` 将工具调用分为两类：
- **并行执行组 (Concurrent Batch)**: 标记为 `isConcurrencySafe: true` 的工具（如多个 `ls` 或 `readFile`）。这些工具通过 `Promise.all` 并发运行，极大缩短了等待时间。
- **串行执行组 (Serial Batch)**: 具有副作用的工具（如 `BashTool` 的写入操作或 `FileEditTool`）。系统保证它们按顺序逐一执行，确保文件一致性。

### 4.2 执行流映射 (Stream Mapping)
工具的执行不是阻塞的。`runToolUse` 会将单个工具的执行映射为一个异步流（Async Generator）：
- **Progress Message**: 实时反馈工具进度（如 Bash 命令的实时 stdout 输出）。
- **Result Message**: 工具结束后的最终 `tool_result`。

## 5. 关键内建工具深度解析

### 5.1 BashTool: 执行沙盒化
`BashTool` 不仅仅是 `child_process.spawn`。它具备：
- **安全拦截**: 通过静态分析拦截危险的命令注入（如针对 Zsh 特有的扩展）。
- **自动挂起**: 运行超时或涉及长任务时，支持自动转入后台执行。

### 5.2 FileEditTool: 原子化重构
在执行文件编辑时，它提供了：
- **Staleness Check**: 在修改前检查磁盘上的文件是否被外部改动（防止覆盖）。
- **Fuzzy Matching**: 支持模糊匹配模型生成的代码片段，容忍细微的缩进或空行差异。

### 5.3 AgentTool: 递归递归执行
通过调用 `AgentTool`，主模型可以启动一个完全隔离的子 `QueryEngine`。这实现了复杂的任务分解，让父模型专注于决策，子模型专注于具体执行。

## 6. 总结

工具系统和权限机制共同构成了 `claude-code` 的“操作系统底层”。它通过 Zod 强类型 Schema 约束输入，通过 `useCanUseTool` 构筑安全防线，最后通过 `toolOrchestration` 在性能（并发）与安全性（一致性）之间找到平衡点。这套系统让模型既能像资深工程师一样灵活操作，又时刻处于用户的掌控之下。
