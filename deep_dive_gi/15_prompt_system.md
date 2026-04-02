# Claude Code 的提示词系统 (Prompt System)

本篇深度分析 Claude Code 中提示词资产的架构设计、运行时装配流程以及针对 Prompt Cache 优化的工程实践。

## 1. 核心定义：提示词即协议

在 Claude Code 中，“提示词”并非一段简单的静态文本，而是一个高度结构化、多层次装配的运行时协议。它决定了 Claude 如何在权限受限的本地环境中扮演软件工程师的角色。

提示词系统主要由以下层次组成：
1. **主系统提示词 (Main System Prompt)**：定义产品人格与全局行为边界。
2. **环境上下文 (Environment Context)**：注入 OS、Git、工作目录等元数据。
3. **工具描述 (Tool Definitions)**：将每个工具的使用规范作为“微提示词”注入。
4. **用户自定义上下文 (User Context)**：如 `CLAUDE.md` 项目规则。
5. **任务专用提示词 (Task-Specific Prompts)**：Slash commands、子代理 (Sub-agents) 或二级模型任务的专用指令。

---

## 2. 运行时装配链：提示词是如何生成的

### 2.1 基础装配链 (`getSystemPrompt`)
核心源码：`src/constants/prompts.ts`

`getSystemPrompt` 函数通过 `resolveSystemPromptSections` 异步组合多个 section：
- **静态部分**：包括 Intro、System、Doing tasks、Executing actions with care、Using your tools 等。这些部分在所有会话中几乎保持不变，是 Prompt Cache 的核心。
- **动态部分**：包括 `session_guidance`、`memory`、`env_info_simple`、`language`、`output_style`、`mcp_instructions` 等。

### 2.2 缓存边界 (`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`)
Claude Code 使用了一个关键常量 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`。在 API 层 (`src/services/api/claude.ts`)，系统会将数组拆分为：
- **Static Prefix**：边界前的所有内容，使用 `cache_control: { type: 'ephemeral' }` 进行标记，确保跨会话缓存。
- **Dynamic Tail**：包含用户特定信息、记忆、环境详情等，这些内容频繁变化，放在边界后以避免破坏前缀缓存。

### 2.3 API 层的最终修饰
在真正发出请求前，`claude.ts` 还会根据配置注入：
- **Attribution Header**：标识请求来源。
- **CLI Prefix**：针对 CLI 环境的特殊指令。
- **Advisor/Chrome 模式指令**：如果启用了相关功能。

---

## 3. 核心提示词资产深度解析

### 3.1 产品人格定义 (`# Doing tasks`)
该部分通过大量“不要 (Don't)”指令压制 LLM 的常见负面行为：
- **最小化改动**：不要过度设计，修复 Bug 时不要顺便重构无关代码。
- **真实性原则**：严禁虚报测试结果。如果没跑测试，必须如实汇报。
- **验证优先**：在宣称任务完成前，必须通过测试或脚本验证。

### 3.2 风险控制 (`# Executing actions with care`)
将动作建模为“可逆”与“高风险”：
- **高风险动作**：删除文件/分支、强制推送、修改 CI/CD 管道、发送外部消息等，必须请求用户确认。
- **自主性边界**：即使用户在 `CLAUDE.md` 中授权，也仅限于特定范围。

### 3.3 工具使用协议 (`# Using your tools`)
建立了严格的工具优先级，防止模型滥用 Bash：
- 读文件优先用 `Read` 而非 `cat`。
- 编辑优先用 `Edit` 而非 `sed`。
- 搜索优先用 `Glob`/`Grep` 而非 `find`/`rg`。
- 鼓励**并行调用 (Parallel Tool Use)** 以提高效率。

---

## 4. 子代理与二级任务提示词

### 4.1 子代理提示词 (`AgentTool`)
每个子代理都有独立的人格设定（位于 `src/tools/AgentTool/built-in/`）：
- **Explore Agent**：强只读，侧重快速、并行的代码库探索。
- **Plan Agent**：侧重分析并输出实施方案，不进行实际修改。
- **Verification Agent**：对抗式设计。其目标是“证明代码是坏的”，列出了诸如“验证规避”和“前 80% 诱惑”等失败模式。

### 4.2 任务摘要与压缩 (`Compact Prompt`)
位于 `src/services/compact/prompt.ts`：
- 使用 `NO_TOOLS_PREAMBLE` 禁用工具调用，专注于信息压缩。
- 强制输出结构化的 `<analysis>` 和 `<summary>`，确保关键技术上下文（文件、错误、待办任务）在压缩后不丢失。

---

## 5. 提示词工程的性能优化

1. **Section Memoization**：通过 `systemPromptSection` 包装函数，只有在依赖项变化时才重新计算 section 内容。
2. **MCP Instructions Delta**：针对 MCP 服务，仅在必要时更新指令，避免频繁破坏缓存。
3. **Numeric Length Anchors**：通过明确的字数限制（如“工具调用间文本不超过 25 词”）减少输出冗余。
4. **Scratchpad 指令**：引导模型使用会话隔离的临时目录进行中间计算，避免污染工作区。

## 6. 总结
Claude Code 的提示词系统是其“代理行为”的灵魂。它不追求写出“文采斐然”的回复，而是通过严密的逻辑协议、清晰的风险边界和极致的缓存优化，将一个通用的语言模型约束为一个稳健、高效且安全的本地编程代理。
