# API Provider 选择、请求构造与重试状态机

本篇拆解 `query()` 如何在不同云平台（First-party, Bedrock, Vertex, Foundry）之间路由，以及如何通过 `withRetry` 状态机处理复杂的重试、降级与错误治理。

## 1. Provider 路由：不只是 Client Factory

关键文件：`src/services/api/client.ts`

`getAnthropicClient()` 承担了“环境探测”与“Provider 路由”的双重职责：
- **多平台支持**: 实时支持 Anthropic 直连、AWS Bedrock、Azure Foundry 和 Google Vertex AI。
- **动态认证**:
    - **First-party**: 走 OAuth 流程或 API Key，包含 `checkAndRefreshOAuthTokenIfNeeded()`。
    - **Bedrock**: 自动刷新 AWS 凭证，支持角色继承或静态 Key。
    - **Vertex**: 显式处理 GCP 项目 ID 与凭证刷新，防止在非 GCP 环境下触发 metadata server 的 12s 超时。
- **环境隔离**: Provider 选择被视为进程级的“基座”配置，而非单词请求的即兴选项。

## 2. 请求构造：Params from Context

关键文件：`src/services/api/claude.ts`

请求参数的最终组装由 `paramsFromContext()` 闭包完成，它是一个动态的状态折叠器：
- **模型规范化**: 将 CLI 的模型简称映射为各 Provider 实际支持的模型 ID。
- **Beta Headers 管理**: 系统并不使用固定的 Beta 列表，而是根据功能（如 `THINKING`、`PROMPT_CACHING`、`FAST_MODE`）动态拼装 Beta 头。
- **Latched Headers 机制**: 对于 `fast_mode`、`afk_mode` 等状态，采用“Header 锁定 (Latched)，Body 实时 (Live)”的策略，以在不破坏 Prompt Cache 稳定性的前提下调整运行期行为。

## 3. 重试状态机：`withRetry` 的精细控制

关键文件：`src/services/api/withRetry.ts`

项目通过 `withRetry()` 自行维护了一套复杂的重试逻辑，而非直接依赖 SDK 的默认机制。

### 3.1 529 重试分层 (Capacity Error Management)
- **前台优先**: 仅对用户阻塞等待的源（如 `repl_main_thread`、`agent`）启用 529 (Overloaded) 重试。
- **后台限制**: 建议性的、非用户感知的请求（如标题生成、摘要）在 529 时直接熔断，避免在容量抖动期间放大网关负载。

### 3.2 动态修正策略
- **Context Overflow**: 当触发 Max Tokens 溢出时，状态机会解析错误响应，自动下调下一次尝试的 `max_tokens`。
- **Auth Refresh**: 遇到 401 (Expired) 或特定的 OAuth 废弃错误时，触发凭证刷新并重建 client，而非直接抛错。
- **Fast Mode 降级**: 如果 Fast Mode 持续触发 429 或 529，系统会进入 `cooldown` 状态，临时回退到标准模型。

### 3.3 无人值守模式 (Unattended Retry)
- **心跳维持**: 在长时间限流（429）或重试等待期间，系统会以 `HEARTBEAT_INTERVAL_MS` 发送心跳信号，防止宿主环境（如 SSH 或 CI 运行器）判定进程闲置。

## 4. 错误翻译层：从异常到会话语义

关键文件：`src/services/api/errors.ts`

API 层的错误不会原样上抛给 REPL，而是被“翻译”成 Assistant Message：
- **结构化配额展示**: 对 429 错误解析 `anthropic-ratelimit-unified-*` 响应头，向用户展示具体的重置时间与配额状态。
- **多媒体错误处理**: 将底层的文件/图片校验失败转换为易读的错误说明。
- **语义收割**: 确保 529 等瞬时失败被归类为“模型过载”信号，从而触发 fallback 模型切换。

## 5. 关键源码锚点

| 主题 | 源码路径 | 核心语义 |
| :--- | :--- | :--- |
| **Provider 路由** | `api/client.ts: getAnthropicClient` | 处理不同云平台的认证与 Endpoint |
| **请求组装** | `api/claude.ts: paramsFromContext` | 将 context 状态折叠进 API Payload |
| **重试主循环** | `api/withRetry.ts: withRetry` | 处理 429/529 与 Client 重连 |
| **前台源定义** | `api/withRetry.ts: FOREGROUND_529_RETRY_SOURCES` | 控制重试策略的范围 |
| **错误翻译** | `api/errors.ts` | 将原始 API 错误转为会话消息 |

## 6. 总结

API 提供者与重试层是 Claude Code 的“网络免疫系统”：
1. **多云兼容**: 屏蔽了不同基础设施供应商（First-party vs CSPs）的差异。
2. **主动治理**: 通过 `withRetry` 实现了参数自动修正、凭证热刷新与模型动态降级。
3. **闭环体验**: 通过错误翻译层，将底层的基础设施波动转化为用户（和模型）可理解的交互反馈。
