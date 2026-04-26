# @mariozechner/pi-web-ui — Course Introduction

## What You Will Learn

- Embed a full AI chat interface in a browser app using a single web component.
- Bind a pi-agent-core `Agent` to `ChatPanel` and receive streaming responses.
- Persist chat sessions and API keys to IndexedDB.
- Handle file attachments (PDF, DOCX, images) and artifact rendering in a sandboxed iframe.
- Extend the UI with custom tools, themes, and internationalization.

## Prerequisites

- HTML, CSS, and JavaScript basics
- Familiarity with [pi-agent-core](../agent/README.md) and [pi-ai](../ai/README.md)
- A bundler (Vite recommended)

## Quick Start

```bash
npm install @mariozechner/pi-web-ui @mariozechner/pi-agent-core @mariozechner/pi-ai
```

```typescript
// main.ts
import "@mariozechner/pi-web-ui";          // registers <pi-chat-panel>
import { Agent } from "@mariozechner/pi-agent-core";
import { models } from "@mariozechner/pi-ai";

const agent = new Agent({
  initialState: {
    model: models.find(m => m.id === "claude-3-5-sonnet-20241022")!,
    systemPrompt: "You are a helpful assistant.",
  },
});

const panel = document.querySelector("pi-chat-panel") as any;
panel.agent = agent;
```

```html
<!-- index.html -->
<pi-chat-panel></pi-chat-panel>
```

That's it. The panel renders a full chat UI backed by your Agent.

## Documentation Map

| File | Topic |
|------|-------|
| [c4-01-context.md](./c4-01-context.md) | System context and trust boundaries |
| [c4-02-container.md](./c4-02-container.md) | Browser container view |
| [c4-03-component.md](./c4-03-component.md) | Internal component breakdown |
| [c4-04-code-walkthrough.md](./c4-04-code-walkthrough.md) | Annotated code trace |
| [features/chat-panel.md](./features/chat-panel.md) | ChatPanel setup and configuration |
| [features/storage.md](./features/storage.md) | IndexedDB sessions and API keys |
| [features/artifacts.md](./features/artifacts.md) | Sandboxed artifact rendering |
| [extension-points.md](./extension-points.md) | Custom tools, themes, i18n |
| [observability.md](./observability.md) | Browser-side telemetry |
| [design-patterns.md](./design-patterns.md) | Patterns and rationale |
| [terminology.md](./terminology.md) | Package-specific terms |
