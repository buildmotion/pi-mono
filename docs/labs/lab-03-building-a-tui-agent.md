# Lab 03: Building a TUI-Based AI Assistant with `pi-tui` and `pi-agent-core`

Terminal UIs are not relics — they are fast, keyboard-driven, SSH-friendly, and require zero frontend tooling. In this lab you build a minimal interactive AI assistant that runs entirely in a terminal: the user types a prompt, presses Enter, and watches the model's reply stream in as rendered Markdown.

## Learning Objectives

- Create a `Tui` instance and understand the render loop
- Add a `Markdown` component for live-updating streamed output
- Add an `Editor` component for user input
- Connect user input to `agent.prompt()` via event subscription
- Drive UI updates from `AgentEvent` callbacks
- Add a loading indicator that reflects agent state

## Prerequisites

- Labs 01 and 02 complete — you should understand `Agent`, tools, and `AgentEvent`
- Linux or macOS terminal (Windows requires WSL2)
- Node.js 20 or later
- A terminal emulator that supports 256 colors and Unicode (iTerm2, GNOME Terminal, Windows Terminal in WSL)

```bash
mkdir lab-03 && cd lab-03
npm init -y
npm install @mariozechner/pi-ai @mariozechner/pi-agent-core @mariozechner/pi-tui @sinclair/typebox tsx typescript
```

---

## Background

### How TUI Frameworks Work

A TUI framework owns the terminal: it puts the terminal into raw mode, intercepts every keystroke, and controls where the cursor sits. The render loop works like a game engine:

1. State changes (a new text chunk arrives, the user presses a key)
2. Each component re-computes its output lines
3. A differential repaint writes only the lines that changed

`pi-tui` uses this model. Components are objects with a `render()` method that returns an array of strings. The `Tui` orchestrator calls `render()` on every component each cycle and emits only the diff. This keeps the terminal flicker-free even when the LLM is streaming hundreds of tokens per second.

### Synchronized Rendering

Because rendering is synchronous and driven by state, you never need to worry about interleaved writes. You update state, the render loop picks it up on the next tick. The agent's async event callbacks write into state objects; the TUI reads from them.

---

## Step 1: Install `pi-tui` and Create a `Tui` Instance

Create `tui.ts`:

```typescript
import { Tui } from "@mariozechner/pi-tui";

const tui = new Tui({
  // The Tui takes over the full terminal.
  // Set fps to 30 for a smooth streaming experience without burning CPU.
  fps: 30,
});

// Start the render loop — this does not block; it runs in the background
tui.start();

// Give the render loop a moment to initialize before adding components
await new Promise((r) => setTimeout(r, 50));

console.error("TUI started"); // console.error goes to stderr, visible even in raw mode
```

> **Note:** While the TUI is active, `console.log` writes to the managed screen buffer. Use `console.error` for debugging output that should bypass the TUI.

Export `tui` and import it in subsequent files throughout this lab.

---

## Step 2: Add a Markdown Component for Displaying Messages

`pi-tui` includes a `Markdown` component that renders Markdown syntax — bold, code blocks, lists — as ANSI-escaped terminal output. This is what makes streamed LLM output readable as it arrives.

```typescript
import { Tui, Markdown } from "@mariozechner/pi-tui";

const tui = new Tui({ fps: 30 });
tui.start();
await new Promise((r) => setTimeout(r, 50));

// State object that the Markdown component reads from
const assistantState = {
  content: "_Waiting for your first message..._",
};

const messageDisplay = new Markdown({
  // The Markdown component calls this function each render cycle
  getText: () => assistantState.content,
  // Max width in columns; 0 = auto-fit to terminal width
  maxWidth: 0,
});

tui.addComponent(messageDisplay);
```

Test it by mutating `assistantState.content` from a `setInterval` and watching the terminal update live:

```typescript
let tick = 0;
const interval = setInterval(() => {
  assistantState.content = `**Tick ${tick++}**: _Hello from the render loop_\n\n\`some code\``;
  if (tick > 5) {
    clearInterval(interval);
    tui.stop();
    process.exit(0);
  }
}, 500);
```

---

## Step 3: Add an Editor Component for User Input

The `Editor` component is a single-line or multi-line text input field. It captures keystrokes in raw mode, supports basic editing (backspace, arrows, Ctrl+A/E), and emits a `submit` event when the user presses Enter.

```typescript
import { Tui, Markdown, Editor } from "@mariozechner/pi-tui";

const tui = new Tui({ fps: 30 });
tui.start();
await new Promise((r) => setTimeout(r, 50));

const assistantState = { content: "_Ask me anything._" };

const messageDisplay = new Markdown({
  getText: () => assistantState.content,
  maxWidth: 0,
});

const input = new Editor({
  placeholder: "Type a message and press Enter...",
  // Prefix shown to the left of the cursor line
  prefix: "> ",
});

tui.addComponent(messageDisplay);
tui.addComponent(input);

// Test: echo the submitted text back as Markdown
input.on("submit", (text: string) => {
  assistantState.content = `**You said:** ${text}`;
  input.clear();
});
```

Run this and verify:
- The editor appears below the message display
- Typing updates the editor field
- Pressing Enter triggers `submit`, clears the field, and updates the Markdown above

---

## Step 4: Wire the Editor Submit Event to `agent.prompt()`

Now connect the input to a real agent. Create `index.ts`:

```typescript
import { models } from "@mariozechner/pi-ai";
import { Agent } from "@mariozechner/pi-agent-core";
import { Tui, Markdown, Editor } from "@mariozechner/pi-tui";

const model = models.find(
  (m) => m.provider === "anthropic" && m.id.includes("claude-3-5")
);
if (!model) throw new Error("Model not found");

const agent = new Agent({
  model,
  systemPrompt: "You are a helpful assistant. Format responses in Markdown when appropriate.",
});

const tui = new Tui({ fps: 30 });
tui.start();
await new Promise((r) => setTimeout(r, 50));

const assistantState = { content: "_Ask me anything. Press Ctrl+C to quit._" };

const messageDisplay = new Markdown({
  getText: () => assistantState.content,
  maxWidth: 0,
});

const input = new Editor({
  placeholder: "Type a message and press Enter...",
  prefix: "> ",
});

tui.addComponent(messageDisplay);
tui.addComponent(input);

let isStreaming = false;

input.on("submit", async (text: string) => {
  if (isStreaming || !text.trim()) return;

  isStreaming = true;
  input.setEnabled(false);
  input.clear();

  // Show the user's message immediately while we wait for the model
  assistantState.content = `**You:** ${text}\n\n_Thinking..._`;

  await agent.prompt(text);

  input.setEnabled(true);
  isStreaming = false;
});

// Graceful exit
process.on("SIGINT", () => {
  tui.stop();
  process.exit(0);
});
```

At this point the submit handler calls `agent.prompt()` but does not yet show the streaming output. That comes in Step 5.

---

## Step 5: Subscribe to `message_update` Events and Re-Render

Subscribe to the agent's event stream and update `assistantState.content` as text arrives. Because the Markdown component reads from `assistantState` on every render cycle, the display will update automatically.

```typescript
agent.subscribe((event) => {
  switch (event.type) {
    case "agent_start":
      // Show the last user message above the streaming reply
      assistantState.content = `_Streaming..._`;
      break;

    case "message_update":
      // event.message.content is the full accumulated text so far
      if (event.message.content) {
        assistantState.content = event.message.content;
      }
      break;

    case "tool_execution_start":
      assistantState.content += `\n\n_Calling tool: **${event.toolCall.name}**..._`;
      break;

    case "agent_end":
      // Final message is already in assistantState.content from the last message_update
      // No action needed; the render loop will display it
      break;
  }
});
```

Add the subscription before the `input.on("submit", ...)` block. Run `npx tsx index.ts` and type a message. You should see Markdown-rendered text appear incrementally as the model streams.

---

## Step 6: Add a Loader Component That Shows While Streaming

`pi-tui` includes a `Loader` component that renders a spinning animation. Show it between the message display and the editor while `isStreaming` is true.

```typescript
import { Tui, Markdown, Editor, Loader } from "@mariozechner/pi-tui";

// ...existing setup...

const loader = new Loader({
  // The Loader calls this each render cycle to decide whether to display
  isVisible: () => isStreaming,
  label: "Generating response...",
});

// Order matters: message display → loader → editor (bottom)
tui.addComponent(messageDisplay);
tui.addComponent(loader);
tui.addComponent(input);
```

The loader will appear and disappear automatically based on `isStreaming`. No manual show/hide calls are needed — the state-driven render loop handles it.

Test: send a prompt and watch the spinner appear, then disappear when `agent_end` fires.

---

## Step 7 (Stretch): Add a Tool-Execution Banner

When the agent calls a tool, show a banner above the loader indicating which tool is running and what arguments it received.

```typescript
const bannerState = {
  visible: false,
  text: "",
};

const toolBanner = new Markdown({
  getText: () => (bannerState.visible ? bannerState.text : ""),
  maxWidth: 0,
});

// Insert between message display and loader
tui.addComponent(messageDisplay);
tui.addComponent(toolBanner);
tui.addComponent(loader);
tui.addComponent(input);
```

Update the agent subscription:

```typescript
case "tool_execution_start":
  bannerState.visible = true;
  bannerState.text = `> **Tool:** \`${event.toolCall.name}\`  \n> **Input:** \`${JSON.stringify(event.toolCall.input)}\``;
  break;

case "tool_execution_end":
  bannerState.visible = false;
  bannerState.text = "";
  break;
```

Add a tool to your agent (refer to Lab 02's `weatherTool`) and ask a question that triggers it. The banner should appear, show the tool name and arguments, then disappear when execution completes.

---

## Expected Outcomes

By the end of this lab you should have:

- A terminal application that renders Markdown-formatted LLM output live
- A text editor that submits prompts and clears on submit
- A loading indicator that appears exactly while the agent is working
- State-driven rendering with no manual DOM manipulation
- (Stretch) A tool-execution banner that surfaces tool activity to the user

---

## Check Yourself

1. The Markdown component calls `getText()` on every render cycle. What happens if `getText()` is expensive? How would you optimize it?
2. Why does the lab use `console.error` for debug output instead of `console.log`?
3. The editor is disabled (`input.setEnabled(false)`) while streaming. What user experience problem does this solve?
4. If two `message_update` events arrive within the same 33 ms render frame (at 30 fps), how many times does the Markdown component actually repaint?
5. The `Loader` component visibility is driven by a function `() => isStreaming`. Why is a function used instead of passing the boolean value directly?

---

## What's Next

- **Lab 04** — [Building a Web UI Agent](./lab-04-building-a-web-ui-agent.md): move the same agent into a browser with `pi-web-ui`
- **`@mariozechner/pi-tui` README** — full component catalogue, layout options, and keyboard event API
- **`packages/tui/src/`** — component source code; reading it is the fastest way to learn what each component can do
