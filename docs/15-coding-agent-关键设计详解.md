# coding-agent AgentSession 关键设计详解

对《13-coding-agent-AgentSession事件消费.md》中列出的关键设计逐条展开：问题、机制、关键代码（附文件与行号）、为何必要。所有行号指 `packages/coding-agent/src/`。

---

## 1. 单枢纽、双出口：`_handleAgentEvent` 是唯一入口

**问题**：Agent 事件要同时喂给 3 类消费者——扩展（可**改写**消息）、会话监听器（UI/RPC）、内部横切逻辑（持久化、重试、compaction）。若各消费者各自订阅 `agent`，顺序与改写传播无法保证：扩展改了消息，UI 却看到旧版；持久化先落盘、重试后删，时序漂移。

**机制**（`core/agent-session.ts` 402 行订阅 + 623-703 行处理器）：

```ts
// 402 行：内部必订阅，会话的一切事件都先经过这里
this._unsubscribeAgent = this.agent.subscribe(this._handleAgentEvent);
```

```ts
// 623-703 行（节选）
private _handleAgentEvent = async (event: AgentEvent): Promise<void> => {
    // ① 队列对账（message_start + user 时移除对应 steering/followUp 条目）——626-646 行
    // ② 扩展优先：可返回替换消息
    await this._emitExtensionEvent(event);                       // 647 行
    // ③ 广播（agent_end 附加 willRetry）
    this._emit(event.type === "agent_end"
        ? { ...event, willRetry: this._willRetryAfterAgentEnd(event) }  // 650 行
        : event);
    // ④ 横切逻辑：message_end 持久化 / 重试计数 / turn_end flush —— 656-702 行
};
```

**扩展改写消息**：`_emitExtensionEvent` 的 `message_end` 分支（749-809 行）调用 `_extensionRunner.emitMessageEnd`，若扩展返回替换消息，经 `_replaceMessageInPlace`（732-746 行）**原地替换对象属性**：

```ts
// 732 行：不换引用，原地改写——agent 状态、后续事件、监听器、持久化全部看到同一对象
const targetRecord = target as unknown as Record<string, unknown>;
for (const key of Object.keys(targetRecord)) delete targetRecord[key];
Object.assign(targetRecord, replacement);
```

**为何必要**：事件是异步循环（steering/followUp 续跑）里唯一的全局时间线；"先扩展后监听器、扩展可原地改写"保证所有下游看到的版本一致；`message_end` 后 Agent 已经把最终对象放进 state，原地替换避免了"监听器/持久化各持一份旧拷贝"的幽灵不一致。

---

## 2. 事件 = 副作用编排点

**问题**：run 不是一次同步调用，而是 `prompt + N 次 continue` 的异步循环。若持久化/重试/flush 放在 `agent.prompt()` 返回后做，会在 continue 之间漏掉中间态（例如某条 assistant 消息已经流完但 run 还在跑）。

**机制**：副作用全部挂在事件上——`message_end` 是消息"落定"的唯一时机（656-702 行）：

```ts
// 持久化分派（约 658-684 行）
if (event.message.role === "custom") {
    this.sessionManager.appendCustomMessageEntry(...);
} else if (role 是 user|assistant|toolResult) {
    this.sessionManager.appendMessage(event.message);      // 写 JSONL 转录
}
if (event.message.role === "assistant") {
    this._lastAssistantMessage = event.message;            // 供 run 结束后 compaction/retry 判定
    if (stopReason 非 error && this._retryAttempt > 0) {   // 成功即重置重试计数
        this._emit({ type: "auto_retry_end", success: true, ... });
        this._retryAttempt = 0;
    }
}
```

`turn_end` 是"assistant + 全部工具结果已入上下文"的**最早安全点**，`_flushPendingCustomMessages`（1512 行）在此把 run 期间排队、不能插在工具调用与结果之间的 context-only 自定义消息落进会话（701 行调用）。

**为何必要**：续跑循环里事件是唯一单调递增、无遗漏的观察点；把副作用绑到事件上，任何一轮续跑产生的消息都会恰好在"它该出现的时刻"被持久化或 flush，顺序天然正确。

---

## 3. run 边界比 agent 事件宽：`agent_settled` 才是空闲信号

**问题**：`agent_end` 只是"agent 循环不再发射事件"；但它之后还有重试退避、overflow 压缩、扩展在 `agent_end` 排队的新消息等**后续动作**。把 `agent_end` 当空闲会过早释放 UI 状态。

**机制**（`core/agent-session.ts` 1085-1124 行）：

```ts
private async _runAgentPrompt(messages): Promise<void> {
    this._isAgentRunActive = true;
    try {
        await this.agent.prompt(messages);
        while (await this._handlePostAgentRun()) {   // 1089 行：重试/压缩/队列 → continue
            await this.agent.continue();
        }
    } finally {
        this._systemPromptOverride = undefined;
        this._flushPendingBashMessages();
        this._flushPendingCustomMessages();
        await this._emitAgentSettled();              // 1096 行：真正的空闲信号
    }
}

private async _handlePostAgentRun(): Promise<boolean> {
    const msg = this._lastAssistantMessage;          // 1101-1103 行
    if (!msg) return false;
    if (this._isRetryableError(msg) && (await this._prepareRetry(msg))) return true;  // 1107 行
    if (msg.stopReason === "error" && this._retryAttempt > 0) { /* 发 auto_retry_end 失败 */ }
    if (await this._checkCompaction(msg)) return true;           // 1121 行
    return this.agent.hasQueuedMessages();                       // 1124 行：agent_end 扩展排的队
}
```

`_emitAgentSettled`（609-619 行）：置 `_isAgentRunActive = false`，先通知扩展 `agent_settled` 再广播监听器，最后 `_resolveIdleWaitIfIdle()`。

**为何必要**：UI 在 `agent_settled` 才检查关停请求、RPC 才知道可以处理下一批命令；把"循环结束"与"会话空闲"分开，重试/压缩期间输入框保持 active、状态指示器不被错误清掉。

---

## 4. 错误恢复分层：auto-retry 与 compaction 各管一摊

**问题**：LLM 调用的失败分两类——**瞬时错误**（过载/限流/5xx，重试即可）与**上下文问题**（溢出/截断，重试只会更糟，需要压缩）。混在一起处理会导致死循环或无效重试。

**机制**：

**分流判定**（2825-2830 行）：

```ts
private _isRetryableError(message: AssistantMessage): boolean {
    if (isContextOverflow(message, this.model?.contextWindow ?? 0)) return false;  // 溢出不重试
    return isRetryableAssistantError(message);   // 只认瞬时错误
}
```

**auto-retry**（`_prepareRetry`，2866-2920 行）：

```ts
this._retryAttempt++;
if (this._retryAttempt > settings.maxRetries) { this._retryAttempt--; return false; }
const delayMs = settings.baseDelayMs * 2 ** (this._retryAttempt - 1);   // 指数退避
// 从 agent 状态移除错误消息（保留在会话历史）——2879-2883 行
const messages = this.agent.state.messages;
if (messages.length > 0 && messages[messages.length - 1].role === "assistant") {
    this.agent.state.messages = messages.slice(0, -1);
}
// abortable 退避 sleep（2886-2906 行）
```

**compaction**（`_checkCompaction`，2105 行起）三种自动触发：

| 场景 | 判定 | 行为 |
|---|---|---|
| overflow + retry | `isContextOverflow` 且 `stopReason !== "stop"` | 移除失败消息 → `_runAutoCompaction("overflow", willRetry=true)` → continue 重试一次 |
| overflow 无 retry | 响应已完整但超窗 | 压缩，不重试 |
| threshold | `shouldCompact(tokens, contextWindow, settings)` | 压缩，不重试 |

关键防御（2145-2174 行附近）：
- `_overflowRecoveryAttempted` 标记：**只允许一次** compact-and-retry，二次失败直接报错终止，防死循环；
- `sameModel` 检查：换过模型的消息不触发（旧模型溢出不误伤新模型）；
- compaction 边界时间戳检查：压缩前的旧消息不重复触发。

压缩后（`_runAutoCompaction`，2221 行起）：`appendCompaction` 落盘 → `sessionManager.buildSessionContext()` 重建 `agent.state.messages` → 若 `willRetry` 且末尾仍是 error/length 消息则再移除（2363 行注释：overflow 响应在 message_end 时已落盘，重建可能把它带回内存），返回 true 让 `_handlePostAgentRun` 调 `continue()`。

**为何必要**：
- 重试只处理"再试可能成功"的错误；上下文问题重试只会再烧 token 再失败。
- "从 state 移除但保留历史"的对称性：`appendMessage` 在 `message_end` 先落盘，压缩/重试只删内存里的上下文——**转录完整、上下文干净**，恢复/导出不受影响。

---

## 5. UI 状态机：streamingComponent + pendingTools 跟随事件流

**问题**：TUI 需要把"流式增量事件"渲染成稳定的 UI 对象（消息卡、工具执行面板）。事件可能乱序到达 `tool_execution_end`（并行工具按完成序）、可能缺失终态（error/aborted 时工具没跑完）。UI 必须**事件驱动地增量更新**，并在终态统一兜底。

**机制**（`modes/interactive/interactive-mode.ts` 3175 行起，`handleEvent`）：

- **消息卡**：`message_start`(assistant) 新建 `AssistantMessageComponent` 挂到 chat（3244-3254 行）；`message_update` 调 `updateContent` 增量渲染（3255 行起）；`message_end` 终态渲染并清理（3290 行起）。
- **工具面板**（`pendingTools: Map<toolCallId, ToolExecutionComponent>`）：
  - `message_update` 时扫描 assistant 消息 content 里的 `toolCall` 块，按 id 创建/更新组件（3266-3286 行）：

```ts
for (const content of this.streamingMessage.content) {
    if (content.type === "toolCall") {
        if (!this.pendingTools.has(content.id)) {
            const component = new ToolExecutionComponent(content.name, content.id, content.arguments, ...);
            this.chatContainer.addChild(component);
            this.pendingTools.set(content.id, component);
        } else {
            this.pendingTools.get(content.id)?.updateArgs(content.arguments);
        }
    }
}
```

  - `tool_execution_start/update/end` 更新参数/流式输出/结果与错误态（3334-3375 行）；并行工具完成序到达时组件仍按 id 命中。
- **终态兜底**（3290-3332 行）：`message_end` 若 `stopReason === "aborted" | "error"`，把所有 pending 工具组件标错误并清空；否则 `setArgsComplete()` 触发 edit 工具 diff 计算。`agent_end`（3377 行）再统一清 streamingComponent 与 pendingTools。
- **进度/重试指示器**：`compaction_start/end`（3396/3410 行）、`auto_retry_start/end`（3461/3474 行）期间把 ESC 临时改绑到 `abortCompaction`/`abortRetry`，结束后恢复。

**为何必要**：
- 组件与事件解耦：工具组件由 `message_update` 的 toolCall 块创建（流式期间参数已可见），由 `tool_execution_*` 补充执行结果——两条事件路径都写同一个 map，UI 不依赖二者的到达顺序。
- 兜底清理保证异常/中止时 UI 不残留"永远转圈"的面板；`agent_end` 作为最后防线再次清场。

---

## 附：横切关注点的事件挂载点汇总

| 关注点 | 挂载事件 | 位置 |
|---|---|---|
| 会话持久化 | `message_end` | agent-session.ts 656-702 |
| 重试计数 / 成功重置 | `message_end`(assistant) + `_handlePostAgentRun` | 同上 / 1100-1124 |
| 队列 UI 同步 | `message_start`(user) 对账 + `queue_update` | 626-646 / 发起点 |
| 自定义消息 flush | `turn_end` | 701 + 1512 |
| 扩展消息改写 | `message_end`（emitMessageEnd） | 749-809 + 732 |
| run 级空闲 | `agent_settled`（finally） | 1085-1097 |
| 自动重试 | `auto_retry_start/end` | 2866-2920 |
| 自动压缩 | `compaction_start/end` + `session_compact*` 扩展 | 2105-2365 |
