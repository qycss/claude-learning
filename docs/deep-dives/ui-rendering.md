# UI Rendering Engine

## Overview

Claude Code uses a **deeply customized Ink** (React-for-terminal) engine — not a wrapper, but a near-complete rewrite with custom DOM, reconciler, double-buffered screen rendering, Yoga layout, text selection, search overlay, and Perfetto-level frame metrics.

## Component Hierarchy

```
App (Provider shell: FpsMetrics + Stats + AppState)
  → REPL (5005 lines — main screen)
    → FullscreenLayout (ScrollBox + bottom panel)
      → Messages
        → VirtualMessageList (virtual scroll)
          → MessageRow
            → Markdown / StreamingMarkdown
      → Spinner (tool/teammate status tree)
```

## File Inventory

| File Path | Lines | Responsibility |
|-----------|-------|---------------|
| `src/ink/ink.tsx` | 1722 | Core: render loop, double-buffer, alt-screen, selection, mouse |
| `src/ink/reconciler.ts` | 512 | React reconciler bridge, DOM diff, Yoga nodes |
| `src/ink/renderer.ts` | 178 | Yoga layout → screen buffer, alt-screen height clamping |
| `src/ink/dom.ts` | 200+ | Custom DOM (ink-root/ink-box/ink-text), event handlers |
| `src/ink/screen.ts` | — | Cell-level buffer (StylePool/CharPool/HyperlinkPool) |
| `src/ink/selection.ts` | — | Text selection state machine |
| `src/screens/REPL.tsx` | 5005 | Main REPL screen |
| `src/components/FullscreenLayout.tsx` | 636 | Layout: ScrollBox + panels + sticky prompt + "N new" pill |
| `src/components/Messages.tsx` | 833 | Message rendering with filtering |
| `src/components/VirtualMessageList.tsx` | 1081 | Virtual scroll, height cache, search handle |
| `src/components/Markdown.tsx` | 235 | Markdown pipeline: marked.lexer → format → Ansi |
| `src/components/Spinner.tsx` | 561 | Animated spinner with tool/teammate tree |

## Double-Buffered Rendering

`ink.tsx:196-198`:
```
frontFrame ←→ backFrame
```

Each frame: Yoga layout → render to backFrame → `writeDiffToTerminal` writes only changed cells → swap. Object pools (StylePool, CharPool, HyperlinkPool) with generational reset prevent memory growth.

## Render Scheduling

`ink.tsx:212-216`: Reconciler's `resetAfterCommit` calls `scheduleRender` via `queueMicrotask` — delays rendering until after layout effects. This ensures `useDeclaredCursor` declarations don't lag one frame. Throttled by `FRAME_INTERVAL_MS`.

## Markdown Rendering

`Markdown.tsx:37-71`:

1. **LRU token cache** (500 entries)
2. **Fast-path detection**: `MD_SYNTAX_RE` regex check — pure text skips `marked.lexer` (~3ms savings)
3. **StreamingMarkdown** (`Markdown.tsx:186-228`): `stablePrefixRef` enables incremental tokenization — only re-lex the last unstable block, O(delta) not O(full text)

## Virtual Scroll

`VirtualMessageList.tsx`:
- `useVirtualScroll` hook manages visible window
- Height cache invalidated by column count changes
- StickyTracker tracks user prompt positions for sticky headers
- Imperative `JumpHandle` for search: `warmSearchIndex`, `nextMatch`/`prevMatch`, `scanElement`

## LogoHeader Optimization

`Messages.tsx:55-76`: `React.memo` + `OffscreenFreeze` wraps LogoV2 to prevent dirty flag cascade. Without this, in 2800+ message sessions, `seenDirtyChild` propagation causes 150K+ writes per frame → CPU 100%.

## React Compiler Integration

All components use `_c()` memo cache sentinel — the output signature of React 19 Compiler. Claude Code may be the **largest React Compiler deployment in a terminal application**.

`StreamingMarkdown` uses `'use no memo'` pragma to explicitly opt out of React Compiler optimization for that component.

## Unique Findings

1. **Frame contamination tracking** (`ink.tsx:159`): `prevFrameContaminated` flag tracks selection overlay modifications to the screen buffer, forcing full redraw next frame.
2. **`RECENT_SCROLL_REPIN_WINDOW_MS = 3000`** (`REPL.tsx:305`): Special fix for Josh Rosen's workflow — prevents auto-scroll-to-bottom when user starts typing after scrolling. Comment links directly to Slack discussion thread.
3. **Alt-screen atomic updates**: Frames wrapped in BSU/ESU (Begin/End Synchronized Update) for atomic terminal updates.
4. **CJK IME support**: Native cursor declaration (`useDeclaredCursor`) ensures correct cursor positioning for CJK input method editors.
