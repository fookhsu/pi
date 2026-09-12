# agent 包事件流分析

分析 `packages/agent`（`@earendil-works/pi-agent-core`）的事件体系。核心文件：`types.ts`（事件类型）、`agent-loop.ts`（底层循环，事件源头）、`agent.ts`（`Agent` 封装：订阅/状态归约）、`proxy.ts`（事件转发到代理服务端）、`harness/events.ts`（另一套独立运行时的极简事件总线）。

## 1. 三层事件体系

| 层 | 事件类型 | 生产者 | 消费者 |
|---|---|---|---|
| LLM 流 | `AssistantMessageEvent`（pi-ai） | provider 流式实现 | agent-loop 的 `streamAssistantResponse` |
| 循环层 | `AgentEvent` | `agent-loop.ts` 的 `runAgentLoop`/`runLoop` | `Agent.processEvents` |
| 应用层 | 订阅回调 `(event, signal)` | `Agent.processEvents` | 调用方（UI、持久化） |
| harness 层 | `HarnessEvent`（独立运行时） | `HarnessEventBus.emit` | 订阅者/监视者 |

## 2. AgentEvent 类型（`types.ts`）

```
run 级    agent_start / agent_end(messages)
turn 级   turn_start / turn_end(message, toolResults)
消息级    message_start(message) / message_update(message, assistantMessageEvent) / message_end(message)
工具级    tool_execution_start(toolCallId, toolName, args)
          tool_execution_update(toolCallId, toolName, args, partialResult)
          tool_execution_end(toolCallId, toolName, result, isError)
```

- `message_update` 只对 assistant 消息在流式期间发射，且携带底层 LLM 事件（`text_delta` 等）用于增量渲染。
- 工具结果（toolResult）也走 `message_start`/`message_end`，没有独立的 `tool_result` 事件。
- `agent_end` 是 run 的最后一个事件，但它携带的 `messages`（本次 run 新增消息）是事件流 EventStream 的终结值。

## 3. 一次 prompt 的完整事件时序

```
Agent.prompt(text)
└─ runWithLifecycle：创建 AbortController，isStreaming=true
   └─ runAgentLoop(prompts, context, config, processEvents, signal, streamFn)
      ├─ emit agent_start
      ├─ emit turn_start
      ├─ for prompt of prompts:            # 用户消息先入上下文
      │    ├─ emit message_start(prompt)
      │    └─ emit message_end(prompt)
      └─ runLoop（内层循环：有工具调用或 pending 消息就继续）
         ├─ (非首轮) emit turn_start
         ├─ 注入 pendingMessages（steering）：各发 message_start/end
         ├─ streamAssistantResponse：      # 核心 LLM 调用
         │    1. transformContext(AgentMessage[] → AgentMessage[])
         │    2. convertToLlm(AgentMessage[] → Message[])
         │    3. getApiKey(provider) 动态解析
         │    4. streamFunction(model, llmContext, {…, apiKey, signal})
         │    5. 遍历 AssistantMessageEventStream：
         │       - start      → partialMessage 入 context，emit message_start
         │       - text/thinking/toolcall 的 *_start/*_delta/*_end
         │                     → 更新 partial，emit message_update（转发底层事件）
         │       - done/error → response.result() 得最终消息，
         │                      更新 context，emit message_end，返回
         ├─ 提取 toolCalls
         ├─ stopReason=="length" → failToolCallsFromTruncatedMessage：
         │    每个调用 emit tool_execution_start → tool_execution_end(isError) → 工具结果消息
         │    （防截断参数被误执行，全部标记为错误让模型重发）
         ├─ executeToolCalls（顺序或并行，见下）
         ├─ emit turn_end(message, toolResults)
         ├─ prepareNextTurn?(ctx) → 可替换 context/model/thinking 影响下一轮
         ├─ shouldStopAfterTurn?(ctx) → true 则 emit agent_end 退出
         └─ 取 steering 队列 → 有则回内层循环
         （外层循环：无 steering 后查 followUp 队列，有则继续，无则退出）
      └─ emit agent_end(messages)
   └─ finishRun：isStreaming=false，pendingToolCalls 清空
```

### 工具执行的事件细节

**顺序模式**（`toolExecution: "sequential"` 或存在 `executionMode: "sequential"` 的工具）：

```
对每个 toolCall：
  emit tool_execution_start
  prepareToolCall：
     查找工具 → prepareArguments → validateToolArguments
     beforeToolCall? → {block:true} 则产生错误结果（可带 terminate）
     signal aborted → 错误结果
  executePreparedToolCall：
     工具 execute(id, args, signal, onUpdate)
     onUpdate 异步排队 → 逐条 emit tool_execution_update
     抛异常 → 错误结果
  finalizeExecutedToolCall：
     afterToolCall? 字段级覆盖 result/isError/terminate
  emit tool_execution_end
  emit message_start(toolResult) / message_end(toolResult)
```

**并行模式**（默认）：先顺序预检全部（emit 各自的 `tool_execution_start`，被 `beforeToolCall` 拦下的立即 `tool_execution_end`），然后允许的工具并发执行；`tool_execution_end` 按完成顺序发射，工具结果消息按 assistant 原始顺序补发。批量 `terminate` 规则：所有已定稿工具结果 `terminate===true` 才提前结束。

### 错误/中止路径

- LLM 流以 `error` 事件结束 → 消息带 `stopReason: "error"/"aborted"`，循环直接 `turn_end` + `agent_end` 退出。
- 循环内抛异常 → `Agent.handleRunFailure` 合成一条失败 assistant 消息（`stopReason` 视 aborted 而定），依次 emit `message_start` → `message_end` → `turn_end` → `agent_end`，保证订阅方总能看到完整 run 序列。
- `Agent.abort()` → abort controller → 贯穿传给 streamFunction 的 signal 与各钩子。

## 4. Agent 类：订阅与状态归约（`processEvents`）

`subscribe(listener)`：监听器收到 `(event, signal)`；按订阅顺序**等待**（async），`agent_end` 的监听器全部 settle 后 run 才算 idle（`waitForIdle()`）。

`processEvents` 在分发前先把事件归约进 `AgentState`：

| 事件 | 状态变更 |
|---|---|
| `message_start` / `message_update` | `streamingMessage = message` |
| `message_end` | `streamingMessage = undefined`；push 进 `messages` |
| `tool_execution_start` | `pendingToolCalls.add(toolCallId)` |
| `tool_execution_end` | `pendingToolCalls.delete(toolCallId)` |
| `turn_end`（assistant 且有 errorMessage） | `errorMessage` 记录 |
| `agent_end` | `streamingMessage = undefined` |

## 5. EventStream（pi-ai 的 `utils/event-stream.ts`）

`EventStream<Event, TResult>` 一物两用：既是 `AsyncIterable<Event>`（for await 消费流式事件），又有 `result()`（终结值）。agent-loop 用它包装：

```ts
createAgentStream(): new EventStream<AgentEvent, AgentMessage[]>(
  (e) => e.type === "agent_end",   // 终止条件
  (e) => e.type === "agent_end" ? e.messages : [],  // 终结值
)
```

所以 `Agent` 内部通过 `emit` 回调直推，而暴露给底层调用方的 `agentLoop()` / `agentLoopContinue()`（供扩展直接使用）返回该 EventStream。

## 6. proxy.ts：事件跨进程转发

`streamProxy` 作为 `streamFn` 使用，把 LLM 调用经 SSE 转发给代理服务端（服务端管认证）：

- 请求：`POST {proxyUrl}/api/stream`，`Authorization: Bearer`，body = `{model, context, options}`（options 只序列化可序列化字段）。
- 线上协议：`data: ` 前缀的 `ProxyAssistantMessageEvent` —— 与完整事件相比**剥离了 `partial` 字段**省带宽（只留 contentIndex/delta/id/toolName 等增量信息）。
- 客户端 `processProxyEvent` 逐事件重建 `partial` 消息（text/thinking 累加 delta、toolcall 用 `parseStreamingJson` 增量解析参数），再还原为标准 `AssistantMessageEvent` 推给上游。
- abort：本地 signal 取消 reader，流以 `error` 事件（reason 视 aborted）收尾。
- `done`/`error` 事件携带 usage，客户端据此填充最终消息。

## 7. harness 层（独立运行时，`harness/events.ts`）

`AgentHarness` 是另一套面向持久化/恢复的运行时，其事件与上面完全无关：

- `HarnessEvent = run_start(lane, runId) | run_end(lane, runId, outcome: completed|aborted|failed, leafId)`。
- `HarnessEventBus`：`on(type, listener)` 订阅未来事件（不重放、无缓冲）；`emit(event)` 同步派发给类型监听与 watch 监听；`watch(captureSnapshot)` 返回 `{snapshot, start(listener), unsubscribe}` —— start 前事件进缓冲、start 后直送（期间重入仍保序）。

## 8. 下游消费（简述）

`pi-coding-agent` 的 `AgentSession` 通过 `Agent.subscribe` 订阅 `AgentEvent`：把 `message_start/end` 转录为会话历史并持久化，`message_update` 驱动 TUI 增量渲染，`turn_end`/`tool_execution_*` 更新 footer/状态栏与工具执行面板；run 结束后按 `messages` 落盘。远程场景（`pi-protocol`）则把会话快照/增量映射为 `TranscriptProgress`（`item_started`/`assistant_delta`/`item_finished`）。
