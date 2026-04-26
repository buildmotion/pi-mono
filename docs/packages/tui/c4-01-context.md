# C4 Level 1 — Context Diagram: pi-tui

The context level answers: **who uses pi-tui, and what external systems does it talk to?**

## Diagram (text representation)

```
┌────────────────────────────────────────────────────────────────┐
│                        System Context                          │
│                                                                │
│   ┌───────────┐                                                │
│   │ Developer │  writes TypeScript that imports pi-tui         │
│   │  / App    │                                                │
│   └─────┬─────┘                                                │
│         │ uses                                                  │
│         ▼                                                       │
│   ┌───────────────────────────────────────┐                    │
│   │               pi-tui                  │                    │
│   │   (@mariozechner/pi-tui)              │                    │
│   │                                       │                    │
│   │  • Manages render loop                │                    │
│   │  • Parses keyboard input              │                    │
│   │  • Renders components to stdout       │                    │
│   └──────────────┬──────┬────────────────┘                    │
│                  │      │                                       │
│          writes  │      │  reads raw bytes                     │
│          ANSI    │      │  from process.stdin                  │
│                  ▼      ▼                                       │
│   ┌──────────────────────────────────────┐                    │
│   │         Terminal Emulator             │                    │
│   │  (Kitty, iTerm2, xterm, Windows      │                    │
│   │   Terminal, SSH clients, etc.)        │                    │
│   └──────────────────────────────────────┘                    │
└────────────────────────────────────────────────────────────────┘
```

## Actors and Systems

### Developer / Application

The consuming application imports pi-tui and:

1. Creates a `ProcessTerminal` (or a test double implementing `Terminal`).
2. Creates a `Tui` instance, adds components, and calls `tui.start()`.
3. Receives keyboard events via component `handleInput` callbacks.
4. Emits output by returning `string[]` from component `render()` methods.

The application developer never writes escape codes directly — pi-tui handles all ANSI sequencing.

### pi-tui

The library itself. It owns:

- The render loop (`requestAnimationFrame`-style polling via `setInterval`).
- All ANSI output written to `process.stdout`.
- The stdin pipeline (raw mode, sequence splitting, key parsing).

### Terminal Emulator

pi-tui talks to the terminal emulator via two channels:

| Channel | Direction | Protocol |
|---|---|---|
| `process.stdout` | pi-tui → terminal | ANSI / VT100 escape codes, OSC sequences |
| `process.stdin` | terminal → pi-tui | Raw byte stream (legacy VT sequences or Kitty keyboard protocol) |

#### External Protocols Used

**Kitty Keyboard Protocol**
pi-tui probes the terminal on startup by sending `\x1b[?u` (DA query). If the terminal responds with `\x1b[?<flags>u`, Kitty protocol is activated. This unlocks key-release events, unambiguous modifier reporting, and shifted symbol keys.
See: <https://sw.kovidgoyal.net/kitty/keyboard-protocol/>

**iTerm2 / Kitty Inline Image Protocol**
`src/terminal-image.ts` implements both the iTerm2 image protocol (base64 inside OSC 1337) and the Kitty terminal graphics protocol. Terminal capability is probed at startup; the best available protocol is used.

**CSI 2026 — Synchronized Output**
When the terminal supports it, pi-tui wraps each full render frame in `\x1b[?2026h` (begin synchronized update) and `\x1b[?2026l` (end). This prevents tearing when many lines change at once.

**Bracketed Paste Mode**
pi-tui enables bracketed paste (`\x1b[?2004h`) so that pasted text is wrapped in `\x1b[200~` ... `\x1b[201~` markers and handled by the editor rather than being interpreted as individual keystrokes.

**OSC 9;4 — Progress Indicator**
Optional progress bar in the terminal tab title, implemented via `ProcessTerminal.setProgress()`.

## What pi-tui Does NOT Do

- No network I/O.
- No file system access beyond the optional write-log (`PI_TUI_WRITE_LOG`).
- No external process spawning.
- No dependency on a browser or DOM.

## Environment Variables

| Variable | Effect |
|---|---|
| `PI_TUI_WRITE_LOG` | Path (file or directory) to write a raw stdin log for debugging |

## Cross-References

- [Container diagram](c4-02-container.md) — internal process structure
- [Terminology](terminology.md) — definitions for CSI, OSC, bracketed paste
- [docs/glossary.md](../../glossary.md) — project-wide terms
