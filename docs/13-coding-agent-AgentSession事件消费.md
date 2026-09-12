# coding-agent 的 AgentSession 如何消费 Agent 事件

`packages/coding-agent/src/core/agent-session.ts`（约 3500 行）是 `pi-agent-core` `Agent` 事件与上层（TUI、RPC、扩展、持久化）之间的**总枢纽**。它订阅 `Agent.subscribe`，把 `AgentEvent` 转译为扩展事件与 `AgentSessionEvent`，并顺带完成持久化、重试、compaction 等横切逻辑。

核心文件：`core/agent-session.ts`、`core/event-bus.ts`（扩展间通道）、`core/session-manager.ts`（持久化）、`modes/interactive/interactive-mode.ts` / `modes/rpc/rpc-mode.ts`（UI/RPC 消费）。

## 1. 事件转译：AgentEvent → AgentSessionEvent

`AgentSessionEvent`（`agent-session.ts` 顶部定义）是 `AgentEvent` 的**超集**：

- 原样透传 `AgentEvent` 中的大部分事件（唯一改动：`agent_end` 增加 `willRetry: boolean` 字段）。
- 新增会话级事件：
  - `agent_settled`（run 完全落定，含扩展与监听器）
  - `queue_update`（steering/followUp 队列快照，供 UI 显示）
  - `compaction_start / compaction_end`（reason: manual|threshold|overflow，result/aborted/willRetry）
  - `auto_retry_start / auto_retry_end`（attempt/maxAttempts/delayMs）
  - `summarization_retry_*`（compaction 摘要与分支摘要的重试进度）
  - `entry_appended`、`session_info_changed`、`thinking_level_changed`
  - `bash_execution_update`（bash 流式输出增量）
- 消费方：`session.subscribe(listener)`（TUI 在 `interactive-mode.ts`，RPC 在 `rpc-mode.ts`）。

## 2. 事件处理主入口 `_handleAgentEvent`（第 623 行）

`AgentSession` 构造时 `this.agent.subscribe(this._handleAgentEvent)`（**内部必订阅**），每个事件按固定顺序走四步：

```
1. 队列对账（仅 message_start + user）
2. 转发射扩展（_emitExtensionEvent）
3. 广播给 session 监听器（_emit）
4. 横切逻辑（持久化 / 重试计数 / flush 待定消息）
```

### 2.1 队列对账

`message_start` 且角色为 user 时，若文本在 `_steeringMessages` / `_followUpMessages` 里，先移除并发 `queue_update`——保证 UI 在消息渲染前看到队列已清空。

### 2.2 扩展转发（`_emitExtensionEvent`）

`AgentEvent` 逐类型映射为扩展事件，交给 `ExtensionRunner.emit`：

| AgentEvent | 扩展事件 | 备注 |
|---|---|---|
| `agent_start` | `agent_start` | 重置 `_turnIndex = 0` |
| `agent_end` | `agent_end(messages)` | |
| `turn_start` / `turn_end` | 同构 + `turnIndex` | `turn_end` 后 `_turnIndex++` |
| `message_start/update/end` | 同构 | `message_end` 走 `emitMessageEnd`，扩展可返回**替换消息**（经 `_replaceMessageInPlace` 原地替换，保证 agent 状态/后续事件/持久化一致） |
| `tool_execution_start/update/end` | 同构 | |

此外还有不依赖 AgentEvent 的扩展钩子：`beforeToolCall`/`afterToolCall`（`_installAgentToolHooks`，在**执行前/后**拦截，而非事件），`input` 拦截（prompt 前）、`before_agent_start`（注入自定义消息与 systemPrompt 覆盖）。

### 2.3 监听器广播

`agent_end` 广播前调用 `_willRetryAfterAgentEnd` 判断是否要自动重试，把结果塞进事件的 `willRetry` 字段。监听器是同步 for 循环调用（不 await）。

### 2.4 横切逻辑

- **持久化**：`message_end` 时按角色分派——
  - `custom` → `sessionManager.appendCustomMessageEntry`
  - `user|assistant|toolResult` → `sessionManager.appendMessage`（写 JSONL 转录）
  - `bashExecution/compactionSummary/branchSummary` 由各自流程另行写入。
- **assistant 消息记账**：记录 `_lastAssistantMessage`（供 run 结束后 compaction/retry 判定）；成功响应（stopReason 非 error）时重置重试计数（发 `auto_retry_end` success）。
- **flush 时机**：`turn_end` 是"assistant + 全部工具结果都已入上下文"的**最早安全点**，`_flushPendingCustomMessages` 在这里把 run 期间排队、不能插在工具调用与结果之间的 context-only 自定义消息落进会话。

## 3. run 生命周期：_runAgentPrompt 与 _handlePostAgentRun

```
prompt(text)
  ├─ 扩展命令 / input 拦截 / skill/模板展开
  ├─ streaming 中 → steer()/followUp() 入队列（发 queue_update）
  ├─ 空闲 → 模型/认证预检 → 前一轮后 compaction 检查 → 组装 messages
  └─ _runAgentPrompt(messages)
       ├─ _isAgentRunActive = true
       ├─ await agent.prompt(messages)
       ├─ while (_handlePostAgentRun()) await agent.continue()
       └─ finally: 清理覆盖/排队 → _emitAgentSettled()
```

`_handlePostAgentRun`（agent_end 之后）按优先级链决定是否续跑：

```
1. _lastAssistantMessage 可重试 → _prepareRetry（指数退避，_retryAttempt++）
2. 重试用尽 → 发 auto_retry_end 失败
3. _checkCompaction(最后 assistant 消息) → 需要则压缩
4. agent.hasQueuedMessages()（agent_end 扩展处理器排的队）→ continue
```

注意 `agent.prompt()` 内部自带一轮 `_handlePostAgentRun` 循环；`continue()` 返回后同样再检查。

### 自动重试（_prepareRetry）

- 只对**可重试错误**生效（过载/限流/服务端错误；`isContextOverflow` 交给 compaction 而非重试）。
- 从 agent 状态移除失败消息（保留在会话历史），指数退避 `baseDelayMs * 2^(attempt-1)`，可被 `abortRetry()` 取消。
- 事件：`auto_retry_start` / `auto_retry_end`。

### 自动 compaction（_checkCompaction / _runAutoCompaction）

在 `agent_end` 后对最后一条 assistant 消息判定，三种自动触发：

| 场景 | 条件 | 行为 |
|---|---|---|
| overflow + retry | `isContextOverflow` 且 stopReason != stop | 移除失败消息 → 压缩 → `continue()` 重试一次（`_overflowRecoveryAttempted` 防死循环） |
| overflow 无 retry | 响应已完整但超窗 | 压缩，不重试 |
| threshold | `shouldCompact(tokens, contextWindow, settings)` | 压缩，不重试（error/零 usage 消息用估算） |

细节：跳过 aborted 消息；不同模型的消息不触发（避免旧模型 overflow 误伤新模型）；compaction 边界前的旧消息不重复触发。`_runAutoCompaction` 先 `prepareCompaction` + `session_before_compact` 扩展钩子（可取消或自供摘要），默认走共享的 `compact()`（带 summarization 重试回调），成功即 `appendCompaction` + 重建 `agent.state.messages`（`sessionManager.buildSessionContext()`），最后发 `compaction_end`（willRetry 决定是否 `continue()`）。

## 4. 持久化与状态重建（SessionManager）

- `appendMessage`（message_end 时调用）写入会话 JSONL；`appendCompaction` / `appendBranchSummary` / `appendCustomMessageEntry` 分别写各自条目类型。
- `buildSessionContext()`：从条目重建 `AgentMessage[]`（compaction 摘要前置、分支导航），供 `agent.state.messages` 刷新与恢复。
- 重试/overflow 时"从 agent 状态移除失败消息但保留在历史"的不对称，靠 `appendMessage` 先落盘、`_checkCompaction` 再删内存来实现；压缩后重建上下文时若又带回了失败 assistant 消息，会再次剔除（因为 `agent.continue()` 拒绝以 assistant 结尾的上下文）。

## 5. UI 消费：interactive-mode（TUI）

`subscribeToAgent()` → `handleEvent`（约 3350 行处）按事件类型驱动渲染：

| 事件 | TUI 行为 |
|---|---|
| `agent_start` | 清 pendingTools、terminal progress、工作状态指示器 |
| `message_start` (user) | 消息入聊天区 + 队列显示更新 |
| `message_start` (assistant) | 新建 `AssistantMessageComponent`（流式）挂到 chatContainer |
| `message_update` (assistant) | `updateContent` 增量渲染；内容里的 `toolCall` 块创建/更新 `ToolExecutionComponent`（参数流式显示） |
| `message_end` (assistant) | 终态渲染；aborted/error → 所有 pending 工具组件标记错误；否则触发 edit 工具 diff 计算、缓存未命中提示 |
| `tool_execution_start/update/end` | 工具执行面板：创建/更新参数/写入结果与错误态 |
| `agent_end` | 关闭 progress/状态指示器、清理流式组件与 pending 工具 |
| `agent_settled` | 检查关停请求（session 彻底空闲） |
| `queue_update` | 更新待发送消息数（footer/editor） |
| `compaction_start/end` | 压缩状态指示器、ESC 变为 abortCompaction；结束后清空聊天区按新上下文重渲染 + 插入压缩摘要消息 |
| `auto_retry_start/end` | RetryStatusIndicator、ESC 变为 abortRetry |
| `summarization_retry_*` | 摘要/压缩重试指示器 |

## 6. RPC 模式消费

`rpc-mode.ts`：

- `session.subscribe` → 把 `AgentSessionEvent` 序列化为 JSONL 响应/事件流发给外部进程（`type:"response"` 携带状态，事件独立行输出）。
- 另 `session.agent.subscribe`（底层）做背压协调（`unsubscribeBackpressure`），确保 stdout 写入排队时不被撑爆。

## 7. 关键设计要点

1. **单枢纽、双出口**：`_handleAgentEvent` 是唯一入口，先扩展后监听器，扩展可改写消息（原地替换），监听器看到的是改后事件。
2. **事件 = 副作用编排点**：持久化、重试计数、flush、queue 对账都挂在事件上而不是 run 返回后，因为 run 是异步循环（steering/followUp 续跑），事件是唯一可靠的时间线。
3. **run 的边界比 agent 事件宽**：`_isAgentRunActive` 覆盖 prompt + 全部 continue（重试/overflow/队列续跑），`agent_end` 只是"循环不再发事件"，真正的空闲信号是 `agent_settled`。
4. **错误恢复分层**：可重试错误 → auto-retry；上下文溢出/截断 → compaction + 单次重试；两者都从 agent 状态移除坏消息但保留会话历史，保证可恢复且不丢记录。
5. **UI 状态机**：TUI 用 `streamingComponent`/`pendingTools` 跟随 `message_*`/`tool_*` 事件构建增量视图，事件缺失时（如 error 消息无 tool_end）在 `message_end` 统一兜底清理。
