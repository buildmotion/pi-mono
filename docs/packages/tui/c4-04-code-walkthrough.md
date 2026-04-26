# C4 Level 4 — Code Walkthrough: pi-tui

This walkthrough traces a complete request cycle: the user opens the app, types a message, and submits it.

## Step 1: Instantiate and Start

```typescript
const terminal = new ProcessTerminal();
const tui = new Tui(terminal);
const editor = new EditorBox();
editor.onSubmit = handleSubmit;
tui.addComponent(editor);
tui.setFocused(editor);
tui.start();
```

**Inside `tui.start()`:**

1. Calls `terminal.start(onInput, onResize)`.
2. `ProcessTerminal.start()`:
   - Calls `process.stdin.setRawMode(true)` — disables line-buffering and echo.
   - Writes `\x1b[?2004h` — enables bracketed paste mode.
   - Calls `process.kill(process.pid, "SIGWINCH")` — refreshes terminal dimensions.
   - Queries Kitty protocol: writes `\x1b[?u`.
   - Calls `setupStdinBuffer()` — constructs `StdinBuffer`, attaches the sequence-split data handler.
3. Back in `Tui.start()`, kicks off `this.renderInterval = setInterval(this.renderFrame, 16)`.

## Step 2: First Render

On the first `renderFrame()` tick:

1. For each component in `this.components`, calls `component.render(terminal.columns)`.
2. Collects all returned `string[]` arrays, concatenates into `newLines`.
3. Because `prevLines` is empty, uses the **full repaint** strategy: writes all lines.
4. Stores `newLines` as `prevLines`.

The editor returns something like:

```
["┌─ Input ─────────────────────────────────┐",
 "│ _                                        │",
 "└──────────────────────────────────────────┘"]
```

where `_` is actually `CURSOR_MARKER`. Tui finds it, strips it, and emits `\x1b[2;3H` to position the hardware cursor at row 2, column 3.

## Step 3: User Types a Character

The user presses `H`. Raw byte `0x48` arrives on `process.stdin`.

1. `StdinBuffer` receives the byte, recognises it as a printable ASCII character, emits it immediately as a single-character string `"H"`.
2. `ProcessTerminal`'s stdin data handler forwards `"H"` to `Tui`'s `onInput(data)`.
3. `Tui.onInput()`:
   - Checks if it is a Kitty key-release event via `isKeyRelease(data)`. It is not.
   - Calls `this.focused.handleInput("H")`.
4. `EditorBox.handleInput("H")`:
   - Inserts `"H"` into the internal text buffer at the cursor position.
   - Advances the cursor column.
   - Records the new text snapshot on `UndoStack`.
   - Calls `this.onChange?.("H")` if registered.
5. The component returns to the Tui event loop without touching the terminal directly.

## Step 4: Next Render Tick — Differential Repaint

16 ms later, `renderFrame()` fires:

1. `editor.render(width)` returns the updated lines with `"H"` in the text.
2. Tui compares `newLines[i]` against `prevLines[i]` for each line index.
3. Only line index 1 changed (the input line). Lines 0 and 2 (borders) are identical.
4. Tui uses the **diff-only** strategy:
   - Moves cursor to row 1: `\x1b[2;1H`.
   - Writes `\x1b[2K` (erase line).
   - Writes the new line content.
5. `prevLines` is updated.

The user sees only line 1 repainted — zero flicker.

## Step 5: User Presses Ctrl+A

`0x01` arrives on stdin. `StdinBuffer` emits `"\x01"`.

1. `Tui.onInput("\x01")` → `editor.handleInput("\x01")`.
2. Editor calls `matchesKey("\x01", keybindingFor("tui.editor.cursorLineStart"))`.
3. The default for `tui.editor.cursorLineStart` includes `["home", "ctrl+a"]`.
4. `matchesKey` parses `"\x01"` → `"ctrl+a"` and finds a match.
5. Editor moves the cursor to column 0 on the current line.
6. Next render tick repositions the hardware cursor.

## Step 6: User Presses Enter (Submit)

`"\r"` arrives. `StdinBuffer` emits it.

1. `editor.handleInput("\r")`.
2. Editor checks `matchesKey("\r", keybindingFor("tui.input.submit"))`. Match.
3. Editor calls `this.onSubmit?.(this.getText())`.
4. Application's `handleSubmit(text)` is called with the full text.
5. Application calls `tui.stop()`.

**Inside `tui.stop()`:**

1. `clearInterval(this.renderInterval)`.
2. `terminal.stop()`:
   - Restores `process.stdin.setRawMode(false)`.
   - Disables bracketed paste: `\x1b[?2004l`.
   - Shows cursor: `\x1b[?25h`.
   - Calls `drainInput()` to consume any pending Kitty key-release events so they do not leak to the parent shell.

## Step 7: Resize Handling

While the app is running, if the user resizes the terminal window:

1. `SIGWINCH` is delivered to the Node.js process.
2. `process.stdout.on("resize")` fires; `ProcessTerminal` calls the registered `onResize` callback.
3. `Tui.onResize()`:
   - Clears `prevLines` (forces full repaint).
   - Calls `terminal.clearScreen()`.
   - Calls `component.invalidate()` on all components (clears caches).
   - Schedules an immediate `renderFrame()` call.

## Key Invariants

| Invariant | Enforced by |
|---|---|
| One hardware cursor position at a time | Tui extracts exactly one `CURSOR_MARKER` per frame |
| Focused component receives input first | `Tui.onInput` routes to `this.focused` before listeners |
| No direct terminal writes from components | Components return `string[]`; only `Tui` writes to terminal |
| Key-release events filtered by default | `isKeyRelease()` check before dispatch; `wantsKeyRelease` opt-in |

## Cross-References

- [Rendering deep-dive](features/rendering.md) — all three render strategies
- [Editor component](features/editor-component.md) — undo, kill-ring, autocomplete internals
- [C4 Container](c4-02-container.md) — architectural view of the same flow
