# @mariozechner/pi-agent-core — Package Documentation

`pi-agent-core` is the stateful agent engine of pi-mono. It wraps the single-call streaming API of `pi-ai` in a persistent `Agent` class that manages conversation history, drives the tool-call loop, and emits a typed event stream for UI frameworks.

---

## What you will learn

By reading these docs you will understand:

- How the `Agent` class manages state across multiple LLM turns.
- How the agent loop works — from prompt to tool execution to final response.
- How to define tools with TypeBox schemas and async `execute` functions.
- How to intercept tool calls before and after execution with hooks.
- How to inject messages mid-run (steering) and queue messages for after a run (follow-up).
- How to subscribe to `AgentEvent` for real-time UI updates and structured logging.
- How to extend `AgentMessage` with custom message types via declaration merging.
- How to route LLM calls through a backend proxy for browser deployments.
- Which design patterns the package uses and why.

---

## Prerequisites

- Node.js 20+.
- `@mariozechner/pi-ai` installed and configured with at least one provider API key.
- Familiarity with TypeScript, `async`/`await`, and basic LLM concepts (prompts, tokens, tool calls).

---

## Installation

```bash
npm install @mariozechner/pi-agent-core @mariozechner/pi-ai
```

---

## Quick start

```typescript
import { Agent } from "@mariozechner/pi-agent-core";
import { getModel, Type } from "@mariozechner/pi-ai";

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model: getModel("anthropic", "claude-haiku-4-5"),
    thinkingLevel: "off",
    tools: [],
    messages: [],
  },
});

// Subscribe to events for real-time output
agent.subscribe((event) => {
  if (event.type === "message_update") {
    const e = event.assistantMessageEvent;
    if (e.type === "text_delta") process.stdout.write(e.delta);
  }
  if (event.type === "agent_end") console.log("\nDone.");
});

// Send a prompt and wait for the run to complete
await agent.prompt("What is the capital of France?");
```

Set `ANTHROPIC_API_KEY` in your environment before running. Swap `getModel("anthropic", ...)` for any other provider/model pair from the model registry.

---

## Documentation index

| File | Description |
|---|---|
| [c4-01-context.md](./c4-01-context.md) | C4 Context diagram — what pi-agent-core is and who uses it |
| [c4-02-container.md](./c4-02-container.md) | C4 Container view — major sub-systems inside the package |
| [c4-03-component.md](./c4-03-component.md) | C4 Component view — key modules and their responsibilities |
| [c4-04-code-walkthrough.md](./c4-04-code-walkthrough.md) | Annotated walkthrough of a full `agent.prompt()` call |
| [features/agent-loop.md](./features/agent-loop.md) | End-to-end guide to the agent loop state machine |
| [features/tool-execution.md](./features/tool-execution.md) | Tool definition, parallel vs sequential execution, hooks |
| [features/steering-and-followup.md](./features/steering-and-followup.md) | Steering and follow-up queues — mid-run injection |
| [extension-points.md](./extension-points.md) | All extension points: convertToLlm, transformContext, hooks, etc. |
| [observability.md](./observability.md) | Subscribing to AgentEvent for logging, tracing, and metrics |
| [design-patterns.md](./design-patterns.md) | Design patterns used in the package |
| [terminology.md](./terminology.md) | Glossary of pi-agent-core–specific terms |

See also the monorepo [glossary](../../glossary.md) and the [`pi-ai` package docs](../ai/README.md) for lower-level streaming concepts.
