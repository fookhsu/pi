# pi-protocol — 线协议

`packages/protocol`，包名 `@earendil-works/pi-protocol`。运行环境中立（无 Node 专属 import）的 schema、类型、CBOR 编解码与字节流分帧。依赖仅 `typebox`。

## 线布局（协议版本 1）

每个二进制消息：

1. 4 字节大端无符号长度（payload 字节数）。
2. 一段定长 CBOR item（即完整消息）。

默认 `DEFAULT_MAX_FRAME_LENGTH = 16 MiB`，可用 `maxFrameLength` 覆盖（客户端与服务端需配置一致）。

## 消息类型（`src/schemas.ts`，全部经 typebox 严格校验 `additionalProperties: false`）

### 客户端 → 服务端

- 首条必须是 `hello`：`{ type: "hello", version }`（版本是整数，不做字符串强转；不匹配时服务端回 `hello_error`）。
- 之后是 request 信封：`{ type: "request", id, request: Command }`。

命令（`Command`，按 `command` 字段判别）：

| 命令 | 载荷 |
|---|---|
| `list` | — |
| `create` | 可选 `cwd` / `name` / `model` / `thinkingLevel` |
| `attach` / `detach` | `sessionId` |
| `prompt` / `steer` | `sessionId` + `text` |
| `abort` | `sessionId` |
| `set_model` | `sessionId` + `model` |
| `set_thinking` | `sessionId` + `thinkingLevel` |

### 服务端 → 客户端

- `hello`：`{ type: "hello", version, connectionId, snapshot: ServerSnapshot }`；或 `hello_error`。
- response 信封：`{ type: "response", id, ok: true, result }` 或 `{ type: "response", id, ok: false, error: ProtocolError }`。
- event 信封：`{ type: "event", event: ServerEvent }`。

`ProtocolError`：`{ code, message, details? }`，code 为 `version | busy | session_locked | not_found | invalid_request | not_implemented | internal_error`。

### 服务器事件（`ServerEvent`）

- `server_snapshot`：服务器级快照（serverId、protocolVersion、revision、sessions 元数据列表、models 列表）。
- `session_snapshot`：会话级快照（见下）。
- `session_progress`：增量 `TranscriptProgress` —— **只是 UI 提示，不得并入权威状态**。
- `session_removed`。

## 权威状态模型

- `SessionMetadata`（列表可见，无需获取会话运行时）：仅 `id` + `createdAt` 必填；`updatedAt` / `parentSessionId` / `sessionName` / `cwd` 视后端支持。
- `SessionSnapshot`（获取会话后）：`id`、`name`、`cwd`、`createdAt/updatedAt`、`phase`（`idle | turn | compaction | branch_summary | retry`，与 AgentHarness 阶段词汇一致）、`model`（`ModelRef`）、`thinkingLevel`、`attached`、`locked`、`revision`、`transcript`（`TranscriptItem[]`）、`queuedSteer` / `queuedSteerCount`。
- `TranscriptItem`：`user | assistant | tool` 三种角色的消息条目。assistant 有 `streaming | complete | error | aborted` 状态与 `stopReason`；tool 有 `running | complete | error` 状态与 `isError`。
- `TranscriptProgress`：`item_started` / `assistant_delta`（按 messageId + contentIndex 增量）/ `item_updated` / `item_finished`。
- 内容块（content）：`text`、`thinking`（可带 `redacted`）、`image`（base64 + mimeType）、`toolCall`（含 `input: JsonValue`）。
- `Usage`：input/output/cacheRead/cacheWrite + reasoning + 成本明细。
- 模型元数据 `ModelMetadata`：provider、id、name、api、reasoning、input 类型、contextWindow、maxTokens、成本、支持的 thinking levels、`authenticated`。

## 编解码与分帧（`src/codec.ts` / `framing.ts` / `cbor/`）

- `encodeClientMessage(msg, opts)` / `encodeServerMessage(msg, opts)`：校验并返回完整分帧的 `Uint8Array`。
- 增量解码器：`createClientMessageDecoder()` / `createServerMessageDecoder()`（也可直接用 `ClientMessageDecoder` / `ServerMessageDecoder` 类）。`push(chunk)` 接受任意碎片/合并，返回完整消息数组；流结束时调 `end()` 检测截断。
- `FrameDecoder`：增量地把字节拆成长度前缀载荷（64 KiB 块缓冲，超限或截断抛 `FrameError`）。
- `cbor/`：`encoder.ts` / `decoder.ts` / `options.ts` —— 定长 CBOR 编解码。
- schema 违规、坏 CBOR、非法分帧抛 `ProtocolValidationError`；校验错误不保留被拒载荷。
