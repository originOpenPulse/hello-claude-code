# 08. 子代理与任务协同系统深度分析

在 `claude-code` 中，任务协同系统 (Agent Swarms & Tasks) 是处理复杂工程问题的核心。它允许主智能体将任务分解、委派并分发给多个独立的子代理执行，支持同步阻塞、异步后台、上下文继承 (Fork) 以及远程隔离 (Remote Isolation) 等多种模式。

## 8.1 任务模型：`Task.ts` 与状态机

系统中所有的后台操作都被抽象为 `Task`。核心定义位于 `src/Task.ts`。

-   **`TaskType`**: 定义了任务类型，包括 `local_agent` (子代理), `remote_agent` (远程代理), `local_bash` (后台 Shell), `in_process_teammate` (进程内队友) 等。
-   **`TaskStatus`**: `pending` -> `running` -> `completed` / `failed` / `killed`。
-   **`TaskStateBase`**: 存储 ID、状态、描述、起始时间、输出文件路径 (`outputFile`) 等基础元数据。

所有活动任务都维护在全局 `AppState.tasks` 中，由 `Task.ts` 提供的 `generateTaskId` 负责生成具有唯一前缀（如 `a` 代表 local_agent, `r` 代表 remote_agent）的随机 ID。

## 8.2 委派入口：`AgentTool`

`AgentTool` 是 AI 启动子代理的唯一接口。

### 8.2.1 启动模式与参数
-   **Teammate Spawn**: 走 `spawnTeammate(...)`，适用于多代理团队模式，常与 `tmux` 或 `in-process` 后端结合。
-   **Local Subagent**: 最常见路径，由 `runAgent(...)` 启动。支持异步 (`run_in_background`) 和同步执行。
-   **Remote Isolation**: 当 `isolation: 'remote'` 时，通过 `teleportToRemote(...)` 在云端创建会话。

### 8.2.2 权限与工具过滤
`AgentTool` 负责构建子代理的“初始视图”：
-   **MCP 要求校验**：确保子代理所需的 MCP Servers 已在线。
-   **权限注入**：子代理可以有独立的 `permissionMode`。异步代理默认 `shouldAvoidPermissionPrompts: true`。

## 8.3 运行时内核：`runAgent` (Async Generator)

`src/tools/AgentTool/runAgent.ts` 是子代理运行的核心逻辑。它被实现为一个**异步生成器 (Async Generator)**，允许逐步产生流式消息。

### 8.3.1 独立上下文与 `AsyncLocalStorage`
由于 Node.js 单线程特性，系统使用 `AsyncLocalStorage` 确保每个并发代理拥有独立的 `SubagentContext`。这保证了监控、日志和工具调用能自动关联到正确的代理 ID。

### 8.3.2 执行生命周期
1.  **Initialization**: 建立专属 MCP 连接，加载所需的 Skills。
2.  **Prompt 构造**: 生成代理专用的 System Prompt，包含环境细节和工具说明。
3.  **The Loop**: 调用内核 `query()` 函数进入消息循环。
4.  **Sidechain Transcript**: 记录子代理的独立会话历史，与主会话隔离。
5.  **Cleanup**: 清理代理专属 MCP、Hooks、临时文件状态和 Perfetto 追踪记录。

## 8.4 协作模型与分叉 (Forking)

### 8.4.1 Coordinator vs. Worker
-   **Coordinator 模式**: 开启 `CLAUDE_CODE_COORDINATOR_MODE` 后，主代理被限制为只能使用 `AgentTool` 和 `SendMessageTool` 进行管理。
-   **Worker 工具裁剪**: 为了防止无限递归，Worker 代理通常拿不到 `TeamCreate` 或 `SendMessage` 工具。

### 8.4.2 分叉模式 (Fork Model)
为了优化 **Prompt Cache** 命中率，子代理在分叉时会尽量继承父代理的会话前缀。
-   **Context Inheritance**: 子代理继承父代理的系统提示词和工具池。
-   **Cache-Identical Prefix**: 源码注释明确指出，这种设计是为了追求极高的缓存命中。

### 8.4.3 工作树隔离 (Worktree Isolation)
涉及大规模文件修改时，子代理可使用临时 Git Worktree。
-   创建隔离副本 -> 在该路径执行修改 -> 任务完成后由用户决定是否合并或删除。

## 8.5 远程会话与 Teleport

远程模式不仅是“换个地方跑”，而是一套完整的隧道化控制协议。

### 8.5.1 `RemoteSessionManager`
负责管理与云端环境的 WebSocket 通信。
-   **Control Request**: 云端代理需要工具权限时，通过 `can_use_tool` 发送控制请求。
-   **Permission Tunneling**: 本地 UI 处理后，将 `allow/deny` 结果回传，实现了“远程执行、本地授权”。

### 8.5.2 `/ultraplan` 与 `/ultrareview`
-   **`/ultraplan`**: 30 分钟远程规划流。用户在浏览器完成规划，系统轮询状态并带回计划。
-   **`/ultrareview`**: 启动远程 Bughunter Fleet（默认 5 个节点），并行扫描 Bug 并将结果回流至本地。

## 8.6 通信平面 (Communication Plane)

### 8.6.1 非对称通信
1.  **指令注入**: 用户追问通过 `queuePendingMessage` 排队。
2.  **边界检查**: 子代理在“工具轮次边界”检查并排空 (Drain) 队列，从而响应新的指令。
3.  **任务通知**: 完成后发送带有 `<task-notification>` XML 的消息，主代理解析此消息向用户汇报。

### 8.6.2 `SendMessageTool`
支持 `uds:` (Unix Domain Socket) 和 `bridge:` 地址，实现跨会话甚至跨机器的 Peer-to-Peer 通信。

## 8.7 关键源码锚点

| 主题 | 代码锚点 | 说明 |
| :--- | :--- | :--- |
| 任务类型定义 | `src/Task.ts:TaskType` | 系统支持的所有后台任务枚举 |
| 代理委派入口 | `src/tools/AgentTool/AgentTool.tsx` | 决定 spawn 路径的核心逻辑 |
| 运行内核 | `src/tools/AgentTool/runAgent.ts:runAgent` | 复杂的 Async Generator 执行链 |
| 远程会话控制 | `src/remote/RemoteSessionManager.ts` | 权限请求与消息流的隧道封装 |
| 协同模式约束 | `src/coordinator/coordinatorMode.ts` | 管理者与工人的权限边界 |
| 远程创建原语 | `src/utils/teleport.tsx` | 云端会话初始化与代码打包逻辑 |

## 8.8 总结
`claude-code` 的多代理系统不仅是简单的后台化，它构建了一套基于工具协议的 AI 编排层。通过 `AsyncLocalStorage` 解决并发冲突，通过统一的 `Task` 抽象管理生命周期，通过远程隧道技术扩展执行边界，最终实现了工业级的 Agent 协作体验。
