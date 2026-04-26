# Feature: ChatPanel Setup and Configuration

## Learning Objectives

- Instantiate `ChatPanel` and bind it to an `Agent`.
- Configure the model, API key, and system prompt.
- Listen to agent events to drive custom UI logic.
- Understand how the panel handles streaming and tool execution.

---

## Background

`ChatPanel` is a [custom element](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements) that registers as `<pi-chat-panel>`. Once mounted, it renders a complete chat UI. You connect it to an `Agent` by setting the `.agent` property; ChatPanel then manages the full event loop and rendering.

---

## Step 1 — Install and Register

```bash
npm install @mariozechner/pi-web-ui @mariozechner/pi-agent-core @mariozechner/pi-ai
```

```typescript
// Import the package — this auto-registers <pi-chat-panel>
import "@mariozechner/pi-web-ui";
```

```html
<!-- index.html -->
<pi-chat-panel></pi-chat-panel>
```

---

## Step 2 — Create an Agent and Attach It

```typescript
import "@mariozechner/pi-web-ui";
import { Agent } from "@mariozechner/pi-agent-core";
import { models } from "@mariozechner/pi-ai";

const model = models.find(m => m.id === "claude-3-5-sonnet-20241022")!;

const agent = new Agent({
  initialState: {
    model,
    systemPrompt: "You are a helpful assistant. Be concise.",
  },
});

const panel = document.querySelector("pi-chat-panel") as any;
panel.agent = agent;
```

The panel immediately becomes interactive.

---

## Step 3 — Set an API Key Programmatically

If you want to skip the in-UI API key dialog and supply a key from your own backend:

```typescript
panel.agent.getApiKey = async (provider: string) => {
  if (provider === "anthropic") return "sk-ant-...";
  return undefined;
};
```

---

## Step 4 — Listen to Agent Events

Subscribe before calling `agent.prompt()` to receive events:

```typescript
agent.subscribe((event) => {
  if (event.type === "agent_end") {
    console.log("Run complete. Messages:", agent.state.messages.length);
  }
  if (event.type === "tool_execution_end") {
    console.log(`Tool ${event.toolName} finished`, event.result);
  }
});
```

---

## Step 5 — Add a Custom Tool

```typescript
import { Type } from "typebox";

agent.state.tools = [
  ...agent.state.tools,
  {
    name: "get_weather",
    label: "Get Weather",
    description: "Returns the current weather for a city.",
    parameters: Type.Object({ city: Type.String({ description: "City name" }) }),
    execute: async (_id, { city }) => ({
      content: [{ type: "text", text: `Weather in ${city}: sunny, 22°C` }],
      details: {},
    }),
  },
];
```

---

## Step 6 — Disable Auto-Session Saving (Optional)

By default ChatPanel persists the conversation to IndexedDB. To disable:

```typescript
panel.persistSessions = false;
```

---

## Check-Yourself Questions

1. What does importing `@mariozechner/pi-web-ui` do without calling any function?
2. Where does the API key get resolved if you do NOT set `getApiKey`?
3. What property on ChatPanel holds the bound Agent instance?
4. How would you pre-populate the chat with an existing conversation history?
5. What event type signals that the agent has finished all tool calls and is idle?

---

**What's Next:** [Storage](./storage.md) | [Artifacts](./artifacts.md)
