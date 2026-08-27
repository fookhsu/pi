# pi-coding-agent — 编码代理 CLI / SDK

`packages/coding-agent`，包名 `@earendil-works/pi-coding-agent`。聚合 `pi-agent-core`（agent 循环）、`pi-ai`（模型）、`pi-tui`（界面）、`pi-protocol`/`pi-client`（远程协议）的面向用户的编码代理。提供 CLI、可嵌入 SDK、扩展系统、RPC 模式与远程会话客户端。

## 入口与模式

- `cli.ts`：Node shebang 入口，`process.title = APP_NAME`，设置环境变量（`PI_CODING_AGENT`、`AI_AGENT=pi`），配置 undici HTTP dispatcher，调 `main()`。
- `main.ts`：参数解析（`cli/args.ts`：`Args`、`Mode`、`parseArgs`、`printHelp`），初始化后按模式分发：
  - **交互模式**（`modes/interactive/interactive-mode.ts`，约 6500 行）：TUI 全功能界面，业务逻辑委托 `AgentSession`。含主题（`modes/interactive/theme/`）、组件（assistant-message、diff、bash-execution、custom-message、armin、daxnuts 等）、会话分享（`session-share.ts`）、模型搜索/目录刷新。
  - **打印模式**（`modes/print-mode.ts`）：一次性 prompt，输出到 stdout（`-p`）。
  - **RPC 模式**（`modes/rpc/rpc-mode.ts`）：headless JSON stdin/stdout 协议，供外部应用嵌入。命令/响应/事件均为 JSONL（`rpc-types.ts`、`jsonl.ts`、`rpc-client.ts`）。
  - 另有实验性子命令（`cli/experimental/commands/`：`client`/`pi`/`server`）。
- 常用 CLI 能力：`--list-models`（`cli/list-models.ts`）、`--auth`（`cli/auth-command.ts`、`auth-check.ts`）、`--config`/包管理（`package-manager-cli.ts`）、会话选择（`cli/session-picker.ts`）、首次运行设置向导（`cli/startup-ui.ts`）、项目信任（`cli/project-trust.ts`、`core/project-trust.ts`）、stdin 管道输入（`readPipedStdin`）、文件参数（`cli/file-processor.ts`）。

## SDK（`core/sdk.ts`）

`createAgentSession(options: CreateAgentSessionOptions)` 是核心入口，返回 `AgentSession`。选项包括：`cwd`、`agentDir`、`modelRuntime`（默认用 `agentDir/auth.json` + `models.json`）、`model`、`thinkingLevel`、`scopedModels`（交互模式 Ctrl+P 循环切换）、工具控制（`noTools` / `tools` 白名单 / `excludeTools` 黑名单 / `customTools`）、`resourceLoader`、`sessionManager`、`settingsManager`、`sessionStartEvent`（扩展启动元数据）。文件顶部 `setDefaultStreamFn(streamSimple)` 为旧版扩展提供 streamFn 回退。

## AgentSession（`core/agent-session.ts`，约 3500 行）

所有模式共享的核心类：封装 agent 生命周期与会话管理。

- 持有 `Agent`（`pi-agent-core`），暴露 `messages`、`model`、`thinkingLevel`、`tools` 等。
- **事件订阅 + 自动会话持久化**：订阅 agent 事件，转录写入会话目录。
- 模型/思考级别管理：`setModel` / `setThinkingLevel`（clamp 到模型能力，`clampThinkingLevel`、`getSupportedThinkingLevels`）。
- **Compaction**（`core/compaction/`）：`shouldCompact` / `prepareCompaction` / `compact`，上下文估算（`estimateTokens`、`calculateContextTokens`）、溢出检测（`isContextOverflow`）、分支摘要（`generateBranchSummary`、`collectEntriesForBranchSummary`）。
- **Bash 执行**（`core/bash-executor.ts`）：工具内 shell 执行（带操作审计：命令/文件变更跟踪）。
- 会话切换与分支（`SessionManager`）、HTML 导出（`core/export-html/`，含 vendored 前端资源）、错误恢复（`isRetryableAssistantError`、重试退避）、steering/follow-up 队列。

## ModelRuntime（`core/model-runtime.ts`，约 800 行）

把 `pi-ai` 的 `Models` 适配为编码代理的模型运行时：

- 组装内置 provider 目录（`pi-ai/providers/all`）+ 扩展自定义 provider（`provider-composer.ts`：`composeModelProvider`、`resolveConfiguredModelHeaders`、配置请求认证状态）+ 远程目录（`remote-catalog-provider.ts`）。
- 凭据：`AuthStorage`（`auth-storage.ts`，读写 `agentDir/auth.json`）、`RuntimeCredentials`、`ModelConfig`（`model-config.ts`）、文件模型存储（`models-store.ts`：`FileModelsStore`）。
- 提供认证状态、模型可用性过滤（含扩展 provider 校验 `validateExtensionProvider`）、`ModelsError` 处理与授权指引（`auth-guidance.ts` 格式化"未配置/无 key"提示）。

## SessionManager（`core/session-manager.ts`，约 1700 行）

会话持久化与发现：

- session 目录（`getDefaultSessionDir(cwd)`、`ENV_SESSION_DIR` 覆盖），`assertValidSessionId`。
- 会话文件：transcript（JSONL）、元数据、prompt 文件、分支信息。
- 支持会话列表/恢复/删除/分支（parent session）、`selectSession`、会话 CWD 管理（`session-cwd.ts`：`MissingSessionCwdError` 等）。
- `session-export.ts`：导出会话。

## SettingsManager（`core/settings-manager.ts`，约 1400 行）

全局 + 项目级设置分层（`~/.pi` 与项目 `.pi/`）：

- 设置项定义（默认值、校验、类型），支持环境变量与 settings 文件覆盖；`resolve-config-value.ts` 解析配置值（含函数值）。
- 诊断（`settings-diagnostics.ts`、`collectSettingsDiagnostics`）。
- 主题、键位、工具默认集、模型默认等运行时读取。

## 扩展系统（`core/extensions/`）

扩展是 TypeScript 模块（`extensions/index.ts`、`core/extensions/types.ts` 约 1800 行），能力：

- 订阅 agent 生命周期事件（`AgentEvent`）。
- 注册 LLM 可调用工具（`ToolDefinition`）、命令、键盘快捷键、CLI flags。
- 通过 UI 原语与用户交互：overlay、对话框、widget、working indicator（`ExtensionUIDialogOptions` 等）。
- 注册自定义 provider（`Provider`）、模型目录刷新（`RefreshModelsContext`）。
- 生命周期：`loader.ts`（加载：路径解析、内联扩展、缓存）、`runner.ts`（运行：事件分发、工具注册、UI 请求转发）、`wrapper.ts`。
- 内置扩展在 `src/extensions/index.ts`（`builtInExtensions`）；llama.cpp 集成在 `src/extensions/llama/`。
- 相关文档：`packages/coding-agent/docs/extensions.md`。

## 工具（`core/tools/`）

- `index.ts`：`createCodingTools`、`createReadOnlyTools`、内置工具工厂（`createReadTool`、`createBashTool`、`createEditTool`、`createWriteTool`、`createFindTool`、`createGrepTool`、`createLsTool`、`createPowerShellTool`）。
- 实现细节：`bash.ts`（shell 执行 + 输出积累 `output-accumulator.ts` + 截断 `truncate.ts`）、`edit.ts` / `edit-diff.ts`（diff 风格编辑，`file-mutation-queue.ts` 串行化写操作）、`read.ts` / `write.ts` / `find.ts` / `grep.ts` / `ls.ts` / `path-utils.ts`、`tool-definition-wrapper.ts`、`render-utils.ts`。
- `withFileMutationQueue`：文件变更串行化，防止并发工具竞争写同一文件。

## 其他核心模块

| 模块 | 说明 |
|---|---|
| `config.ts` | `APP_NAME`、`VERSION`、`getAgentDir`、`getPackageDir`、路径展开 |
| `core/package-manager.ts` | 扩展/包安装（npm 子进程、锁文件、信任） |
| `core/resource-loader.ts` | 资源加载（~1100 行）：上下文文件、`.pi` 资源、提示词模板 |
| `core/event-bus.ts` | 进程内事件总线 |
| `core/http-dispatcher.ts` | undici 全局 dispatcher + HTTP 代理配置（`applyHttpProxySettings`） |
| `core/keybindings.ts` / `modes/interactive/theme/` | 键位与主题（含 watcher `stopThemeWatcher`） |
| `core/messages.ts`、`prompt-templates.ts`、`system-prompt.ts` | 消息转换与系统提示词 |
| `core/output-guard.ts` | stdout 接管（RPC/打印模式防污染） |
| `core/trust-manager.ts` | 项目信任（`ProjectTrustStore`、`hasTrustRequiringProjectResources`） |
| `core/pi-manifest.ts`、`core/radius.ts` | 元数据与 radius 兼容 |
| `core/usage-totals.ts`、`cache-stats.ts`、`timings.ts` | 用量/缓存/计时统计 |
| `core/experimental.ts`、`core/provider-attribution.ts` | 实验开关与 provider 归属头（`mergeProviderAttributionHeaders`） |
| `migrations.ts` | 配置迁移 + 弃用警告（`runMigrations`、`showDeprecationWarnings`） |
| `utils/` | 路径、shell、睡眠、frontmatter、工具结果图像等 |
| `server/create-harness.ts` | 会话服务器 harness（供 `pi-server` 测试/集成） |
| `client/` | 远程会话客户端：`index.ts`、`remote-session.ts`、`transcript.ts`（把远程 `PiClient` 会话映射为本地 `AgentSession` 风格） |

## 模式细节

### 交互模式（interactive-mode.ts）

- TUI 渲染循环 + 输入（`pi-tui`），prompt 编辑器、自动补全（文件路径 + 斜杠命令）、Markdown/图像/LaTeX 渲染。
- 模型切换（Ctrl+P）、会话切换/分支 UI、compaction 进度 UI、扩展 UI（overlay/dialog/widget）、bash 执行面板。
- 主题：`modes/interactive/theme/theme.ts`（`initTheme`、`stopThemeWatcher`、`getThemeByName`）。
- 会话分享：`session-share.ts`（把会话打包/分享）。

### RPC 模式（rpc-mode.ts）

- stdin 读 JSONL 命令，stdout 写 JSONL 响应/事件；`takeOverStdout` 防止库日志污染协议。
- 命令含 prompt、steer、abort、set_model、set_thinking、session 管理、扩展 UI 请求（`extension_ui_response` 回包）。
- `rpc-client.ts`：对端客户端封装（约 600 行）。
