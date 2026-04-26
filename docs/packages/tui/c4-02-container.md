# C4 Level 2 — Container Diagram: pi-tui

The container level shows the major runtime units inside a Node.js process that uses pi-tui.

## Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          Node.js Process                                 │
│                                                                          │
│  ┌────────────────┐      ┌──────────────────────────────────────────┐   │
│  │  Application   │      │              pi-tui Library               │   │
│  │  (your code)   │─────▶│                                          │   │
│  └────────────────┘      │  ┌─────────────────────────────────┐    │   │
│                           │  │         Tui (tui.ts)            │    │   │
│                           │  │                                 │    │   │
│                           │  │  • Component tree               │    │   │
│                           │  │  • Render loop (setInterval)    │    │   │
│                           │  │  • Differential line diff       │    │   │
│                           │  │  • Input listener registry      │    │   │
│                           │  └──────────┬──────────────────────┘    │   │
│                           │             │                            │   │
│                           │      ┌──────┴──────┐                    │   │
│                           │      │             │                     │   │
│                           │  ┌───▼──────┐  ┌──▼────────────────┐   │   │
│                           │  │ Terminal  │  │   StdinBuffer     │   │   │
│                           │  │(terminal  │  │  (stdin-buffer.ts)│   │   │
│                           │  │   .ts)    │  │                   │   │   │
│                           │  │           │  │ • Splits batched  │   │   │
│                           │  │ • Raw mode│  │   stdin chunks    │   │   │
│                           │  │ • ANSI out│  │ • IME timeout     │   │   │
│                           │  │ • Kitty   │  │   reassembly      │   │   │
│                           │  │   probe   │  └──────┬────────────┘   │   │
│                           │  └─────┬─────┘         │                │   │
│                           │        │       ┌────────▼────────┐      │   │
│                           │        │       │   Key Parser    │      │   │
│                           │        │       │   (keys.ts)     │      │   │
│                           │        │       │                 │      │   │
│                           │        │       │ • matchesKey()  │      │   │
│                           │        │       │ • parseKey()    │      │   │
│                           │        │       │ • KeyId type    │      │   │
│                           │        │       └────────┬────────┘      │   │
│                           │        │                │               │   │
│                           │        │       ┌────────▼────────┐      │   │
│                           │        │       │  Component Tree  │      │   │
│                           │        │       │  (Component[])  │      │   │
│                           │        │       └─────────────────┘      │   │
│                           └──────────────────────────────────────────┘   │
│                                                                          │
│  process.stdout ──────────────────────────────────────── Terminal Emu.  │
│  process.stdin  ────────────────────────────────────────▶               │
└──────────────────────────────────────────────────────────────────────────┘
```

## Containers

### Application Code

Your TypeScript that imports pi-tui. Responsibilities:

- Instantiates `ProcessTerminal` and `Tui`.
- Constructs the component tree (built-in or custom components).
- Registers `onSubmit` / `onChange` callbacks.
- Calls `tui.start()` and eventually `tui.stop()`.

### Tui (`src/tui.ts`)

The central orchestrator. Key behaviors:

| Responsibility | Mechanism |
|---|---|
| Render loop | `setInterval` at configurable FPS (default ~60 fps) |
| Differential repaint | Three-way diff of previous vs new `string[]` output |
| Input dispatch | Calls the focused component's `handleInput()`, then registered listeners |
| Overlay management | Renders overlays (autocomplete, fuzzy finder) on top of base components |
| Cursor placement | Finds `CURSOR_MARKER` in rendered output and emits a `\x1b[row;colH` sequence |
| Focus tracking | One component holds focus at a time; Kitty key-release events are filtered unless `wantsKeyRelease` is set |

### Terminal (`src/terminal.ts`)

Two implementations:

| Class | Purpose |
|---|---|
| `ProcessTerminal` | Production: uses `process.stdin` / `process.stdout`, enables raw mode, Kitty protocol, bracketed paste |
| `Terminal` (interface) | Test doubles implement this to drive Tui in tests without a real terminal |

`ProcessTerminal` also manages:
- `StdinBuffer` construction and lifecycle.
- Kitty protocol negotiation (send query, detect response in data stream).
- `modifyOtherKeys` fallback for terminals that support it but not full Kitty.
- Windows virtual terminal input mode.
- Optional raw stdin write-log via `PI_TUI_WRITE_LOG`.

### StdinBuffer (`src/stdin-buffer.ts`)

Node.js delivers stdin in arbitrary-sized chunks. A single chunk may contain multiple CSI sequences batched together, especially during fast key presses or paste operations. `StdinBuffer` splits these chunks back into individual sequences using a finite-state machine parser, then emits them one at a time with a configurable reassembly timeout (default: 10 ms). This ensures `matchesKey()` receives complete, single-event sequences.

IME candidate input (multi-byte composition sequences) is held in the buffer until the timeout expires, then emitted as a single string to allow correct wide-character handling.

### Key Parser (`src/keys.ts`)

Stateless functions:

- `parseKey(data: string): KeyId` — converts a raw escape sequence or printable character into a structured `KeyId` string (e.g., `"ctrl+shift+left"`, `"enter"`, `"a"`).
- `matchesKey(data: string, keyId: KeyId): boolean` — convenience wrapper used by keybinding consumers.
- `isKeyRelease(data: string): boolean` — detects Kitty protocol key-release events.

The `KeyId` type is a union of all valid key identifiers, providing compile-time safety against typos in keybinding declarations.

### Component Tree

An ordered array of `Component` objects managed by `Tui`. Each component implements:

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

Components are rendered top-to-bottom. The focused component receives input events first; remaining listeners are called in registration order until one consumes the event.

## Data Flow Summary

```
process.stdin bytes
  → StdinBuffer (split into sequences)
  → Key Parser (matchesKey / parseKey)
  → focused component handleInput()
  → component updates internal state
  → Tui render loop fires (next tick)
  → each Component.render(width) called
  → diff against previous frame
  → changed lines written via Terminal.write()
  → process.stdout → terminal emulator screen
```

## Cross-References

- [Component diagram](c4-03-component.md) — internal structure of individual containers
- [Rendering deep-dive](features/rendering.md) — three render strategies explained
- [Terminology](terminology.md) — raw mode, CSI, IME
