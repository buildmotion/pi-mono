# Design Patterns — pi-tui

## 1. Composite (Component Tree)

**Pattern:** Objects and containers of objects share the same interface.

In pi-tui, every visual element implements `render(width, height): string[]`. The `Tui` class is itself a container that calls `render()` on each child in order, stacking their output lines top-to-bottom. You can nest containers to create split-pane layouts.

**Why:** Allows new components to be added without touching the Tui core.

---

## 2. Strategy (Render Strategy)

**Pattern:** An algorithm is selected at runtime from a family of interchangeable implementations.

pi-tui switches between three render strategies:

| Strategy | When used | Mechanism |
|----------|-----------|-----------|
| Full repaint | First frame / resize | Clears screen, writes all lines |
| Differential | Stable terminal | Computes line diff, only repaints changed lines |
| Synchronized | Terminal supports CSI 2026 | Wraps repaint in `\x1b[?2026h` / `\x1b[?2026l` to suppress flicker |

**Why:** Different terminals need different approaches. The strategy is selected once at startup and never requires branching inside the render loop.

---

## 3. Observer (Input Events)

**Pattern:** Components subscribe to events emitted by a central dispatcher.

The `Tui` reads stdin, parses key sequences, and dispatches events to the focused component. Components call `editor.on("submit", handler)` — they never poll stdin directly.

**Why:** Decouples input source from input consumers. A test harness can inject synthetic key events without touching stdin.

---

## 4. Command (Keybindings)

**Pattern:** An action is encapsulated as an object that can be stored, passed, and invoked later.

Each keybinding maps a key sequence to a named command string (`"submit"`, `"clear"`, `"undo"`). The editor dispatches the command string; handlers respond to names, not key sequences. Custom keybindings reuse existing handler code without modification.

**Why:** Keybindings are user-configurable without touching handler logic.

---

## 5. Memento (Undo Stack and Kill Ring)

**Pattern:** Capture and restore object state without exposing internals.

`UndoStack` (`src/undo-stack.ts`) records snapshots of editor text and cursor position. The kill ring (`src/kill-ring.ts`) buffers cut text for yank (paste). Both are implemented as simple arrays with a cursor pointer.

**Why:** Provides standard Emacs-style editing UX expected by developer-focused terminal tools.

---

**What's Next:** [Terminology](./terminology.md) | [Extension Points](./extension-points.md)
