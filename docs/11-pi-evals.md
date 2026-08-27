# pi-evals — 行为评测

`packages/evals`，包名 `@earendil-works/pi-evals`。pi 工作流的行为、模型背书（model-backed）评测：把真实的 `AgentSession` 适配到 `vitest-evals`，在隔离的临时项目/agent 目录中运行，并附着原生 pi 会话产物。

## 运行

```bash
# 仓库根目录，带默认 provider + model
npm run eval -- --provider openai --model gpt-5.6-sol
# 等价环境变量：PI_PROVIDER=openai PI_MODEL=gpt-5.6-sol
# 额外参数透传给 Vitest
npm run eval -- src/extensions.eval.ts
npm run eval -- -t "creates, reloads, and uses"
```

- 认证走 pi 正常 `ModelRuntime`（含 Pi 订阅凭据与各 provider 的 API key 环境变量）。
- 每次运行打印被忽略的 `.eval/` 产物目录；`runs.jsonl` 索引已完成的 harness 运行及其 `sessions/` 下的原生 pi 会话 JSONL 附件（可能包含提示、响应、源码与工具输出）。

## 编写评测

- 按 `vitest-evals`（getsentry）的一般指导组织 suite/judge/assertion/规范化 trace。
- pi 专属：`createPiCodingAgentHarness(...)`（`src/pi-harness.ts`），每个 `describeEval(...)` suite 绑定一个 harness。
- `PiCodingAgentInput`：字符串或 `{type:"prompt", content}` / `{type:"reload"}` 序列（模拟重启/恢复）。
- 选项：`name`、`model`（provider+id）、`noTools`、`transformSystemPrompt`、`output`（从响应 + 会话自定义输出）。
- 模型选择：显式 `model` 或 `PI_PROVIDER` + `PI_MODEL`（必须同时给出；每个 harness 也可自行配置，此时允许无默认）。
- 已有 suite：`smoke.eval.ts`、`extensions.eval.ts`。

## 本地 harness 支撑（`src/vitest-evals/`）

- `harness-table.ts`：harness 结果表格。
- `artifacts.ts`：pi 会话快照产物（`PI_SESSION_SNAPSHOT_ARTIFACT`）。
- `reporter.ts` / `setup.ts` / `summary.ts`：vitest 报告/初始化/摘要。

## 用途

度量端到端行为、对比提示词/工具/技能/模型/harness 配置（例如扩展安装、会话 reload 恢复等流程）。
