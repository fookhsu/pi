# agent 包关键设计详解

对《12-agent-事件流分析.md》中列出的关键设计逐条展开：问题、机制、关键代码（附文件与行号）、为何必要。

---

## 1. 消息契约：失败必须编码为 done/error 事件，而非抛异常

**问题**：agent 循环与 UI 订阅方需要一个**可预测的终止序列**。如果 provider 层在流中间 throw，循环拿不到完整消息，UI 无法渲染最终状态，持久化也无法落一条完整记录。

**契约**（`packages/ai/src/types.ts` 约 528-556 行，`AssistantMessageEvent` 注释）：

> Streams should emit `start` before partial updates, then terminate with either:
> - `done` carrying the final successful AssistantMessage, or
> - `error` carrying the final AssistantMessage with stopReason "error" or "aborted" and errorMessage.

**agent-loop 侧的强约束**（`packages/agent/src/types.ts`，`StreamFn` 注释）：

> Contract: Must not throw or return a rejected promise for request/model/runtime failures. Failures must be encoded in the returned stream via protocol events and a final AssistantMessage with stopReason "error" or "aborted".

**消费端**（`packages/agent/src/agent-loop.ts` 281-379 行，`streamAssistantResponse`）——`done`/`error` 用同一路径终结：

```ts
case "done":
case "error": {
    const finalMessage = await response.result();   // 348 行
    if (addedPartial) {
        context.messages[context.messages.length - 1] = finalMessage;
    } else {
        context.messages.push(finalMessage);
    }
    if (!addedPartial) {
        await emit({ type: "message_start", message: { ...finalMessage } });   // 未发过 start 也要补
    }
    await emit({ type: "message_end", message: finalMessage });                 // 363 行
    return finalMessage;
}
```

**闭环**：`Agent` 的兜底（`packages/agent/src/agent.ts` 约 430-455 行，`handleRunFailure`）——若循环本身抛了异常（契约被破坏或内部 bug），合成一条 `stopReason: aborted|error` 的失败消息，按 `message_start → message_end → turn_end → agent_end` 完整补发，订阅方永远能等到 `agent_end`。

**为何必要**：订阅方（TUI、持久化、RPC）都依赖"事件序列完整"做状态归约；混入 throw 会让 UI 卡在 `isStreaming`，让转录缺尾。把失败降级为"一条特殊消息"比传播异常便宜且可靠。

---

## 2. 截断保护：stopReason == "length" 时全部工具调用直接失败

**问题**：流式工具调用参数是**增量 JSON**。输出被 token 上限截断时，`toolcall_end` 前的参数可能被"尽力拯救"成一段能解析、能通过 schema 校验、但**语义不完整**的 JSON——比如 `{"path": "/tmp/foo"` 缺了 `}` 却被 salvage 补全。执行这种调用是危险的（可能截断文件路径、写错位置）。

**机制**（`packages/agent/src/agent-loop.ts` 100-108 行 + 381-409 行，`failToolCallsFromTruncatedMessage`）：

```ts
const executedToolBatch =
    message.stopReason === "length"
        ? await failToolCallsFromTruncatedMessage(toolCalls, emit)   // 101 行
        : await executeToolCalls(...);
```

```ts
for (const toolCall of toolCalls) {
    await emit({ type: "tool_execution_start", ... });
    const finalized = {
        toolCall,
        result: createErrorToolResult(
            `Tool call "${toolCall.name}" was not executed: the response hit the output token limit, ` +
            `so its arguments may be truncated. Re-issue the tool call with complete arguments.`,
        ),
        isError: true,
    };
    await emitToolExecutionEnd(finalized, emit);       // 404 行
    ...messages.push(toolResultMessage);
}
```

**效果**：每个截断调用都以**错误工具结果**回喂模型，模型看到错误后会用完整参数重新发出工具调用。不执行任何可疑参数。

**为何必要**：静默执行截断参数会造成破坏性副作用且难以追踪；全量失败 + 错误结果让模型自愈，成本只是多一轮 LLM 调用。

---

## 3. 并行工具执行：预检顺序化 + 执行并发 + 双序事件

**问题**：一次 assistant 消息可能带多个工具调用。串行太慢；直接全并发又会让 `beforeToolCall` 拦截/参数校验的语义混乱，且 UI 需要可预期的"开始"顺序。

**机制**（`packages/agent/src/agent-loop.ts` 411-580 行）：

- **选择执行模式**（411-431 行）：`config.toolExecution === "sequential"` 或存在 `executionMode: "sequential"` 的工具 → 全串行；否则并行。
- **并行**（`executeToolCallsParallel`，489 行起）三阶段：
  1. **预检循环**：按 source 序逐一对每个 toolCall 做 `prepareToolCall`（查找工具 → `prepareArguments` → `validateToolArguments` → `beforeToolCall` 拦截），**顺序** emit `tool_execution_start`；被拦下的（`kind: "immediate"`）立即 emit `tool_execution_end`。中途 `signal.aborted` 即 break。
  2. **并发执行**：`finalizedCalls` 里收集"函数形态"的条目，`Promise.all` 同时执行 `executePreparedToolCall`；每个完成时按其**完成顺序** emit `tool_execution_end`（见 500-560 行）。
  3. **结果收尾**：`orderedFinalizedCalls`（仍按 source 序）逐个 `createToolResultMessage` + emit `message_start/end`。
- **terminate 规则**（582-585 行，`shouldTerminateToolBatch`）：

```ts
function shouldTerminateToolBatch(finalizedCalls: FinalizedToolCallOutcome[]): boolean {
    return finalizedCalls.length > 0 && finalizedCalls.every((finalized) => finalized.result.terminate === true);
}
```

`beforeToolCall` 的 `block + terminate`、`afterToolCall` 的 `terminate` 都只投一票；**整批全部为 true** 才提前结束循环（避免单个工具单方面终止整批）。

**为何必要**：
- 预检顺序化保证 `beforeToolCall` 副作用（如扩展拦截）与参数校验有确定顺序；
- 执行并发把工具 I/O 时间重叠掉；
- "完成序发 end、source 序发消息"让 UI 既能看到真实完成进度，上下文/转录又保持 assistant 工具调用顺序，避免 LLM 重放时顺序错乱。

---

## 4. proxy.ts：剥 partial 省带宽 + 客户端重建

**问题**：agent 经代理服务端跑 LLM 时，完整流事件里每个 `*_delta` 都携带**整份 partial 消息**（重复拷贝全部 content 数组）。带宽浪费，且代理端不需要这份数据。

**机制**（`packages/agent/src/proxy.ts`）：

- **线上事件**（15-51 行，`ProxyAssistantMessageEvent`）：与完整事件同构但**去掉 `partial` 字段**，只保留增量信息（`contentIndex`、`delta`、`toolCallId` 等）。
- **客户端重建**（240-360 行，`processProxyEvent`）：每个事件把增量写回本地维护的 `partial` 消息，再还原为标准 `AssistantMessageEvent`。关键片段——toolcall 参数增量解析（323-331 行）：

```ts
case "toolcall_delta": {
    const content = partial.content[proxyEvent.contentIndex];
    if (content?.type === "toolCall") {
        (content as any).partialJson += proxyEvent.delta;
        content.arguments = parseStreamingJson((content as any).partialJson) || {};
        partial.content[proxyEvent.contentIndex] = { ...content }; // Trigger reactivity
        return { type: "toolcall_delta", contentIndex: proxyEvent.contentIndex, delta: proxyEvent.delta, partial };
    }
    throw new Error("Received toolcall_delta for non-toolCall content");
}
```

- **传输**（118-230 行，`streamProxy`）：`POST {proxyUrl}/api/stream`，`Authorization: Bearer`，body 只序列化白名单字段（`ProxySerializableStreamOptions`，20-28 行）；按 `data: ` 前缀的 SSE 行切分解析。
- **abort**（145 行）：`signal` 上挂 listener 调 `reader.cancel("Request aborted by user")`；循环内也检查 `signal.aborted`，最终走 `error` 事件（reason 按 aborted 区分）收尾，维持契约第 1 条。

**为何必要**：带宽削减是次要收益；关键是**单一事实源**——partial 只在客户端累积，代理端无状态，天然支持水平扩展与多客户端（各客户端各自重建，互不共享可变状态）。

---

## 5. harness 层事件完全独立：极简 run 级总线

**问题**：`AgentHarness` 是面向持久化/恢复的另一套运行时，与 `Agent`/`agent-loop` 无依赖关系。它的事件需要"极简、可缓冲、可快照"，而不是 `AgentEvent` 那样丰富的增量语义。

**机制**（`packages/agent/src/harness/events.ts`）：

- 事件只有两种（1-9 行）：

```ts
export interface RunStartEvent { type: "run_start"; lane: string; runId: string; }
export interface RunEndEvent {
    type: "run_end"; lane: string; runId: string;
    outcome: "completed" | "aborted" | "failed"; leafId: string;
}
```

- `HarnessEventBus`（37 行起）三种订阅语义：
  - `on(type, listener)`（39-64 行）：只收**未来**事件，不重放、无缓冲、无快照；按类型分发。
  - `emit(event)`（66-74 行）：同步派发（async 结果不 await），同时喂给 watch 监听。
  - `watch(captureSnapshot)`（76-99 行）：返回 `{ snapshot, start(listener), unsubscribe }`。`start()` 之前的事件进 `buffered`；`start()` 时先回放缓冲（用"换缓冲再 flush"保持重入顺序），之后实时直送。**快照在 watch 创建时捕获**，语义上等价"订阅历史 + 当前状态"。

**为何必要**：
- 持久化恢复只需知道"哪个 lane 的哪个 run 何时开始/结束、结果如何"——两个事件足够，避免 harness 与 agent 事件语义耦合。
- `watch()` 的缓冲语义解决"订阅者在 run 开始前才挂上、错过 `run_start`"的竞态：先拿快照 + 缓冲，`start()` 时按序补发，UI 与存储都能从一致点开始。

---

## 附：两条设计主线的关系

| 维度 | Agent 事件（agent-loop/Agent） | harness 事件（HarnessEventBus） |
|---|---|---|
| 粒度 | 消息级增量（text/tool 流式） | run 级生命周期 |
| 面向 | UI 渲染、转录、扩展、重试/compaction 编排 | 持久化恢复、分支导航 |
| 传播 | 回调链 + await（订阅顺序） | 同步 emit + 可缓冲 watch |
| 状态来源 | 事件流本身（增量） | 权威快照（`watch().snapshot`）+ 事件 |

两者互补：coding-agent 用前者（Agent + AgentSession），远程/持久化场景的底层可用后者。
