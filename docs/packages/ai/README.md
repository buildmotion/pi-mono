# @mariozechner/pi-ai — Package Documentation

`pi-ai` is the foundation layer of pi-mono. It provides a single, unified streaming API that works across 20+ LLM providers. Every higher-level package in this monorepo calls into `pi-ai` to talk to language models.

---

## What you will learn

By reading these docs you will understand:

- How to make a streaming LLM call with a single function (`streamSimple`).
- How the event protocol works and how to consume it incrementally.
- How providers, APIs, and models relate to each other.
- How to authenticate with API keys, environment variables, and OAuth flows.
- How to query and extend the model registry.
- How to hook into requests for logging and payload inspection.
- How to register a custom provider or model.

---

## Prerequisites

- Node.js 20+ (or Bun 1+).
- A working API key for at least one supported provider (e.g. `ANTHROPIC_API_KEY`).
- Familiarity with TypeScript and `async`/`await`.

---

## Installation

```bash
npm install @mariozechner/pi-ai
```

---

## Quick start

```typescript
import { streamSimple } from "@mariozechner/pi-ai";
import { getModel } from "@mariozechner/pi-ai/models";

const model = getModel("anthropic", "claude-opus-4-5");
const stream = streamSimple(model, { messages: [{ role: "user", content: "Hello!", timestamp: Date.now() }] });

for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const message = await stream.result();
console.log("\nTotal tokens:", message.usage.totalTokens);
```

Set `ANTHROPIC_API_KEY` in your environment before running. Swap `getModel("anthropic", ...)` for any other provider/model pair from the model registry.

---

## Documentation index

| File | Description |
|---|---|
| [c4-01-context.md](./c4-01-context.md) | C4 Context diagram — what pi-ai is and who uses it |
| [c4-02-container.md](./c4-02-container.md) | C4 Container view — major sub-systems inside the package |
| [c4-03-component.md](./c4-03-component.md) | C4 Component view — key modules and classes |
| [c4-04-code-walkthrough.md](./c4-04-code-walkthrough.md) | Annotated walkthrough of a `streamSimple()` call |
| [features/streaming.md](./features/streaming.md) | End-to-end guide to the streaming API |
| [features/provider-auth.md](./features/provider-auth.md) | Authentication — API keys, env vars, OAuth |
| [features/model-registry.md](./features/model-registry.md) | Model registry — lookup, custom models, cost calculation |
| [extension-points.md](./extension-points.md) | How to register custom providers and models |
| [observability.md](./observability.md) | Logging, payload inspection, usage tracking |
| [design-patterns.md](./design-patterns.md) | Design patterns used in the package |
| [terminology.md](./terminology.md) | Glossary of pi-ai–specific terms |

See also the monorepo [glossary](../../glossary.md) for cross-package terminology.
