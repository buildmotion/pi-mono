# Rendering in pi-tui

pi-tui renders without a retained scene graph. Each tick, components return plain `string[]`, and the render engine decides the cheapest way to update the physical terminal.

## The Render Loop

`Tui.start()` calls `setInterval(renderFrame, 16)` — approximately 60 frames per second. On each tick:

1. Collect component output: `newLines = components.flatMap(c => c.render(width))`.
2. Choose a render strategy.
3. Write the minimal set of escape sequences to bring the terminal up to date.
4. Save `newLines` as `prevLines` for the next tick.

## Three Render Strategies

### Strategy 1: Full Repaint

**When used:**
- First render after `start()`.
- After a terminal resize (window dimensions changed).
- After `tui.invalidateAll()` is called (e.g., theme change).

**How it works:**

1. Move cursor to (0, 0): `\x1b[H`.
2. For each line in `newLines`, write the line content followed by `\x1b[K` (erase to end of line).
3. If `newLines` is shorter than the terminal height, emit `\x1b[J` (erase from cursor to end of screen).

Full repaint is the simplest strategy but writes every line every frame. It is only used when the render state is unknown.

### Strategy 2: Diff-Only

**When used:**
- Normal operation after the first render.
- When CSI 2026 (synchronized output) is not supported.

**How it works:**

1. Compare `newLines[i]` against `prevLines[i]` for every line index.
2. For lines that differ:
   a. Move cursor to `(i, 0)`: `\x1b[<i+1>;1H`.
   b. Write `\x1b[2K` (erase entire line).
   c. Write the new line content.
3. If `newLines.length < prevLines.length`, erase the extra trailing lines.

This strategy minimizes bytes written. On a 40-line terminal where only one line changed, only three escape sequences are emitted.

**Wide character handling:** Lines containing wide (CJK/emoji) characters require special care. `visibleWidth()` and `sliceByColumn()` from `utils.ts` are used to compute correct column positions so the cursor placement sequences remain accurate.

**Image lines:** Lines detected by `isImageLine()` (Kitty/iTerm2 image data) are never diffed — they are always fully rewritten because image protocol sequences carry binary payloads that cannot be compared character-by-character.

### Strategy 3: Synchronized Output (CSI 2026)

**When used:**
- Terminal reports support for CSI 2026 (via `\x1b[?2026h` capability query).
- The diff-only strategy is applied *inside* the synchronized block.

**How it works:**

```
\x1b[?2026h   ← begin synchronized update (terminal freezes screen)
  ... diff-only writes ...
\x1b[?2026l   ← end synchronized update (terminal flushes atomically)
```

CSI 2026 instructs the terminal to buffer all output between the begin and end markers and render it in a single compositing pass. This eliminates tearing on rapidly updating lines. It is the preferred strategy when available because it gives the best visual result at no extra code cost over the diff-only strategy.

## Cursor Placement

Components that implement `Focusable` emit `CURSOR_MARKER = "\x1b_pi:c\x07"` at the exact cursor position in their render output.

After diffing, the Tui renderer scans `newLines` for `CURSOR_MARKER`. When found:

1. Record the line index `row` and compute column `col` from the marker's position in the string using `visibleWidth()`.
2. Strip the marker from the line before writing it to the terminal.
3. After all line writes, emit `\x1b[row;colH` to position the hardware cursor.

This design means components never directly control cursor position — they declaratively report where the cursor *should* be, and the renderer handles the actual ANSI sequence.

## Wide Characters and ANSI Stripping

Terminal cells are monospace. ASCII characters occupy exactly 1 column. CJK characters (Unicode codepoints ≥ U+1100 in certain ranges) and emoji occupy 2 columns.

`visibleWidth(str)` in `utils.ts` accounts for:
- ANSI escape sequences (zero display width).
- Wide characters (2 columns each).
- Surrogate pairs and composite emoji sequences.

`sliceByColumn(str, start, end)` slices a string by display column rather than character index. This is used when the editor needs to render a window of text that fits within a given pixel width.

## Invalidation

When a component's cached render state is stale (e.g., a theme color changed), call `component.invalidate()`. The Tui class calls `invalidate()` on all components after a resize and provides `tui.invalidateAll()` for forced full refreshes.

Components that do not cache anything can implement `invalidate()` as a no-op.

## Performance Profile

On a typical 220×50 terminal:

| Event | Lines written | Bytes (approx.) |
|---|---|---|
| Idle frame (nothing changed) | 0 | ~0 |
| Single line text change | 1 | ~230 |
| Full repaint | 50 | ~11 500 |
| Resize | 50 (forced full repaint) | ~11 500 |

Because the diff-only strategy writes zero bytes on idle frames, 60 fps polling has negligible CPU cost in steady state.

## Cross-References

- [Observability](../observability.md) — how to measure render latency
- [C4 Code Walkthrough](../c4-04-code-walkthrough.md) — rendering in context of a user keypress
- [Terminology](../terminology.md) — synchronized output, differential rendering, CSI
