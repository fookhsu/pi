# pi-client — 远程会话客户端

`packages/client`，包名 `@earendil-works/pi-client`。传输无关的远程 pi 会话客户端：`PiClient` 通过极小的 `ByteTransport` 接口交换长度前缀 CBOR 消息。**无 Node 专属 import**，可跑在 WebSocket、Unix socket 或任何有序字节传输上。

## 核心概念

### ByteTransport（`transport.ts`）

```ts
interface ByteTransport {
  send(chunk: Uint8Array): void;   // 按调用序投递，尊重背压
  close(): void;
}
interface ByteTransportFactory {
  (handlers: ByteTransportHandlers): Promise<ByteTransport>;
}
```

- 工厂每次连接尝试都要创建全新传输，并在 resolve 前完成传输层认证（如 WebSocket 升级请求携带凭据）。
- 入站字节调 `handlers.onData(chunk)`；有序关闭调 `onClose()`；传输失败调 `onError(error)`。

### 连接与状态

- `connect()` → 握手后拿到 `ServerSnapshot`；**不自动重连**，断线后手动 `reconnect()`。
- `connectionState`：`"disconnected" | "connecting" | "connected"`，可用 `onConnectionStateChange()` 订阅。
- 一条连接可挂多个会话；请求按 id 关联。
- 服务器快照与成功响应快照是权威的；progress 事件不做乐观状态变更。
- 会话元数据读 `client.snapshot?.sessions`（缓存）；`listSessions()` 向服务端请求刷新持久元数据；运行时状态需 `acquireSession()` 后获取。

### SessionLease — 会话所有权（`session-handle.ts`）

- `acquireSession(id, { mode })` 返回独立 `SessionLease`（不可直接构造）；`attachSession()` 是 `{ mode: "shared" }` 的便捷方法；`createSession()` 返回新建会话的 exclusive 租约。
- `mode: "exclusive"`：生命周期/变更协调者。已存在任何租约时获取失败（`PiSessionOwnershipError`）。
- `mode: "shared"`：多个低级消费者共享。存在 exclusive 租约时获取失败。
- `dispose()` / `detach()` 只释放当前租约；租约开始释放后拒绝命令。最后一个租约释放后客户端才发协议 detach 请求。
  - `detach()` 失败：租约恢复 active 可重试。
  - `dispose()`（清理语义）失败：报告协议错误但放弃本地所有权；`PiClient` 会在下次获取前对失败清理做对账（reconcile）。
- 服务端移除或断线会使相关附着的所有租约失效；失效租约再 dispose 是 no-op。命令失败错误：断线 → `PiDisconnectedError`；已连接但租约在释放/已释放/失效 → `PiSessionDetachedError`。租约实现 `AsyncDisposable`。

### 订阅

- `subscribe(listener)`：权威快照（`ServerSnapshot`）。
- `onEvent(listener)`：协议事件。
- 两者都返回 unsubscribe 函数。服务端结构化错误暴露为 `PiServerError`。

## 内部实现（`client.ts`）

- `Connection`（`connection.ts`）：管理传输生命周期、握手、入站消息解析（FrameDecoder + ClientMessageDecoder）。
- `ClientState`（`state.ts`）：快照存储与订阅分发、会话快照/事件订阅。
- 请求管线：`#request(command)` 生成 `request-N` id，挂入 `pendingRequests`，编码发送；收到响应按 id 取回并校验 `result.command` 与请求一致（不一致视为协议违规，fail 连接）。
- 会话附着/分离去重：`#sessionAttachments` / `#sessionDetachments` 保证并发请求下 attach/detach 只发一次。
- 断线/关闭：拒绝所有 pending 请求、清空附着、使所有租约失效、通知状态监听。

## 限制与安全

- `PiClientOptions.maxFrameLength` 约束出入站 CBOR 载荷；客户端与服务端需配置一致。传输侧应单独限制排队出站字节并保持发送顺序。
- 把对端视为不可信：用安全传输 + 合适的访问控制，并在传输建立阶段完成认证。
