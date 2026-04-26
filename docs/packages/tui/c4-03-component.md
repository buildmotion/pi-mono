# C4 Level 3 — Component Diagram: pi-tui

This level zooms into the major TypeScript classes and interfaces inside pi-tui.

## Tui Class

```
┌────────────────────────────────────────────────────────┐
│                    Tui (tui.ts)                        │
│                                                        │
│  Fields                                                │
│  ─────                                                 │
│  terminal: Terminal                                    │
│  components: Component[]                               │
│  focused: (Component & Focusable) | null               │
│  overlays: Overlay[]                                   │
│  inputListeners: InputListener[]                       │
│  prevLines: string[]          ← last rendered frame    │
│  renderInterval: NodeJS.Timer                          │
│                                                        │
│  Methods                                               │
│  ───────                                               │
│  start()          → enables terminal, starts loop      │
│  stop()           → disables terminal, clears screen   │
│  addComponent()   → appends to component tree          │
│  setFocused()     → routes input to component          │
│  addInputListener() → low-level key hook               │
│  addOverlay()     → modal / dropdown layer             │
│  render()         → private: diff + repaint            │
└────────────────────────────────────────────────────────┘
```

## Component Interface

Every renderable unit implements `Component`:

```typescript
interface Component {
  // Required: return an array of terminal lines for the given width
  render(width: number): string[];

  // Optional: receive raw key sequences when focused (or via listener)
  handleInput?(data: string): void;

  // Optional: set true to receive Kitty key-release events
  wantsKeyRelease?: boolean;

  // Required: clear any cached render state
  invalidate(): void;
}
```

The `Focusable` interface extends this contract for components that display a hardware cursor:

```typescript
interface Focusable {
  focused: boolean;   // set by Tui when focus changes
}
```

When focused, the component must emit `CURSOR_MARKER` (`\x1b_pi:c\x07`) inside its render output at the cursor position. Tui strips the marker, computes the cursor row and column, and emits `\x1b[row;colH` so the OS places the hardware cursor there (needed for IME candidate window positioning).

## Built-In Components

### box (`components/box.ts`)

A layout wrapper that renders child components inside a Unicode border. Accepts a title, border color function, and optional padding.

### editor (`components/editor.ts`)

A full-featured multi-line editor built on top of `EditorComponent`. Provides:

- Undo/redo via `UndoStack`
- Emacs-style kill ring via `KillRing`
- Autocomplete overlay via `AutocompleteProvider`
- History navigation (up/down arrows cycle through submission history)
- Bracketed paste ingestion
- Wide-character and IME support

### input (`components/input.ts`)

A single-line text input that accepts a placeholder and a submit callback. Simpler than `editor`; no multi-line support, no kill ring.

### select-list (`components/select-list.ts`)

A scrollable list of items. Emits `onSelect` / `onCancel` callbacks. Renders a highlight bar on the active item. Supports page-up / page-down navigation via the `tui.select.*` keybindings.

### settings-list (`components/settings-list.ts`)

Displays a key-value table of settings. Used in the pi interactive mode settings panel.

### markdown (`components/markdown.ts`)

Renders Markdown text to ANSI-styled terminal lines. Handles headings, bold, italic, inline code, fenced code blocks, and lists. Uses `visibleWidth` from `utils.ts` to handle wide characters correctly.

### image (`components/image.ts`)

Displays an image inline in the terminal. Delegates to `terminal-image.ts` which selects Kitty or iTerm2 protocol based on detected capability.

### loader (`components/loader.ts`)

A spinner animation for long-running operations. `cancellable-loader.ts` adds a key handler that fires an `onCancel` callback.

### spacer (`components/spacer.ts`)

Emits N blank lines. Used for vertical spacing between components.

### text (`components/text.ts`)

Static text display. Wraps long lines to the available width.

### truncated-text (`components/truncated-text.ts`)

Like `text` but truncates to a maximum number of lines and appends an ellipsis.

## Supporting Classes

### StdinBuffer (`stdin-buffer.ts`)

```
StdinBuffer
  ├── EventEmitter (emits "data" events)
  ├── _parser: FSM that splits CSI / OSC / printable sequences
  └── _timeout: reassembly timer for multi-byte compositions
```

### KillRing (`kill-ring.ts`)

A circular buffer of cut text entries. Implements the Emacs yank/yank-pop model: `ctrl+w` or `ctrl+k` pushes to the ring; `ctrl+y` yanks the top entry; repeated `alt+y` cycles backwards through the ring.

### UndoStack (`undo-stack.ts`)

A generic stack that records state snapshots. The editor pushes a snapshot on every edit action. `ctrl+/` pops the previous snapshot and restores it.

### AutocompleteProvider (`autocomplete.ts`)

An interface that the editor calls to populate the autocomplete overlay:

```typescript
interface AutocompleteProvider {
  getSuggestions(prefix: string): string[] | Promise<string[]>;
}
```

The overlay renders as a floating dropdown anchored to the cursor position.

### FuzzySelector (`fuzzy.ts`)

A standalone component that overlays the terminal with a text input and a filtered list. Used for model selection and file picker scenarios. Implements fuzzy scoring internally.

### Keybindings (`keybindings.ts`)

A compile-time-safe keybinding registry using TypeScript interface declaration merging:

```typescript
// Add new bindings from your own package:
declare module "@mariozechner/pi-tui" {
  interface Keybindings {
    "myapp.action.doThing": true;
  }
}
```

At runtime, `TUI_KEYBINDINGS` holds the default key assignments. The `matchesKeybinding()` helper resolves a `Keybinding` name to its configured `KeyId[]` and tests the current input against them. This keeps keybinding checks out of component code and avoids hardcoded key strings.

### TerminalImage (`terminal-image.ts`)

Handles the two inline image protocols:

| Protocol | Terminal | Mechanism |
|---|---|---|
| Kitty graphics | Kitty, ghostty | APC sequence `\x1b_Ga=T,...data...\x1b\\` |
| iTerm2 | iTerm2, WezTerm | OSC 1337 `\x1b]1337;File=...data...\x07` |

Capability is probed once via terminal query responses and cached in `getCapabilities()`. The `isImageLine()` predicate is used by the Tui renderer to skip diffing on image lines (they must always be fully repainted).

### Utils (`utils.ts`)

- `visibleWidth(str)` — computes display columns accounting for wide (CJK) characters and ANSI escape sequences.
- `sliceByColumn(str, start, end)` — slices a string by display column, correctly handling multi-column characters.
- `extractSegments(str)` — splits a line into alternating text and escape-sequence segments.

## Cross-References

- [Code walkthrough](c4-04-code-walkthrough.md) — how these components interact at runtime
- [Editor component deep-dive](features/editor-component.md)
- [Design patterns](design-patterns.md) — Component, Observer, Strategy, Command patterns in pi-tui
