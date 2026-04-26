# `@mariozechner/pi-tui` — Terminal UI Framework

## What is it?

`pi-tui` is a lightweight, TypeScript-native terminal user interface framework. It provides a component model, a differential renderer that only repaints changed lines, and built-in primitives (text, editor, Markdown, autocomplete, overlays, inline images) for building interactive terminal applications without flicker or third-party native dependencies.

**npm package:** `@mariozechner/pi-tui`  
**Version:** 0.70.2  
**Node requirement:** >= 20.0.0  
**License:** MIT  
**Primary consumer:** `packages/coding-agent` (the `pi` CLI)

---

## The Problem It Solves

Building interactive terminal applications in Node.js is harder than it looks:

- Naive `console.log` causes the whole screen to scroll — unusable for live-updating UIs.
- Libraries like `blessed` or `ink` either pull in native dependencies, impose a React-like model, or have incomplete Unicode support.
- Rendering the same unchanged text repeatedly causes visible flicker.
- Handling special inputs (paste, IME, wide East Asian characters, inline images) requires deep terminal knowledge.

`pi-tui` solves all of these with a small, focused API and zero mandatory native dependencies.

---

## Audience

- Node.js developers building interactive CLI tools (chat interfaces, progress dashboards, file selectors)
- Consumers of `pi-coding-agent` who want to build similar terminal UIs
- Any developer who needs to display live-updating content in a terminal without flicker

---

## Core Concepts

### TUI (the container)

`TUI` is the root manager. It owns the terminal output connection, holds the list of child components, and drives the render loop.

```typescript
import { TUI, ProcessTerminal } from '@mariozechner/pi-tui';

const terminal = new ProcessTerminal(); // wraps process.stdout / stdin
const tui = new TUI(terminal);

tui.addChild(myComponent);
tui.start();
// … later …
tui.stop();
```

### Components

Every UI element implements the `Component` interface:

```typescript
interface Component {
  render(width: number): string[];   // returns one string per line, no line may exceed `width`
  handleInput?(data: string): void;  // called when this component has focus
  invalidate?(): void;               // called to clear cached render state
}
```

The framework calls `render(width)` on each component each frame and applies the three-strategy differential renderer.

### Differential Rendering (how flicker-free updates work)

Each frame, `TUI` compares the new rendered lines to what was last written:

1. **Strategy A — Overwrite in place:** If a line changed and is the same length, overwrite with a carriage return (`\r`) and rewrite the line. No scrolling.
2. **Strategy B — Erase and redraw:** If the length changed, emit `\033[K` (erase to end of line) and redraw.
3. **Strategy C — Full repaint:** Only used as a fallback when the terminal size changed.

Because only changed lines are touched, the user sees no flicker even during fast streaming updates.

### Synchronized Output (CSI 2026)

Before writing a frame, `pi-tui` emits the Begin Synchronized Update escape sequence (`\x1b[?2026h`) and the End Synchronized Update sequence (`\x1b[?2026l`) after. Terminals that support this (most modern ones) buffer the output and paint it atomically — completely eliminating the partial-update artifact you sometimes see when many lines change at once.

### Overlays

Overlays render components on top of the existing content — useful for dialogs, menus, and modal UI.

```typescript
const handle = tui.showOverlay(myDialogComponent, {
  anchor: 'center',
  width: '80%',
  maxHeight: 20,
  margin: 2,
});

// Later
handle.hide();
handle.setHidden(true); // temporarily invisible
handle.focus();
```

---

## Built-in Components

| Component | What it does |
|-----------|-------------|
| `Text` | Static text with ANSI color support |
| `TruncatedText` | Text that clips to available width with an ellipsis |
| `Input` | Single-line text input field |
| `Editor` | Multi-line editor with Emacs-style keybindings, undo/redo, kill ring |
| `Markdown` | Renders a Markdown string with syntax-highlighted code blocks |
| `Loader` | Animated spinner |
| `SelectList` | Scrollable, keyboard-navigable list |
| `SettingsList` | Key-value settings display |
| `Spacer` | Vertical whitespace |
| `Image` | Inline image (Kitty or iTerm2 graphics protocol) |
| `Box` | Bordered container |
| `Container` | Layout grouping |

---

## Editor Component (deep dive)

The `Editor` is the most complex component and is the primary input widget in the `pi` coding agent.

Features:
- Multi-line text with hard and soft wrapping
- Emacs-style navigation: `Ctrl+A`/`E` (line start/end), `Ctrl+F`/`B` (character), `Alt+F`/`B` (word)
- Kill ring: `Ctrl+K` (kill to end of line), `Ctrl+Y` (yank), `Alt+W` (copy)
- Undo/redo stack
- Bracketed paste mode: pastes of more than 10 lines show a marker instead of dumping raw text, preventing accidental multi-line submission
- Autocomplete for file paths and slash commands
- IME support via the `Focusable` interface and `CURSOR_MARKER` injection

```typescript
const editor = new Editor(tui, theme);
editor.onSubmit = (text) => {
  console.log('User submitted:', text);
};
editor.onAbort = () => {
  console.log('User pressed Escape');
};
tui.addChild(editor);
```

---

## Inline Image Rendering

The `Image` component checks the terminal environment and uses:

- **Kitty graphics protocol** (kitty, WezTerm, Ghostty) — sends base64-encoded PNG via the `\x1b_G` control sequence
- **iTerm2 protocol** (iTerm2, some others) — sends via `\x1b]1337;File=` escape

If neither protocol is supported, the image is silently skipped (or replaced with alt text).

---

## Autocomplete

`pi-tui` includes fuzzy-search autocomplete (used in `Editor` for file paths and `/commands`):

```typescript
import { fuzzyMatch } from '@mariozechner/pi-tui';

const candidates = ['read_file', 'write_file', 'list_dir'];
const matches = fuzzyMatch('rf', candidates);  // → ['read_file']
```

The autocomplete dropdown renders as an overlay positioned below the cursor line.

---

## Keybindings

All keybindings are defined in `keybindings.ts` as a `DEFAULT_EDITOR_KEYBINDINGS` object. They are fully configurable — no hardcoded key checks in component code.

```typescript
import { DEFAULT_EDITOR_KEYBINDINGS } from '@mariozechner/pi-tui';
// Customize as needed before passing to Editor
```

---

## Source Layout

```
packages/tui/src/
├── index.ts               Public re-exports
├── tui.ts                 TUI class, render loop, overlay management
├── terminal.ts            ProcessTerminal — wraps stdin/stdout, handles raw mode
├── components/            All built-in components
│   ├── text.ts
│   ├── editor-component.ts  (delegated to editor-component.ts)
│   ├── markdown.ts
│   ├── select-list.ts
│   └── …
├── editor-component.ts    Multi-line editor logic
├── autocomplete.ts        Fuzzy autocomplete overlay
├── keybindings.ts         DEFAULT_EDITOR_KEYBINDINGS
├── keys.ts                Key name constants and raw escape parsing
├── kill-ring.ts           Emacs-style kill ring
├── undo-stack.ts          Undo/redo stack
├── stdin-buffer.ts        Buffers raw stdin input for correct escape sequence handling
├── terminal-image.ts      Kitty/iTerm2 inline image support
├── fuzzy.ts               Fuzzy string matching
└── utils.ts               ANSI string width, wrapping, truncation helpers
```

---

## Key Dependencies

| Dependency | Purpose |
|-----------|---------|
| `chalk` | ANSI color helpers |
| `marked` | Markdown parsing (for Markdown component) |
| `get-east-asian-width` | Correct display width for CJK characters |
| `mime-types` | MIME type detection for inline images |
| `koffi` (optional) | Native clipboard access on Linux |

---

## Pros and Cons

**Pros**
- Flicker-free differential rendering out of the box
- No mandatory native dependencies
- Correct Unicode/East Asian width handling
- Built-in Editor is feature-rich (undo, kill ring, autocomplete, bracketed paste)
- Inline image support for Kitty and iTerm2
- Fully configurable keybindings

**Cons**
- Terminal-only — not usable in browsers
- Some features (inline images, synchronized output) depend on terminal emulator support
- Component model is simpler than React/Ink — no virtual DOM diffing within a component; components manage their own re-render caching
- No layout engine — components are stacked vertically; horizontal layout requires manual column math

---

## How to Use It (Junior Developer Walkthrough)

Think of `TUI` as a vertical stack of components. Each component is a black box that knows how to paint itself. The `TUI` manager calls `render()` on each one every frame and only repaints the lines that changed.

1. **Install:**
   ```bash
   npm install @mariozechner/pi-tui
   ```

2. **Hello World:**
   ```typescript
   import { TUI, Text, ProcessTerminal } from '@mariozechner/pi-tui';

   const tui = new TUI(new ProcessTerminal());
   tui.addChild(new Text('Hello, terminal!'));
   tui.start();
   ```

3. **Add an editor:**
   ```typescript
   import { Editor } from '@mariozechner/pi-tui';

   const editor = new Editor(tui, { /* theme options */ });
   editor.onSubmit = (text) => {
     tui.addChild(new Text(`You typed: ${text}`));
   };
   tui.addChild(editor);
   ```

4. **Add an overlay (dialog):**
   ```typescript
   const dialog = new Text('Press any key to close');
   const handle = tui.showOverlay(dialog, { anchor: 'center', width: 40 });
   dialog.handleInput = () => handle.hide();
   ```

5. **Stop cleanly on Ctrl+C:**
   ```typescript
   process.on('SIGINT', () => { tui.stop(); process.exit(0); });
   ```
