# 1. Claude Code 核心架构深度剖析

本篇对 `claude-code` 的源码架构进行分层拆解，并提供关键代码锚点，用于建立整套系统的全景地图。

## 1.1 架构分层全览

Claude Code 采用了典型的高度解耦分层架构，从入口到核心引擎分为六层：

### 1. 引导与入口层 (Entry & Bootstrap)
- **关键文件**: `src/entrypoints/cli.tsx`, `src/main.tsx`, `src/entrypoints/init.ts`
- **职责**: 负责最轻量级的启动（fast-path）、性能打点、早期输入捕获、CLI 参数定义以及全局服务的懒加载。

### 2. TUI 交互层 (UI & Interaction)
- **关键文件**: `src/replLauncher.tsx`, `src/screens/REPL.tsx`, `src/components/App.tsx`, `src/ink.ts`
- **职责**: 基于 React + Ink 实现终端渲染。管理会话 UI 状态（消息流、输入框、加载指示器、Sidebars），并作为交互控制器驱动底层引擎。

### 3. 状态管理层 (State Management)
- **关键文件**: `src/state/store.ts`, `src/state/AppStateStore.ts`, `src/state/AppState.tsx`
- **职责**: 维护整个 REPL 的运行时状态，提供高性能的订阅机制。核心抽象为 `AppState`。

### 4. 输入编排与命令分发 (Orchestration & Commands)
- **关键文件**: `src/utils/handlePromptSubmit.ts`, `src/commands.ts`, `src/utils/processUserInput.ts`
- **职责**: 解析用户输入的 Prompt、Slash Commands 和附件。决定是直接在本地执行命令，还是将 Prompt 送入 AI 引擎。

### 5. AI 执行引擎 (AI Engine & Runtime)
- **关键文件**: `src/query.ts`, `src/QueryEngine.ts`, `src/services/api/claude.ts`
- **职责**: 系统的“大脑”。负责组装系统提示词、管理上下文压缩（Compaction）、调用 Claude API、处理流式响应，并递归调度工具执行。

### 6. 工具体系与扩展 (Tools & MCP)
- **关键文件**: `src/Tool.ts`, `src/tools.ts`, `src/services/mcp/client.ts`, `src/services/tools/toolExecution.ts`
- **职责**: 定义了 50+ 核心工具（Bash, Edit, Read 等）和 MCP (Model Context Protocol) 客户端，使模型具备与本地系统及第三方服务交互的能力。

---

## 1.2 核心组件深度代码分析

### 1.2.1 `entrypoints/cli.tsx`: 极致性能的引导者
这是物理入口，其设计哲学是“能不加载就不加载”：
- **Fast-path**: 直接在 `main()` 中检查 `--version`，若匹配则直接输出并退出，耗时极低。
- **早期输入捕获 (`startCapturingEarlyInput`)**: 在 Node.js 加载庞大模块的几百毫秒内，已经开始缓冲用户键盘输入，确保存量字符不丢失。
- **动态导入**: 使用 `await import("../main.js")` 延迟加载主业务逻辑，避开顶层 side-effects 阻塞。

### 1.2.2 `main.tsx`: 业务编排与 Side-effects
系统的逻辑中心，负责组装所有能力：
- **顶层副作用 (Top-level Side-effects)**: 
    - `profileCheckpoint`: 性能监控起点。
    - `startMdmRawRead()`: 并行启动 MDM（移动设备管理）策略读取子进程。
    - `startKeychainPrefetch()`: 并行读取 macOS Keychain。
- **Commander 深度集成**: 定义了极其详尽的 CLI 选项，并利用 `preAction` 钩子执行最终的 `init()`、`runMigrations()` 和 `initSinks()`。
- **模式分发器**: 决定进入 `launchRepl` (交互) 还是 `runHeadless` (非交互)。

### 1.2.3 `query.ts`: AI 循环的状态机
核心函数 `async function* query()` 是一个复杂的异步生成器：
1. **组装上下文**: 合并系统提示词、用户附件、Git 状态和历史消息。
2. **上下文管理**: 调用 `snip` 或 `autocompact` 处理超长对话，维持 Prompt Cache 的稳定性。
3. **API 交互**: 通过 `callModel` 发起流式请求。
4. **工具调度**: 当模型返回 `tool_use` 时，通过 `runTools` 触发本地执行，并将结果回填，开启 `next_turn` 递归。

### 1.2.4 `setup.ts`: 环境与安全边界
负责建立本次 Session 的物理运行环境：
- **Git Worktree 支持**: 在 `--worktree` 模式下自动创建隔离的 Git 目录。
- **会话持久化**: 管理 `sessionId` 的生成与存储路径。
- **终端恢复**: 处理 iTerm2 等终端的状态备份与恢复。

---

## 1.3 关键数据协议 (The Unified Protocols)

系统的高效运行依赖于四个核心抽象协议：

1. **Message (`src/types/message.ts`)**: 统一了 `User`, `Assistant`, `System`, `Progress`, `ToolUseSummary` 等所有消息类型，作为 UI 和引擎间唯一的通信载体。
2. **Tool (`src/Tool.ts`)**: 标准化的工具接口，定义了 `inputSchema` (Zod)、执行逻辑 `call()` 以及权限元数据。
3. **Command (`src/commands.ts`)**: 斜杠命令（Slash Commands）的统一注册表，将内置命令、插件命令和技能命令归一化。
4. **ToolUseContext (`src/Tool.ts`)**: 注入到每个工具执行过程中的上下文“全家桶”，包含 `getAppState`, `abortController`, `mcpClients` 等。

---

## 1.4 架构设计亮点

- **React in TUI (Ink)**: 这种模式将声明式 UI 开发带入终端，使得管理极其复杂的异步 UI（如多路工具并发执行进度）变得异常简单且类型安全。
- **受控汇聚**: 项目不追求极端的学术分层，而是让 `REPL.tsx` 或 `main.tsx` 成为受控的汇聚点，从而减少了跨层级传递的复杂度，提升了开发效率和运行性能。
- **三层门控架构**: 通过“编译时（Feature Flags） -> 用户类型（Ant-only） -> 远程配置（GrowthBook）”三层逻辑，实现了同一套代码在不同环境下的灵活裁剪。

## 1.5 关键源码位置指南

| 模块 | 关键代码锚点 |
| --- | --- |
| 启动 Side-effects | `src/main.tsx:1-20` |
| CLI 命令定义 | `src/main.tsx:run()` |
| 初始化 Hook | `src/main.tsx:program.hook('preAction')` |
| REPL 入口 | `src/screens/REPL.tsx:onQueryImpl` |
| AI 核心循环 | `src/query.ts:async function* query` |
| 工具主入口 | `src/tools.ts:getTools()` |
| 交互式启动 screens | `src/interactiveHelpers.tsx:showSetupScreens()` |
