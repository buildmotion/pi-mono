# Feature: Markdown Rendering and Inline Images

## Learning Objectives

- Render Markdown text inside a terminal using the pi-tui `Markdown` component.
- Display inline images in terminals that support the Kitty or iTerm2 image protocols.
- Understand fallback behavior for terminals that do not support images.

---

## Background

Terminal output is plain text with ANSI escape codes for color and cursor control. pi-tui ships two components that push this beyond plain text:

1. **`Markdown`** (`src/components/markdown.ts`) — converts a Markdown string into ANSI-styled terminal lines.
2. **`Image`** (`src/components/image.ts`) — encodes a raw image buffer using the Kitty or iTerm2 inline-image protocol and emits the escape sequence that causes a supporting terminal to display the image inline.

Both components implement the standard `render(width, height): string[]` interface.

---

## Markdown Component

### What It Renders

| Markdown element | Terminal output |
|-----------------|-----------------|
| `# Heading` | Bold + color highlight |
| `` `code` `` | Highlighted background span |
| ` ```block``` ` | Indented, dimmed code block |
| `**bold**` | ANSI bold |
| `_italic_` | ANSI dim or italic (terminal-dependent) |
| `- list item` | Bullet prefix |
| Horizontal rule (`---`) | Repeated `─` characters |

### Usage

```typescript
import { Tui, Markdown } from "@mariozechner/pi-tui";

const tui = new Tui();

const markdown = new Markdown();
markdown.text = "# Hello\n\nThis is **bold** and `code`.";

tui.add(markdown);
tui.start();
```

When `markdown.text` changes, call `tui.render()` (or it is called automatically on the next render tick) to repaint just the changed lines.

### Width Wrapping

The `Markdown` component word-wraps at the `width` passed by the Tui layout engine. Long code blocks scroll horizontally rather than wrapping, preserving indentation.

---

## Inline Image Display

### Supported Protocols

| Protocol | Terminal | Detection |
|----------|----------|-----------|
| Kitty | Kitty, WezTerm, Ghostty | `$TERM` contains `kitty` or capabilities query |
| iTerm2 | iTerm2, Hyper | `$TERM_PROGRAM === "iTerm.app"` |

Source: `src/terminal-image.ts`

### Image Component Usage

```typescript
import { Tui, Image } from "@mariozechner/pi-tui";
import { readFileSync } from "fs";

const tui = new Tui();
const img = new Image();

// Raw PNG/JPEG bytes
img.data = readFileSync("screenshot.png");
img.mimeType = "image/png";
img.maxWidth = 80;  // columns
img.maxHeight = 20; // rows

tui.add(img);
tui.start();
```

### Fallback Behavior

When the terminal does not support inline images:
- The `Image` component renders an empty set of lines (no placeholder text by default).
- You can detect support via `terminal.supportsImages()` and conditionally render a text description instead.

```typescript
import { terminal } from "@mariozechner/pi-tui";

if (!terminal.supportsImages()) {
  markdownComponent.text += "\n_(image not supported in this terminal)_";
}
```

---

## Check-Yourself Questions

1. What interface must every pi-tui component implement?
2. How does the `Markdown` component handle code blocks differently from inline code?
3. Name the two inline image protocols supported by pi-tui.
4. What happens if you display an `Image` component in a terminal that does not support either protocol?
5. How would you dynamically update a `Markdown` component with streaming LLM output?

---

**What's Next:** [Editor Component](./editor-component.md) | [Rendering Deep Dive](./rendering.md)
