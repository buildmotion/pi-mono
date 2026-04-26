# Extension Points — pi-tui

pi-tui is designed to be composed rather than configured. Every visible element is a component; every interaction is a keybinding. This page shows you exactly where to plug in custom behavior.

---

## 1. Creating a Custom Component

Any object with a `render(width: number, height: number): string[]` method can be added to the Tui container.

```typescript
import { Tui } from "@mariozechner/pi-tui";

class StatusBar {
  public text = "";

  render(width: number, _height: number): string[] {
    // Return exactly one padded line
    const padded = this.text.padEnd(width, " ");
    return [`\x1b[7m${padded}\x1b[0m`]; // reverse video
  }
}

const tui = new Tui();
const bar = new StatusBar();
bar.text = " Ready";
tui.add(bar);
tui.start();
```

Rules:
- Return exactly as many lines as your component needs (the Tui layout stacks components top-to-bottom).
- Use ANSI escape codes for color/style.
- Avoid side effects inside `render()`; mutate state in event handlers instead.

---

## 2. Custom Keybindings

Keybindings are configured via a `keybindings` object passed to the editor or the Tui. Each entry maps a key sequence string to an action name.

```typescript
import { EditorComponent } from "@mariozechner/pi-tui";

const editor = new EditorComponent();
editor.keybindings = {
  ...editor.keybindings,          // keep defaults
  "ctrl+g": "clear",              // clear the editor
  "ctrl+shift+u": "uppercase",    // custom action name
};

editor.on("action", (name: string) => {
  if (name === "uppercase") {
    editor.value = editor.value.toUpperCase();
  }
});
```

---

## 3. Theming via ANSI Color Codes

pi-tui has no CSS layer. Colors are plain ANSI escape codes. To theme built-in components, subclass them and override the color constants:

```typescript
import { Markdown } from "@mariozechner/pi-tui";

class ThemedMarkdown extends Markdown {
  protected headingColor = "\x1b[35m"; // magenta headings
  protected codeColor    = "\x1b[32m"; // green code
}
```

Or set environment-level theme variables if the component reads them.

---

## 4. Replacing the Markdown Renderer

If the built-in Markdown component does not meet your needs, swap it out entirely:

```typescript
class MyMarkdown {
  public text = "";
  render(width: number, _height: number): string[] {
    // Your own ANSI Markdown renderer
    return myRender(this.text, width);
  }
}
```

Because the Tui only calls `render()`, any object that satisfies the interface is valid.

---

## 5. Adding Components Dynamically

Components can be added and removed at runtime:

```typescript
const loader = new Loader();
tui.add(loader);           // show spinner

await longOperation();

tui.remove(loader);        // hide spinner
tui.render();
```

---

**What's Next:** [Observability](./observability.md) | [Design Patterns](./design-patterns.md)
