# pi-telemetry — 可观测性契约

`packages/telemetry`，包名 `@earendil-works/pi-telemetry`。厂商中立（vendor-neutral）的可观测性契约与类型化 schema 工具。**零依赖**，只有 `src/index.ts`（类型 + 工具）、`noop.ts`、`memory.ts`、`testing/`。

## 核心契约

### TelemetryContext / TelemetrySpan

```ts
interface TelemetryContext {
  startSpan<T>(options: SpanOptions, callback: (span: TelemetrySpan) => T | Promise<T>): Promise<T>;
}
interface TelemetrySpan extends TelemetryContext {
  addEvent(name: string, attributes?: SpanAttributes): void;
  setAttributes(attributes: SpanAttributes): void;
  setStatus(status: SpanStatus): void;   // {status:"ok"} | {status:"error", error?: {name,message}}
}
```

- 显式回调式：span 通过回调作用域界定生命周期，**没有**全局 current-span 状态。
- `SpanAttributes`：string/number/boolean 及数组。
- 配套 `NOOP_TELEMETRY_CONTEXT`（空实现，默认传播用）与 `InMemoryTelemetryContext`（参考实现，测试友好）。

### 适配器

应用可自行实现 `TelemetryContext` 适配 OpenTelemetry、Sentry、日志等后端。`testing/conformance.ts` 提供适配器一致性测试（adapter conformance），保证自定义实现符合契约。

## Typed Schemas

可序列化的 schema 定义 + 类型推断（`defineTelemetrySchema<const T>()` 提供字面量类型）：

- `TelemetrySpanDefinition`：`description`、`parents`（`any` / `root_or_external` / 指定 spans）、`startAttributes` / `endAttributes`（每个属性含 description、`sensitive?`、`cardinality?`、枚举 `values?` / `examples?`）、`events?`、`status`（默认 ok + `errorWhen` 说明）。
- `TelemetryEventDefinition`：事件描述 + 属性。
- `TelemetrySchemaDefinition`：`version` + spans 表。
- `sensitive` 标记让实现可以做脱敏/日志过滤；`cardinality: "low"|"high"` 提示基数（如 low = 枚举，high = 高基数）。

## 集成方式

pi 各包显式传递 telemetry context，并在各自包内定义领域 schema（例如 agent 包的 `packages/agent/docs/telemetry-schema.md`、coding-agent 的 `core/telemetry.ts`）。应用侧把真实后端适配器注入即可，无需改动库内调用点。
