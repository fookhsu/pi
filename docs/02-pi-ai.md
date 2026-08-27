# pi-ai — 统一 LLM 接入层

`packages/ai`，包名 `@earendil-works/pi-ai`。统一 LLM API：provider 集合、自动认证解析、token/成本统计、流式工具调用（含部分 JSON）、图像输入/生成、思考（reasoning）分级。

**注意**：该库只包含支持工具调用的模型——工具调用是 agent 工作流的必需能力。

## 核心概念

### Api 与 Provider

- `Api`：厂商 API 风格，如 `"anthropic-messages"`、`"openai-responses"`、`"openai-completions"`、`"google-generative-ai"`、`"bedrock-converse-stream"`、`"pi-messages"` 等。一个 provider 可以声明它使用哪些 API（泛型 `Provider<TApi>`），从而给模型列表提供类型。
- `Provider`（`src/models.ts`）：
  - `id` / `name` / 可选 `baseUrl` / `headers`
  - `auth: ProviderAuth` —— 必须至少提供 `apiKey` 或 `oauth` 之一；即使只靠环境变量也算 `apiKey` 认证，其 `resolve()` 报告是否已配置。
  - `getModels(): Model[]` —— 同步模型目录（静态 provider 返回内置目录；动态 provider 返回上次 refresh 结果）。
  - `refreshModels?(ctx)` —— 动态 provider 专用：恢复持久化目录，可选联网拉取新目录。
  - `stream / streamSimple` —— 流式实现。
  - 可选 `fetchDeferred / cancelDeferred` —— 延迟（后台）响应。
- 内置 provider 约 30 个（anthropic、openai、google、deepseek、groq、openrouter、xai、mistral、qwen 系、xiaomi 系、amazon-bedrock、cloudflare 系等），集中在 `src/providers/`；模型目录由 `scripts/generate-models.ts` 生成到 `models.generated.ts`（勿手改）。

### Models — provider 集合运行时

`createModels()` 返回 `MutableModels`（`src/models.ts`），职责：

- `setProvider / deleteProvider / clearProviders` 管理 provider 注册。
- `getModels / getModel` 同步读取模型目录（best-effort，坏 provider 不抛异常）。
- `refresh(options)` 并发刷新选中的动态 provider：先恢复缓存目录，再做认证解析与网络拉取；错误与取消以 `ModelsRefreshResult` 返回而不 reject。刷新带**代次（generation）控制**：新一次 refresh 会 abort 上一次，发布（`publish`）只在代次未过期时生效。
- `getAvailable(providerId?)` 返回认证已配置 provider 的模型（可被 `filterModels` 按凭据过滤）。
- `getAuth(provider)` 解析 provider 作用域的认证，带来源标签（供状态 UI 显示）；未配置返回 undefined，刷新失败抛 `ModelsError`（code `"oauth"` / `"auth"`）。
- `login / logout` 运行 provider 自己的登录流程并持久化凭据。
- `stream / complete / streamSimple / completeSimple` 请求入口：`applyAuth` 解析认证后委托给拥有模型的 provider，支持按请求覆盖 apiKey/env/headers、`transformHeaders` 钩子、provider 级 baseUrl 重写。
- `fetchDeferred / cancelDeferred` 延迟响应。
- 认证解析会先尝试持久化凭据（credential store），再退到环境变量（`env-api-keys.ts`）与 provider 静态头。

### 流式事件（`src/utils/event-stream.ts`）

`AssistantMessageEventStream` 是统一的事件流，事件类型（`types.ts`）：

- `start`（部分 AssistantMessage 初值）
- 每块内容三类事件：`text_start/delta/end`、`thinking_start/delta/end`（思考内容增量）、`toolcall_start/delta/end`（工具调用，部分 JSON 经 `partial-json` 增量解析）
- 终态二选一：`done`（成功，reason 为 `stop | length | toolUse | deferred`）或 `error`（reason 为 `aborted | error`，携带最终消息）

流协议约定：先发 `start`，再发增量，最后以 `done` 或 `error` 终止；最终 `result()` 归约为完整 `AssistantMessage`（content + usage + stopReason）。

`lazyStream(model, thunk)`：在订阅者真正消费（`[Symbol.asyncIterator]` 或 `result()`）前不做网络请求。

### Context 与中止

`Context` 携带 session id、telemetry、缓存、平台信息等跨请求数据。`raceWithAbortSignal` / `operationSignal`（`src/utils/abort.ts`）统一处理 AbortSignal；abort 后可通过 `continueWithResponse` 之类机制继续（"Continuing After Abort"）。

### 图像

- 输入：`ImageContent`（base64 data + mimeType），模型元数据 `input: ["text"|"image"]` 声明支持。
- 生成：`src/images-models.ts` / `images-api-registry.ts`，`ImagesApi`（已知 `"openrouter-images"`）。

### 思考分级（Thinking）

- `ThinkingLevel`: `"off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max"`（`"xhigh"/"max"` 仅部分模型族支持）。
- 统一接口 `streamSimple/completeSimple` 自动映射到各 provider 的思考参数；provider 专属选项走 `stream/complete`。
- token 类 provider 支持 `ThinkingBudgets` 每级预算。

### 其他子模块

| 模块 | 说明 |
|---|---|
| `src/api/*.lazy.ts` | 各 API 的懒加载实现，避免启动即加载全部 SDK |
| `src/api/transform-messages.ts` | 消息格式转换 |
| `src/api/openai-prompt-cache.ts` | prompt 缓存控制（`CacheRetention`） |
| `src/api/constrained-sampling.ts` | 约束采样（JSON schema 等） |
| `src/auth/` | `AuthContext`、`CredentialStore`、`resolve.ts`（认证解析）、`oauth/`（设备码流程、OAuth 页面） |
| `src/models-store.ts` | 模型目录持久化（`ModelsStore`：内存 / 文件） |
| `src/utils/` | retry、event-stream、abort、proxy、诊断、token 估算（`estimate.ts`）、溢出处理（`overflow.ts`） |
| `src/compat.ts` | 兼容层导出（旧 API 别名，供 coding-agent 使用） |
| `src/cli.ts` | `pi --list-models` 等 CLI 辅助 |

## 典型用法

```ts
import { createModels } from "@earendil-works/pi-ai";
import { anthropicProvider } from "@earendil-works/pi-ai/providers/anthropic";

const models = createModels();
models.setProvider(anthropicProvider());
const model = models.getModel("anthropic", "claude-sonnet-4-6");

// 流式
const stream = models.streamSimple(model, { sessionId: "s1" });
for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const msg = await stream.result(); // 完整 AssistantMessage
```
