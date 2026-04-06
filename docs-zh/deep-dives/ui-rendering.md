# UI 渲染引擎

## 概述

Claude Code 使用了一个**深度定制的 Ink**（终端版 React）引擎 — 并非简单封装，而是一次近乎完整的重写，包含自定义 DOM、Reconciler、双缓冲屏幕渲染、Yoga 布局、文本选择、搜索浮层以及 Perfetto 级别的帧性能指标。

## 组件层级

```
App（Provider 外壳：FpsMetrics + Stats + AppState）
  → REPL（5005 行 — 主屏幕）
    → FullscreenLayout（ScrollBox + 底部面板）
      → Messages
        → VirtualMessageList（虚拟滚动）
          → MessageRow
            → Markdown / StreamingMarkdown
      → Spinner（工具/队友状态树）
```

## 文件清单

| 文件路径 | 行数 | 职责 |
|-----------|-------|---------------|
| `src/ink/ink.tsx` | 1722 | 核心：渲染循环、双缓冲、alt-screen、选择、鼠标 |
| `src/ink/reconciler.ts` | 512 | React Reconciler 桥接、DOM diff、Yoga 节点 |
| `src/ink/renderer.ts` | 178 | Yoga 布局 → 屏幕缓冲区、alt-screen 高度裁剪 |
| `src/ink/dom.ts` | 200+ | 自定义 DOM（ink-root/ink-box/ink-text）、事件处理 |
| `src/ink/screen.ts` | — | 单元格级缓冲区（StylePool/CharPool/HyperlinkPool） |
| `src/ink/selection.ts` | — | 文本选择状态机 |
| `src/screens/REPL.tsx` | 5005 | 主 REPL 屏幕 |
| `src/components/FullscreenLayout.tsx` | 636 | 布局：ScrollBox + 面板 + 固定提示栏 + "N 条新消息"气泡 |
| `src/components/Messages.tsx` | 833 | 消息渲染与过滤 |
| `src/components/VirtualMessageList.tsx` | 1081 | 虚拟滚动、高度缓存、搜索句柄 |
| `src/components/Markdown.tsx` | 235 | Markdown 管线：marked.lexer → 格式化 → Ansi |
| `src/components/Spinner.tsx` | 561 | 带工具/队友树的动画加载指示器 |

## 双缓冲渲染

`ink.tsx:196-198`：
```
frontFrame ←→ backFrame
```

每帧流程：Yoga 布局 → 渲染到 backFrame → `writeDiffToTerminal` 仅写入变化的单元格 → 交换前后缓冲区。对象池（StylePool、CharPool、HyperlinkPool）采用分代重置机制，防止内存增长。

## 渲染调度

`ink.tsx:212-216`：Reconciler 的 `resetAfterCommit` 通过 `queueMicrotask` 调用 `scheduleRender` — 将渲染延迟到 layout effects 执行之后。这确保了 `useDeclaredCursor` 的声明不会出现一帧延迟。渲染频率受 `FRAME_INTERVAL_MS` 节流控制。

## Markdown 渲染

`Markdown.tsx:37-71`：

1. **LRU Token 缓存**（500 条目）
2. **快速路径检测**：`MD_SYNTAX_RE` 正则检查 — 纯文本直接跳过 `marked.lexer`（节省约 3ms）
3. **StreamingMarkdown** (`Markdown.tsx:186-228`)：`stablePrefixRef` 实现增量分词 — 仅对最后一个不稳定块重新词法分析，时间复杂度从 O(全文) 降至 O(增量)

## 虚拟滚动

`VirtualMessageList.tsx`：
- `useVirtualScroll` Hook 管理可见窗口
- 列数变化时高度缓存失效
- StickyTracker 跟踪用户提示位置以实现固定表头
- 命令式 `JumpHandle` 用于搜索：`warmSearchIndex`、`nextMatch`/`prevMatch`、`scanElement`

## LogoHeader 优化

`Messages.tsx:55-76`：`React.memo` + `OffscreenFreeze` 包裹 LogoV2，防止 dirty 标志级联传播。没有此优化时，在 2800+ 条消息的会话中，`seenDirtyChild` 传播会导致每帧 150K+ 次写入，CPU 占用 100%。

## React Compiler 集成

所有组件使用 `_c()` 缓存哨兵 — 这是 React 19 Compiler 的输出签名。Claude Code 可能是**终端应用中最大规模的 React Compiler 部署**。

`StreamingMarkdown` 使用 `'use no memo'` 编译指示显式退出该组件的 React Compiler 优化。

## 独特发现

1. **帧污染追踪** (`ink.tsx:159`)：`prevFrameContaminated` 标志追踪选择浮层对屏幕缓冲区的修改，在下一帧强制全量重绘。
2. **`RECENT_SCROLL_REPIN_WINDOW_MS = 3000`** (`REPL.tsx:305`)：专门为 Josh Rosen 的工作流修复 — 防止用户在滚动后开始输入时自动滚动到底部。注释中直接附有 Slack 讨论帖的链接。
3. **Alt-screen 原子更新**：帧被包裹在 BSU/ESU（Begin/End Synchronized Update）中以实现终端原子更新。
4. **CJK 输入法支持**：原生光标声明（`useDeclaredCursor`）确保 CJK 输入法编辑器的光标位置正确。
