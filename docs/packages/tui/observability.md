# Observability — pi-tui

pi-tui is a rendering library, not a business-logic framework. Observability here means measuring the performance of the render loop and understanding input patterns — without exposing sensitive user keystrokes.

---

## Measuring Render Performance

Wrap the Tui render tick to record frame duration:

```typescript
import { Tui } from "@mariozechner/pi-tui";

const tui = new Tui();
const originalRender = tui.render.bind(tui);

let frameCount = 0;
let totalMs = 0;

tui.render = () => {
  const start = performance.now();
  originalRender();
  const duration = performance.now() - start;
  frameCount++;
  totalMs += duration;
};

// Log average every 60 frames
setInterval(() => {
  if (frameCount === 0) return;
  console.error(`[tui] avg frame: ${(totalMs / frameCount).toFixed(2)}ms over ${frameCount} frames`);
  frameCount = 0;
  totalMs = 0;
}, 5000);
```

> Log to `stderr` so it does not interfere with stdout-based TUI output.

---

## Tracking Input Events (Non-Sensitive)

Log key event categories, not key values, to avoid capturing passwords or private input:

```typescript
editor.on("keydown", (key: string) => {
  const category =
    key.startsWith("ctrl+") ? "ctrl"
    : key.startsWith("alt+")  ? "alt"
    : key === "Enter"         ? "submit"
    : key === "Backspace"     ? "delete"
    : "char";

  metrics.increment(`tui.key.${category}`);
});
```

---

## Render Latency Display Component

Expose render timing to the user as a compact overlay:

```typescript
class FrameCounter {
  private fps = 0;
  update(fps: number) { this.fps = fps; }
  render(width: number): string[] {
    const label = `FPS: ${this.fps.toFixed(1)}`.padStart(width);
    return [`\x1b[2m${label}\x1b[0m`]; // dim
  }
}
```

---

## Suggested Metrics

| Metric | Description |
|--------|-------------|
| `tui.render.duration_ms` | Time to produce and write one frame |
| `tui.render.lines_changed` | Lines repainted this frame (diff quality) |
| `tui.key.submit` | User submits (Enter) — triggers an agent call |
| `tui.key.ctrl` | Ctrl-key combos (abort, paste, etc.) |
| `tui.component.count` | Number of active components |

---

**What's Next:** [Design Patterns](./design-patterns.md) | [Terminology](./terminology.md)
