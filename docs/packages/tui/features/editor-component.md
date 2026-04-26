# Editor Component

## Learning Objectives

By the end of this walkthrough you will be able to:

- Explain what `EditorComponent` does and when to use it.
- Describe the undo stack and kill ring and how they relate to Emacs conventions.
- Configure keybindings and listen for the submit event.
- Enable IME support for CJK input.
- Wire up the autocomplete overlay.

---

## What the Editor Component Does

`EditorComponent` (defined in `src/editor-component.ts`) is the multi-line text input widget at the heart of pi-tui. It owns the cursor, manages the text buffer, and translates raw terminal key sequences into editing actions. Every keystroke the user types goes through this component before anything else in the UI sees it.

The component renders itself as a block of lines inside the terminal. It tracks the cursor position as a `(row, col)` pair inside the buffer. On every render cycle the component emits the lines that should be drawn, including the ANSI escape sequence that positions the cursor correctly.

---

## Key Features

### Multi-line Input

The buffer is stored as an array of strings, one element per line. The Enter key (when not configured as submit) appends a new empty string and moves the cursor to the beginning of the new line. Backspace across a line boundary merges two lines.

### Cursor Movement

Supported movements out of the box:

| Action | Default key |
|---|---|
| Move left / right | Arrow keys, Ctrl-B / Ctrl-F |
| Move up / down | Arrow keys, Ctrl-P / Ctrl-N |
| Move to line start / end | Ctrl-A / Ctrl-E |
| Move word left / right | Alt-B / Alt-F |
| Move to buffer start / end | Alt-< / Alt-> |

All cursor movements are column-aware and respect East Asian Wide characters (each wide character counts as two columns). See [terminology.md](../terminology.md) for a definition of East Asian Width.

### Undo Stack

Every destructive edit (insert, delete, kill) pushes a snapshot of the buffer and cursor position onto the undo stack. `Ctrl-/` or `Ctrl-_` pops the most recent snapshot and restores it. The stack has a configurable maximum depth (default 200 entries).

### Kill Ring (Emacs-style Cut/Paste Buffer)

The kill ring is a cyclic buffer of "killed" text regions. Killing text (`Ctrl-K` kills to end of line, `Ctrl-U` kills to start of line, `Alt-D` kills the next word) appends the text to the current kill ring entry. Yanking (`Ctrl-Y`) inserts the most recent entry at the cursor. `Alt-Y` (yank-pop) cycles through older entries. This is the same model used by GNU Emacs and Readline.

---

## IME Support for CJK Input

Input Method Editors compose characters across multiple keystrokes before committing them to the buffer. The editor detects IME composition via the bracketed sequences that most modern terminal emulators emit during composition. While a composition is in progress the editor shows the candidate string underlined and does not run autocomplete or submit. When the composition commits the final string is inserted as a single undo unit.

On terminals that do not emit composition events, CJK characters arrive as ordinary Unicode code points and are inserted directly.

---

## Autocomplete Overlay

`src/autocomplete.ts` exports an `Autocomplete` class that the editor consults on every keystroke (unless inside an IME composition). The overlay renders as a floating list below or above the cursor, depending on available space.

To attach autocomplete:

```ts
import { EditorComponent } from "@mariozechner/pi-tui";
import { Autocomplete } from "@mariozechner/pi-tui";

const complete = new Autocomplete(async (prefix) => {
    // Return candidate strings based on the current word prefix.
    return candidates.filter((c) => c.startsWith(prefix));
});

const editor = new EditorComponent({ autocomplete: complete });
```

The overlay intercepts `Tab`, `Shift-Tab`, `Enter` (when the list is open), and `Escape` before the editor sees them, so the editor's own keybindings do not fire while the list is visible.

---

## Configurable Keybindings

All keybindings live in `src/keybindings.ts`. The file exports a `DEFAULT_EDITOR_KEYBINDINGS` object that maps action names to key descriptors. You can override individual bindings by spreading your overrides on top:

```ts
import {
    EditorComponent,
    DEFAULT_EDITOR_KEYBINDINGS,
} from "@mariozechner/pi-tui";

const editor = new EditorComponent({
    keybindings: {
        ...DEFAULT_EDITOR_KEYBINDINGS,
        submit: { key: "enter", modifiers: ["ctrl"] }, // Ctrl-Enter to submit
    },
});
```

Never hardcode key checks directly in application code; always modify the keybindings object.

---

## Listening for the Submit Event

The editor emits a `submit` event when the user presses the configured submit key (default: `Enter` with no modifiers when the editor is in single-line mode, or `Alt-Enter` in multi-line mode).

```ts
editor.on("submit", (text: string) => {
    console.log("User submitted:", text);
    editor.clear(); // reset the buffer after submission
});
```

---

## Full Setup Example

```ts
import { Tui, EditorComponent, Autocomplete } from "@mariozechner/pi-tui";

const tui = new Tui();

const complete = new Autocomplete(async (prefix) => {
    const commands = ["/help", "/clear", "/exit"];
    return commands.filter((c) => c.startsWith(prefix));
});

const editor = new EditorComponent({
    placeholder: "Type a message…",
    multiline: true,
    autocomplete: complete,
});

editor.on("submit", (text) => {
    tui.appendMessage({ role: "user", content: text });
    editor.clear();
});

tui.add(editor);
tui.start();
```

---

## Check Yourself

1. What data structure holds the text buffer in `EditorComponent`?
2. Explain the difference between the undo stack and the kill ring.
3. Which keybinding cycles through older kill-ring entries?
4. Why does the autocomplete overlay intercept `Enter` before the editor does?
5. How would you change the submit keybinding to `Ctrl-Enter`?
6. What happens to cursor movement when a line contains East Asian wide characters?
7. Why does the editor suspend autocomplete during IME composition?

---

*Cross-references: [terminology.md](../terminology.md) · [docs/glossary.md](../../../glossary.md)*
