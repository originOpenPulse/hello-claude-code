# 性能、缓存与长会话稳定性专题

本篇收拢启动性能、Prompt Cache、资源释放与长会话稳定性相关的工程设计。Claude Code 的大量复杂度来自对真实成本（Token 费用、内存、延迟）的极致控制。

## 1. 启动性能：三窗口并行与延迟策略

Claude Code 将启动过程拆分为三个精细的执行窗口，以最小化用户感知的延迟。

### 1.1 顶层 Side Effect 窗口
**代码位置**: `claude-code/src/main.tsx:1-25`

在庞大的模块依赖树（import tree）求值期间，第一时间启动不依赖重型库的子进程：
- `profileCheckpoint`: 启动耗时分析。
- `startMdmRawRead()`: 异步读取 MDM 策略。
- `startKeychainPrefetch()`: 提前预取 macOS Keychain/Windows Credentials。

**设计意图**: 利用 V8 解析 JS 的时间，并行处理底层系统 I/O。

### 1.2 Setup 窗口
**代码位置**: `claude-code/src/setup.ts:285-385`

在首轮 `query` 前必须完成的注册与预热：
- `getCommands(getProjectRoot())`: 预热指令系统。
- `loadPluginHooks()`: 加载插件钩子。
- `logEvent('tengu_started', {})`: 发送最早的可靠启动埋点。

### 1.3 延迟预取（Deferred Prefetch）窗口
**代码位置**: `claude-code/src/main.tsx:380-465` (`startDeferredPrefetches`)

将耗时且非阻塞首屏的操作推迟到用户开始打字之后：
- `getUserContext()`: 扫描 `CLAUDE.md` 及其它上下文。
- `getSystemContext()`: 扫描 `git status`。
- `countFilesRoundedRg()`: 估算项目规模。
- `refreshModelCapabilities()`: 异步更新模型能力信息。

## 2. Prompt Cache 稳定性：字节级防御

Prompt Cache 的命中依赖于请求前缀的字节级绝对一致。任何细微的变化（如随机的文件路径）都会导致昂贵的缓存失效。

### 2.1 临时路径的 Content Hash 策略
**代码位置**: `claude-code/src/main.tsx` (`loadSettingsFromFlag`)
当使用命令行 `--settings` 传入配置时，系统不使用 UUID，而是根据内容生成哈希路径：
```typescript
settingsPath = generateTempFilePath('claude-settings', '.json', {
  contentHash: trimmedSettings
});
```
**原因**: 这个路径会进入工具沙盒策略，工具描述参与 Prompt Cache Key。如果路径随机，每次启动都会导致全量缓存失效。

### 2.2 状态粘滞（Sticky-on Latch）
**代码位置**: `claude-code/src/services/api/claude.ts:1350-1700`
对于 Beta 功能（如 Fast Mode, AFK Mode），一旦在会话中启用过，即便后续关闭，请求 Header 也会被“粘滞”锁定：
- 避免 Header 在会话中途来回切换导致 Cache Key 抖动。
- 宁可发送冗余 Header，也要保证前缀稳定性。

### 2.3 缓存失效监控
**代码位置**: `claude-code/src/services/api/promptCacheBreakDetection.ts`
专门记录并对比 `systemHash`, `toolsHash`, `betas` 的变化，将缓存失效视为可观测的“故障”而非正常行为。

## 3. 上下文治理梯度体系

在 `claude-code/src/query.ts` 中，Claude Code 构建了一套由轻到重的上下文压缩管线，而非并行执行。

### 3.1 治理阶梯
1. **Tool Result Budget**: 运行在最前端，限制单个工具结果的大小。
2. **Snip (HISTORY_SNIP)**: 截断过长的历史片段。
3. **Microcompact**: 合并已失效的中间状态（如报错重试栈）。
4. **Context Collapse**: 将陈旧的历史块压缩为精简摘要。
5. **AutoCompact**: 终极手段，重新提取核心记忆并重置会话。

### 3.2 关键参数
- `MODEL_CONTEXT_WINDOW_DEFAULT = 200,000`: 默认上下文窗口。
- `COMPACT_MAX_OUTPUT_TOKENS = 20,000`: Compaction 摘要的输出配额。
- `CAPPED_DEFAULT_MAX_TOKENS = 8,000`: 初始最大输出 Token 限制（通过 `max_output_tokens_escalate` 可动态提升至 64k）。

## 4. 内存与流资源保护

### 4.1 显式流释放（Native 资源清理）
**代码位置**: `claude-code/src/services/api/claude.ts` (`releaseStreamResources`)
Node.js 的 `fetch` Body 持有原生 TLS/Socket 缓冲区。如果只停止迭代而不调用 `body.cancel()`，内存将缓慢泄漏且 Socket 无法重用。Claude Code 在每轮对话结束或中断时强制清理。

### 4.2 进度条原地更新（Ephemeral Progress）
**代码位置**: `claude-code/src/screens/REPL.tsx`
对于长时间运行的 Bash 工具，每秒产生一条 `progress` 消息。系统采用“原地替换”逻辑而非“持续追加”，避免 `messages` 数组膨胀导致的渲染卡顿和 Transcript 体积爆炸。

## 5. 总结

Claude Code 的性能哲学可以概括为：**极致的异步并行、病态的缓存稳定性维护、以及分级演进的上下文治理**。它不仅关注功能的实现，更将“长会话稳定性”作为核心工程目标。
