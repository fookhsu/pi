# pi-tui — 终端 UI 框架

`packages/tui`，包名 `@earendil-works/pi-tui`。极简终端 UI 框架：差分渲染 + 同步输出，提供无闪烁的交互式 CLI 应用。

## 特性

- **可替换渲染器**：共享 `TUI` 接口，两种实现——主屏（`TuiMainScreen`，全屏）与备屏（`TuiAltScreen`，viewport + 滚动）。
- **差分渲染**：只更新变化的行或 viewport 行。
- **应用自有滚动**：备屏 viewport 支持鼠标、触控板与键盘导航。
- **同步输出**：CSI 2026 原子屏幕更新（无闪烁）。
- **Bracketed Paste**：正确处理大粘贴（>10 行有标记）。
- **组件模型**：`Component` 接口 + `render()`；主题通过组件接受 theme 接口。
- **内联图片**：支持 Kitty 与 iTerm2 图形协议。
- **自动补全**：文件路径与斜杠命令。

## 核心接口

```ts
interface TUI {
  addChild(component: Component): void;
  // 聚焦、渲染循环、输入分发、overlay 等
}
interface Component {
  render(): void;
  onKey?(key: Key): void;
  // ...
}
```

- 渲染循环：任何状态变化触发差分刷新（只重绘变化区域），配 CSI 2026 同步输出。
- 组件（`src/components/`）：`Text`、`TruncatedText`、`Input`、`Editor`（全功能编辑器，主题化、undo、kill-ring、LaTeX 渲染、word navigation）、`Markdown`（marked 渲染）、`Loader` / `CancellableLoader`、`SelectList`、`SettingsList`、`Spacer`、`Image`、`Box`、`Container`、`VStack` / `HStack`、`ScrollView`、`AltScreenFlash`。

## 输入处理（`src/keys.ts` / `keybindings.ts`）

- `Key` 归一化：解析转义序列（含 Kitty keyboard protocol：`decodeKittyPrintable`、`isKittyProtocolActive`）、`KeyId`、`matchesKey` / `parseKey` 便捷函数。
- `KeybindingsManager`：键位定义、冲突检测、`TUI_KEYBINDINGS` 默认表、`getKeybindings/setKeybindings`。
- `StdinBuffer`：stdin 批拆分缓冲。

## 终端能力（`src/terminal.ts` 等）

- `Terminal` 接口 + `ProcessTerminal` 实现（原始模式、尺寸、CSI/OSC 序列）。
- `terminal-colors.ts`：解析 OSC 11 背景色与终端配色方案报告（`parseOsc11BackgroundColor`、`parseTerminalColorSchemeReport`）。
- `terminal-image.ts`：Kitty/iTerm2 图片协议检测与编码（`detectCapabilities`、`getCapabilities`、`encodeKitty`、`encodeITerm2`、`allocateImageId`、`deleteKittyImage`…）。
- `native-modules.ts` / `native-module-path.ts`：原生模块路径解析（用于终端能力检测）。
- `fuzzy.ts`：模糊匹配（`fuzzyFilter` / `fuzzyMatch`）。
- `latex.ts`：`renderLatex`（把 LaTeX 渲染为终端文本/Unicode）。
- `layout.ts` / `layout-node.ts`：布局树（stack 布局、尺寸计算）。
- `alt-screen-search.ts`：备屏搜索。
- `editor-component.ts`：`EditorComponent` 接口（供自定义编辑器实现）。

## 用法

```ts
import { ProcessTerminal, TuiMainScreen, Text } from "@earendil-works/pi-tui";

const tui = new TuiMainScreen(new ProcessTerminal());
tui.addChild(new Text("Hello"));
// 进入渲染循环；按键经 onKey 分发到组件
```
