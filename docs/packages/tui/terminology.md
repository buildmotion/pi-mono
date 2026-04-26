# Terminology — pi-tui

**ANSI escape codes**
Control sequences embedded in terminal output strings that set colors, move the cursor, or clear lines. Example: `\x1b[1m` = bold, `\x1b[0m` = reset.

**Bracketed paste mode**
A terminal feature where pasted text is wrapped in special markers (`\x1b[200~` … `\x1b[201~`) so the application can distinguish paste from typed input. pi-tui enables this on startup to prevent pasted newlines from accidentally submitting the editor.

**Component**
Any object with a `render(width: number, height: number): string[]` method. The Tui container stacks component output lines top-to-bottom.

**Differential rendering**
Comparing the previous frame's lines with the new frame's lines and only repainting lines that changed. Reduces terminal I/O and prevents visible flicker on slow connections.

**East Asian Width (EAW)**
A Unicode property that marks certain characters (CJK ideographs, full-width forms) as occupying two terminal columns instead of one. pi-tui accounts for EAW when calculating cursor positions and line widths. Source: `src/utils.ts`.

**Focus**
The component that currently receives keyboard input. Only one component is focused at a time. The Tui routes key events to the focused component.

**Frame**
One complete output of all components written to the terminal. A frame is triggered by `tui.render()`.

**IME (Input Method Editor)**
A system-level input method used for languages with large character sets (Chinese, Japanese, Korean). IME converts sequences of keystrokes into characters. pi-tui uses bracketed paste and composition event detection to handle IME input correctly.

**iTerm2 protocol**
An inline image protocol used by iTerm2 and compatible terminals. Images are base64-encoded and wrapped in OSC escape sequences: `\x1b]1337;File=...:<data>\x07`.

**Kill ring**
An Emacs-style buffer that stores the last N cut or copied text regions. Pressing the yank key (Ctrl+Y) pastes the most recent entry; repeated yanks cycle through older entries. Source: `src/kill-ring.ts`.

**Kitty protocol**
An inline image protocol developed for the Kitty terminal emulator. Images are transmitted in chunks using APC escape sequences with base64 payloads. Widely supported by modern terminals (WezTerm, Ghostty).

**Render loop**
The cycle of: update component state → call `render()` on each component → diff with previous output → write changed lines to stdout. In pi-tui this is triggered reactively (on input or explicit `tui.render()` call) rather than on a fixed timer.

**Synchronized output (CSI 2026)**
A terminal escape sequence (`\x1b[?2026h` to begin, `\x1b[?2026l` to end) that instructs the terminal to buffer all output between the markers and paint it atomically. Eliminates flicker on terminals that support the feature.

**TUI (Terminal User Interface)**
An interactive application that renders in the terminal using text, ANSI colors, and cursor movement instead of a graphical window system.

---

**Back to:** [README](./README.md) | [Global Glossary](../../glossary.md)
