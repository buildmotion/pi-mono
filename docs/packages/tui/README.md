# pi-tui — Terminal UI Framework

**Package:** `@mariozechner/pi-tui`
**Location:** `packages/tui`

## Course Introduction

This module teaches you how pi-tui works as a foundation for building terminal user interfaces in TypeScript. You will learn how a render loop, differential diffing, and keyboard event parsing combine to produce flicker-free, responsive TUIs without heavy dependencies.

After completing this module, you will be able to:

- Explain the three rendering strategies and when each is chosen.
- Build a custom `Component` that integrates into the Tui render loop.
- Configure keybindings via the `Keybindings` declaration-merging interface.
- Instrument render performance using the built-in hooks.

## Learning Goals

1. Understand differential rendering and the role of the `CURSOR_MARKER`.
2. Trace the input path from raw stdin bytes to a structured `KeyId`.
3. Use the built-in components (`editor`, `select-list`, `markdown`, `image`).
4. Write a custom component from scratch and register it.

## Prerequisites

- Familiarity with Node.js streams and raw terminal mode.
- Basic understanding of ANSI escape codes (CSI sequences).
- TypeScript generics and interface declaration merging.

## Quick Start

```bash
npm install @mariozechner/pi-tui
```

```typescript
import { Tui, ProcessTerminal } from "@mariozechner/pi-tui";
import { EditorBox } from "@mariozechner/pi-tui/components/editor";

const terminal = new ProcessTerminal();
const tui = new Tui(terminal);

const editor = new EditorBox();
editor.onSubmit = (text) => {
  tui.stop();
  console.log("Submitted:", text);
};

tui.addComponent(editor);
tui.setFocused(editor);
tui.start();
```

## Package Structure

```
packages/tui/src/
├── tui.ts                  # Tui class, Component interface, render loop
├── terminal.ts             # Terminal interface, ProcessTerminal
├── keys.ts                 # KeyId type, matchesKey(), parseKey()
├── stdin-buffer.ts         # StdinBuffer — splits batched stdin into sequences
├── editor-component.ts     # EditorComponent interface
├── autocomplete.ts         # AutocompleteProvider, overlay rendering
├── fuzzy.ts                # Fuzzy selector component
├── keybindings.ts          # Keybinding registry, TUI_KEYBINDINGS defaults
├── kill-ring.ts            # Emacs-style kill ring
├── undo-stack.ts           # Generic undo/redo stack
├── terminal-image.ts       # Kitty / iTerm2 inline image protocol
├── utils.ts                # visibleWidth, ANSI stripping, column slicing
└── components/
    ├── box.ts              # Box layout wrapper
    ├── editor.ts           # Full editor component (wraps EditorComponent)
    ├── image.ts            # Image display component
    ├── input.ts            # Single-line input
    ├── loader.ts           # Spinner / progress indicator
    ├── cancellable-loader.ts
    ├── markdown.ts         # Markdown renderer
    ├── select-list.ts      # Scrollable list with selection
    ├── settings-list.ts    # Key-value settings view
    ├── spacer.ts           # Fixed-height spacer
    ├── text.ts             # Static text display
    └── truncated-text.ts   # Text with ellipsis truncation
```

## Documentation Map

| Document | What you will learn |
|---|---|
| [C4 Context](c4-01-context.md) | How pi-tui fits into the broader system |
| [C4 Container](c4-02-container.md) | Node.js process, render loop, stdin pipeline |
| [C4 Component](c4-03-component.md) | Tui, Terminal, KeyParser, StdinBuffer, components |
| [Code Walkthrough](c4-04-code-walkthrough.md) | End-to-end: keypress → render |
| [Rendering](features/rendering.md) | Differential rendering, three strategies |
| [Editor Component](features/editor-component.md) | Multi-line editing, undo, kill-ring |
| [Markdown & Images](features/markdown-and-images.md) | Markdown rendering, Kitty inline images |
| [Extension Points](extension-points.md) | Custom components, keybindings, themes |
| [Observability](observability.md) | Performance logging, key event tracing |
| [Design Patterns](design-patterns.md) | Component, Diff, Observer, Strategy, Command |
| [Terminology](terminology.md) | Glossary of TUI-specific terms |

## Related Packages

- `packages/agent` — the agent runtime that drives the pi interactive mode
- `packages/coding-agent` — uses pi-tui for the interactive chat TUI
- See [docs/glossary.md](../../glossary.md) for cross-package terms
