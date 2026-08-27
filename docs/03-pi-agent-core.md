# pi-agent-core — agent 运行时

`packages/agent`，包名 `@earendil-works/pi-agent-core`。有状态的 agent，带工具执行与事件流式输出，构建于 `pi-ai` 之上。核心文件：`agent.ts`（`Agent` 封装）、`agent-loop.ts`（底层循环）、`types.ts`、`harness/`（AgentHarness）、`search/`（会话全文搜索抽象）、`proxy.ts`。

## 1. Agent 循环（`agent-loop.ts` + `agent.ts`）

### 事件模型（`types.ts`）

- **run**：一次 `prompt()` / `continue()` 调用产生一个 run。
- **turn**：一次 assistant 响应 + 它触发的所有工具调用与结果。
- 事件（`AgentEvent`）：`agent_start` / `agent_end`（run 级）、`turn_start` / `turn_end`（turn 级）、`message_start` / `message_update` / `message_end`（消息级，`message_update` 携带流式增量 `assistantMessageEvent`）、`tool_execution_start` / `tool_execution_update` / `tool_execution_end`（工具级）。

### Agent 消息（`AgentMessage`）

`Message`（user/assistant/toolResult）∪ `CustomAgentMessages`（通过 declaration merging 扩展的自定义应用消息，如 UI 通知、artifact）。

### 工具（`AgentTool`）

- `label`（UI 显示）、`prepareArguments`（参数兼容层，可选）、`execute(toolCallId, params, signal, onUpdate)`（抛异常即失败）、`executionMode`（单工具覆盖）。
- 结果 `AgentToolResult<T>`：`content`（文本/图像，回喂给模型）、`details`（结构化日志/UI 数据）、`usage`、`addedToolNames`（该结果引入的新工具名）、`terminate`（批量提前结束提示）。
- 钩子：
  - `beforeToolCall` → 返回 `{ block, reason, terminate }` 可阻止执行（循环发错误工具结果）。
  - `afterToolCall` → 返回 `AfterToolCallResult` 字段级覆盖结果（content/details/isError/usage/terminate）。
- 执行模式 `ToolExecutionMode`：`"sequential"`（逐个执行）或 `"parallel"`（预检顺序化，允许的工具并发执行；`tool_execution_end` 按完成序发射，消息构件按 source 序）。默认 parallel。

### 循环钩子（`AgentLoopConfig`）

- `convertToLlm`：把 `AgentMessage[]` 转成 LLM 消息（过滤 UI-only 消息）。
- `transformContext`：LLM 调用前的上下文变换（裁剪、注入外部上下文）。
- `getApiKey`：每次调用动态解析 API key（应对短期 OAuth token 过期）。
- `shouldStopAfterTurn`：turn 完成后请求优雅停止（如上下文将满）。
- `prepareNextTurn`：turn 后返回替换的 context/model/thinking 状态以影响下一轮。
- `getSteeringMessages`：中途注入 steering 消息；`getFollowUpMessages`：本会停止时注入后续消息继续。
- `steeringMode / followUpMode`（`QueueMode`）：`"all"`（排空全部队列）或 `"one-at-a-time"`（每次只取最旧一条）。
- `thinkingBudgets`、`maxRetryDelayMs`（重试退避上限）、`sessionId`、`transport`（会话亲和性传输）。

### 契约约束

- `StreamFn` **不得 throw/reject**：失败必须编码为流内事件并以 stopReason `"error"` / `"aborted"` + errorMessage 的最终消息结束。
- 所有钩子不得 throw，出错时返回安全回退值。

## 2. Agent（`agent.ts`）

`Agent` 是有状态封装：拥有 `AgentState`（systemPrompt、model、thinkingLevel、tools、messages、isStreaming、streamingMessage、pendingToolCalls、errorMessage）、事件订阅（`subscribe`，回调可异步）、steering/follow-up 队列（`steer()` / `enqueueFollowUp` 等）、`prompt()` / `continue()` / `abort()`、`convertToLlm` 默认实现（过滤只留 user/assistant/toolResult）。`streamFn` 可通过 `setDefaultStreamFn` 设置默认值。

## 3. AgentHarness（`src/harness/`）

更底层、面向**持久化与崩溃恢复**的会话状态机，是独立于 `Agent`/`agent-loop` 的另一套运行时（coding-agent 不使用它）。详细实现规范见 `packages/agent/docs/harness.md`。核心思想：

- **不可变条目（write-once entries）+ 寄存器（registers）**：对话历史是 append-only 的条目流；可变状态（游标、当前模型、分支位置）放在寄存器里。崩溃后可以精确重建，无需"重放即重算"。
- **会话树**：条目组织成对话树（branch），支持 fork、导航、分支索引；context 构建沿分支收集条目。
- **操作状态机**：每次"接受输入 → 生成 → 工具执行"是一个原子操作，落盘成 program counter 样式的状态；恢复时从中断点继续。
- **三套存储**：entries（写一次）、registers（事务性更新）、usage ledger（token/成本）。接口在 `harness/session/`：`SessionRepository` / `Session` / `MemorySessionRepository` 等；JSONL 参考实现位于 `harness/session/jsonl/`（codec/repo/storage/errors）。
- **compaction / branch summarization**：上下文压缩与分支摘要（`harness/compaction/`）。
- **内置工具**（`harness/tools/`）：`read` / `write` / `edit` / `edit-diff`（带 file-mutation-queue 串行化）/ `bash` / `image`，以及 `path-utils`、`tool-context`。技能系统在 `harness/skills.ts`，提示词模板在 `harness/prompt-templates.ts`。
- **env**：`harness/env/nodejs.ts` 提供 Node 环境适配。
- 公开面：`harness/agent-harness.ts` 导出 `AgentHarness` 等。

## 4. 会话搜索（`src/search/`）

`SessionSearch<T>` 接口：`search(text, options)` 返回命中异步迭代器；`createScanningSessionSearch` 是扫描式参考实现（逐条目投影 + 匹配，支持 entryTypes/limit/signal）。

## 5. 其他

- `src/proxy.ts`：把 agent 事件转发到传输的代理（供远程会话使用）。
- `src/stream-fn.ts`：默认 `StreamFn`（基于 `Models.streamSimple`）。
- `src/node.ts`：Node 专属入口（重导出）。

## 典型用法

```ts
import { Agent } from "@earendil-works/pi-agent-core";
import { createModels } from "@earendil-works/pi-ai";

const agent = new Agent({
  initialState: { systemPrompt: "...", model },
  streamFn: models.streamSimple.bind(models),
});
agent.subscribe((event) => { /* 渲染事件 */ });
await agent.prompt("你好");
```
