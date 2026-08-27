# pi 代码库文档

本文档是对 `packages/` 下各包的源码阅读沉淀，描述每个包的职责、核心概念、关键 API 与数据流。

## 包总览

| 包 | 目录 | 职责 |
|---|---|---|
| `@earendil-works/pi-telemetry` | `packages/telemetry` | 厂商中立的可观测性契约（无依赖，最底层） |
| `@earendil-works/pi-protocol` | `packages/protocol` | pi 协议的 schema、CBOR 编解码、字节流分帧 |
| `@earendil-works/pi-client` | `packages/client` | 传输无关的远程会话客户端（协议之上的 `PiClient`） |
| `@earendil-works/pi-server` | `packages/server` | 会话服务器核心 + 传输监听器 |
| `@earendil-works/pi-ai` | `packages/ai` | 统一 LLM 接入层：provider 集合、认证解析、流式事件 |
| `@earendil-works/pi-agent-core` | `packages/agent` | 有状态 agent：工具执行、事件流、harness、会话存储抽象 |
| `@earendil-works/pi-session-backend-sqlite-node` | `packages/session-backends/sqlite-node` | agent 会话的 SQLite 持久化（node:sqlite 适配） |
| `@earendil-works/pi-tui` | `packages/tui` | 终端 UI 框架（差分渲染、组件、输入处理） |
| `@earendil-works/pi-coding-agent` | `packages/coding-agent` | 编码代理 CLI / SDK / 扩展系统 / RPC 模式 |
| `@earendil-works/pi-evals` | `packages/evals` | 基于真实 `AgentSession` 的行为评测（vitest-evals 集成） |

## 依赖图

```
telemetry  ←── protocol ←── client
    ↑            ↑
    │            └── server
pi-ai ←────────────────────┘
    ↑
agent (pi-agent-core)
    ↑                ↑
    ├── pi-tui       ├── pi-protocol / pi-client
    │                │
coding-agent ────────┘
    ↑
evals
```

- `pi-telemetry`：零依赖，被 `pi-ai` 和 `pi-agent-core` 引用。
- `pi-protocol`：只依赖 `typebox`（schema 定义与校验），无 Node 专属依赖。
- `pi-client`：只依赖 `pi-protocol`，无 Node 专属 import，可运行在任何有字节传输的环境。
- `pi-server`：依赖 `pi-protocol` 与 `pi-ai`（用于模型元数据）。
- `pi-ai`：底层依赖各厂商 SDK（anthropic / openai / google / aws bedrock 等），上层无 Node 专属依赖。
- `pi-agent-core`：依赖 `pi-ai` 与 `pi-telemetry`；`node:sqlite` 适配被拆分到 session-backends 包。
- `pi-coding-agent`：聚合 `pi-agent-core`、`pi-ai`、`pi-tui`、`pi-client`、`pi-protocol`，是唯一面向用户的产物（CLI）。
- `pi-evals`：无运行时依赖，依赖 `vitest-evals` 与 `pi-coding-agent`。

## 版本与发布

所有包共享同一版本号（lockstep 发布，当前 0.84.3），见根目录 `AGENTS.md` 的 Releasing 一节。

## 文档索引

1. [架构总览](01-架构总览.md) — 分层、两大体系（本地 agent / 远程协议）
2. [pi-ai](02-pi-ai.md) — Provider / Models / Auth / API 流式层
3. [pi-agent-core](03-pi-agent-core.md) — Agent 循环、工具、harness、会话存储
4. [pi-protocol](04-pi-protocol.md) — 线协议、消息、分帧、CBOR
5. [pi-client](05-pi-client.md) — PiClient、SessionLease、状态机
6. [pi-server](06-pi-server.md) — PiServer、监听器、快照发布
7. [pi-tui](07-pi-tui.md) — 渲染器、组件、输入、终端能力
8. [pi-telemetry](08-pi-telemetry.md) — Span 契约与 schema
9. [pi-session-backend-sqlite-node](09-pi-session-backend-sqlite-node.md) — SQLite 持久化
10. [pi-coding-agent](10-pi-coding-agent.md) — CLI/SDK、AgentSession、扩展、模式
11. [pi-evals](11-pi-evals.md) — 行为评测
