# 第 8 章 LLM Provider 适配器（llm-pi-ai）

## 8.1 pi-ai 的定位

llm-pi-ai 是 dsh 中最关键的 LLM Provider 适配器插件。它并不是 dsh 仓库里的一部分——它是一个外部插件，通过 Profile 的 `$include` 引入。它的作用是将 dsh 的 `LlmCallConfig` 转换为目标 LLM 后端 API 格式的 HTTP 请求。

`pi-ai` 的命名来自"Pluggable Inference API"——它是一组能够理解和转换多种 API 格式的抽象接口。dsh 官方为 Anthropic API、OpenAI API、Google Gemini API 等提供了 pi-ai 的实现。用户也可以编写自己的 pi-ai 适配器。

## 8.2 与 dsh 的边界

pi-ai 通过实现 `LlmAdapter` 接口接入 dsh：

```typescript
// 伪代码：pi-ai 注册到 dsh
ctx.llm.registerAdapter(
  ['opus', 'sonnet', 'claude-code'],  // provider 路由
  piAiAdapter                           // LlmAdapter 实现
)
```

pi-ai 向 dsh 注册了三个路由：`opus`、`sonnet`、`claude-code`——但这些名字在 dsh 内部只是字符串标签。当 Agent 配置 `provider: opus` 时，llm-runtime 找到路由 `opus` 对应的 pi-ai adapter。

pi-ai 接收的 `LlmCallConfig` 是 dsh 的抽象格式（`system` string + `messages: Message[]` + `tools: ToolSchema[]`）。它将其转换为目标 API 格式。对于 Anthropic，它转换为 `/v1/messages` 格式；对于 OpenAI，它转换为 `/v1/chat/completions` 格式。

## 8.3 StreamChunk 的规范

dsh 定义了九种 StreamChunk 类型：

```typescript
type StreamChunk =
  | { kind: 'text'; delta: string }
  | { kind: 'reasoning'; delta: string }
  | { kind: 'tool-call-begin'; id: string; name: string }
  | { kind: 'tool-call-end' }
  | { kind: 'content-block-delta'; index: number; delta: string }
  | { kind: 'input-json'; json: string }
  | { kind: 'signature'; signature: string }
  | { kind: 'usage'; inputTokens: number; outputTokens: number; ... }
  | { kind: 'error'; ... }
```

pi-ai 的工作就是将目标 API 的 SSE 事件转换为这些 chunk。例如 Anthropic 的 `content_block_delta` 事件转换为 dsh 的 `content-block-delta` chunk，OpenAI 的 `choices[0].delta.content` 转换为 dsh 的 `text` chunk。

反过来，pi-ai 也处理 dsh → API 的转换：将 `ToolSchema` 转换为 Anthropic 或 OpenAI 的工具格式，将 `{{variable}}` 插值后的 system prompt 转换为目标 API 的 `system` 参数。

## 8.4 Credentials 系统

pi-ai 不直接处理 API Key。它使用 dsh 的 credentials 系统。Credentials 是一个独立的 Cordis Service，在当前用户配置目录（如 `~/.dsh/credentials.yaml`）中存储加密的 API Key。

pi-ai 的实例化过程涉及两个阶段：

1. **注册阶段**：pi-ai 告诉 dsh 它支持的 provider 列表、需要的 credential 键名等。
2. **调用阶段**：当 dsh 请求 `stream(config)` 时，pi-ai 从 credentials Service 获取 API Key，拼接到 HTTP 请求的 Authorization header 中。

Credentials 与 provider 的分离意味着：Agent 配置只写 `provider: opus`，实际使用的 API Key 由 credentials 系统根据当前登录用户决定。不同的用户登录同一个 dsh 实例时，请求相同的 provider 但使用各自的 API Key。

## 8.5 Attribution 机制

dsh 在 `llm` 包中包含一个 `attribution.ts` 模块，用于在每次请求和响应中嵌入可追溯的元信息。Attribution 的核心是：dsh 在发送给 LLM Provider 的请求中注入一个特殊字段（如 Anthropic 的 `anthropic-beta` header 中的 attribution 字段，或 OpenAI 的 `user` 字段的一个结构化值），使 LLM Provider 能够返回与该请求相关的诊断信息。

这个机制在调试时很有用：当 LLM Provider 返回错误时，attribution 信息帮助快速确认错误来自哪个 dsh 实例、哪个 session、哪个 turn 和 step。

## 8.6 pi-ai 的休眠挂载

pi-ai 使用"休眠挂载"（lazy mount）模式：它的注册阶段耗时很短（只是注册 provider 名称和能力），实际的网络连接和 model 探测在首次请求时才触发。这意味着即使 Agent 配置了某个 provider，只要模型请求没有到达那个 provider，pi-ai 就不会建立连接。

这也意味着 Credentials 可以在运行时变化——pi-ai 在每次请求时才从 credentials Service 读取 API Key，而不是在注册阶段缓存。

## 8.7 配置模型选择

`agent-default-model` 是一个轻量级的 Service，通过 `installSettingsSection` 暴露可热更新的 provider/model/reasoningEffort 选择：

```typescript
// packages/core/agent-default-model/src/index.ts 的伪逻辑
ctx.llm.registerConfigurableProviders([{
  provider: 'opus',
  displayName: 'Claude Opus',
  settingsNs: 'models',
  settingsPath: ['model-opus'],
}])
```

这个模组告诉 settings Service：有一个可配置的 provider `opus`，用户在 settings UI 中可以修改它的配置。Settings 的变化通过 `settings/updated` 事件触发 `llm/adapters-updated`，dsh 重新解析配置。

模型选择与 Agent 循环的交互点在第 5 章中描述的 `agent/request` waterfall——这是每个请求最终被路由到具体 adapter 的中转站。默认路由由 `agent-default-model` 的配置决定，但任何监听 `agent/request` 的插件都可以临时覆盖。

## 思考题

1. pi-ai 同时支持多个 API 格式（Anthropic / OpenAI / Gemini）。业务中如何确定当前请求应该用哪种格式转换？
2. Credentials 在每次请求时动态读取，而不是在注册阶段缓存。这个设计的隐患是什么？如果有 abase64 编码的 token 在新凭证就绪前被请求了？
3. `StreamChunk` 中的 `signature` chunk 在什么场景下生成？它和 `input-json` 的关系是什么？
4. pi-ai 作为外部插件不在 dsh 仓库中，这对版本兼容性有什么挑战？dsh 更新了 `LlmAdapter` 接口的 `stream()` 方法签名，外部 pi-ai 会不会静默不兼容？