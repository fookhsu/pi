# pi-server — 会话服务器

`packages/server`，包名 `@earendil-works/pi-server`。**实验性**：API 不稳定，可能随时变更或移除。提供 `PiServer` 会话服务器核心 + 传输监听器。

## 架构

```
PiServer
 ├── listeners: PiServerListener[]     # 传输监听器（如 Unix socket）
 ├── connections: Set<ConnectionState> # 活动连接
 ├── LiveSessionManager                # 执行命令、管理会话附着
 └── ServerSnapshotPublisher           # 生成并广播权威快照
```

- 应用提供 `PiServerService` 实现（`listSessions` / `listModels` / `createSession` / `openSession`）。本包不提供 CLI 或 coding-agent 服务。
- `PiServer` 通过 `PiServerListener` 接口组合传输监听器；每个监听器必须在把连接交给 `PiServer` 前完成传输级认证/授权（如 WebSocket 在 HTTP upgrade 时校验凭据，Unix 监听器依赖 socket 文件系统权限）。
- Unix 子模块（`transports/unix/`）导出 `createUnixListener()` 构建块与 `createUnixServer()` 预设：长度前缀 CBOR 消息，常见场景一行配置。

## 连接生命周期（`server.ts`）

1. 监听器 accept → `accept(connection)`：创建 `ConnectionState`（随机 id、`ClientMessageDecoder`、handshake 定时器，默认 5s）。
2. **awaitingHello**：首条消息必须是 `hello`，否则 `failProtocol`（发 `hello_error` 后关闭）。
3. **handshaking**：校验协议版本（`isSupportedProtocolVersion`），生成服务器快照，发送 `hello`（含 `connectionId` + `snapshot`），随后进入 **ready**；若生成期间快照 revision 已变，补发最新 `server_snapshot` 事件。
4. **ready**：`handleRequest` 把 request 信封交给 `LiveSessionManager.executeCommand`，结果以 response 信封返回；命令抛错映射为 `ProtocolError` 响应。
5. 关闭/断线：`transportClosed` 调 `decoder.end()`（检测截断）后 `disconnect`；服务端 `close()` 逐个关闭连接并清理会话管理器。

错误映射（`toProtocolError`）：`InternalServerError` → `internal_error`（隐藏内部原因）；`PiServerError` → 对应 code（`not_implemented` 用固定文案）；`ProtocolValidationError` → `invalid_request`；其余 → `internal_error`。

## 会话管理（`sessions.ts`）

`LiveSessionManager` 职责：

- 执行命令（list/create/attach/detach/prompt/steer/abort/set_model/set_thinking），命令结果即权威快照。
- 每个连接维护其附着的 sessionIds；detach/断线时释放附着。
- `prompt`/`steer` 会真正驱动后端的 agent 运行时（通过 service 提供的会话句柄），并把增量 progress 与最终快照发回相关连接。
- `session_locked` / `busy` 等并发保护。

## 快照发布（`snapshots.ts`）

- `ServerSnapshotPublisher.get()`：汇总 `listSessions()` 的元数据 + `listModels()` 的模型列表 + 当前 revision，生成 `ServerSnapshot`。
- `broadcast()`：revision +1 后向所有 ready 连接广播 `server_snapshot` 事件；服务端还从活动会话的实时状态补充 `SessionMetadata`（无需存储伪造 phase/model/thinking/attachment/lock 等运行时字段）。
- 连接断开后也触发广播（其余客户端看到元数据变化）。

## 测试辅助（`testing/`）

`testing/` 提供内存 `TestService`、内存客户端与测试服务器预设，用于协议级集成测试（`server.ts` 的测试即用此）。
