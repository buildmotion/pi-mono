# Lab 04: Embedding an AI Chat Interface in a Browser App with `pi-web-ui`

The same agent loop from Labs 02 and 03 runs in a browser with zero rearchitecting. `pi-web-ui` exposes a `ChatPanel` web component that bundles the UI, session storage, and API-key management. You drop one HTML tag into a page and get a fully functional chat interface.

## Learning Objectives

- Register and embed the `ChatPanel` web component in an HTML page
- Configure a model and API key through the settings dialog
- Attach a custom `AgentTool` (a JavaScript REPL) to the panel
- Observe how PDF and image attachments flow through the context
- Theme the component with CSS custom properties

## Prerequisites

- Basic web development familiarity (HTML, ES modules, a bundler)
- Node.js 20 or later
- [Vite](https://vitejs.dev/) installed globally or as a dev dependency
- An API key for at least one provider

```bash
mkdir lab-04 && cd lab-04
npm init -y
npm install @mariozechner/pi-web-ui @mariozechner/pi-agent-core @sinclair/typebox
npm install --save-dev vite typescript
```

---

## Background

### Web Components

`ChatPanel` is a [Custom Element](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements) — a native browser API that lets you define new HTML tags. You do not need React, Vue, or any other framework. The component registers itself via `customElements.define()` and the browser treats `<pi-chat-panel>` like a built-in tag.

`pi-web-ui` uses a minimal reactive library (`mini-lit`) internally. From your application's perspective, the component is a black box with a DOM API.

### IndexedDB Storage

The panel stores conversation sessions, API keys, and user settings in the browser's [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API). This means:
- Sessions survive page reloads
- API keys are stored locally — they never leave the browser
- There is no backend required for persistence

### CORS Considerations

When the ChatPanel makes requests to LLM provider APIs, those requests go directly from the browser to the provider. Most providers (OpenAI, Anthropic, Google) support CORS for browser requests, but some do not. If you self-host a model endpoint (Lab 06), ensure your server returns the correct `Access-Control-Allow-Origin` headers.

---

## Step 1: Install `pi-web-ui` and Peer Dependencies

The package ships pre-bundled components as ES modules. Vite will handle bundling and hot-reload for development.

```bash
# Verify installation
node -e "require('@mariozechner/pi-web-ui'); console.log('OK')"
```

Create a `vite.config.ts`:

```typescript
import { defineConfig } from "vite";

export default defineConfig({
  server: {
    port: 5173,
    open: true,
  },
});
```

---

## Step 2: Register the ChatPanel Component

The component is registered by importing the package. Create `src/main.ts`:

```typescript
// Importing this module registers the <pi-chat-panel> custom element globally.
import "@mariozechner/pi-web-ui";

// Confirm registration
console.log(
  "ChatPanel registered:",
  customElements.get("pi-chat-panel") !== undefined
);
```

The import has a side effect: it calls `customElements.define("pi-chat-panel", ChatPanel)`. After this import completes, every `<pi-chat-panel>` tag in the document is upgraded to a live component.

---

## Step 3: Embed `<pi-chat-panel>` in an HTML Page

Create `index.html` at the project root:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Lab 04 — pi-web-ui</title>
    <style>
      * {
        box-sizing: border-box;
        margin: 0;
        padding: 0;
      }

      html,
      body {
        height: 100%;
        font-family: system-ui, sans-serif;
        background: #0f0f0f;
      }

      pi-chat-panel {
        display: block;
        height: 100vh;
        width: 100%;
      }
    </style>
  </head>
  <body>
    <pi-chat-panel></pi-chat-panel>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

Start Vite:

```bash
npx vite
```

Open `http://localhost:5173`. You should see the ChatPanel UI. It will prompt you to configure a model and API key before the first message.

---

## Step 4: Configure a Model and API Key via the Settings Dialog

1. Click the **Settings** icon (gear or cogwheel) in the panel header
2. Select your provider (e.g., Anthropic)
3. Paste your API key into the key field
4. Choose a model from the dropdown (e.g., `claude-3-5-sonnet-20241022`)
5. Click **Save**

The API key is written to IndexedDB. Open DevTools → Application → IndexedDB → `pi-web-ui` to inspect the stored value. Note that the key is stored in plain text — this is acceptable for local development but not for shared or production deployments.

Send a test message: `Hello, are you working?`

You should see the response stream into the panel in real time.

**To set the model programmatically** (useful for testing), use the component's JavaScript API:

```typescript
import "@mariozechner/pi-web-ui";
import { models } from "@mariozechner/pi-ai";

const panel = document.querySelector("pi-chat-panel");
if (!panel) throw new Error("Panel not found");

const model = models.find(
  (m) => m.provider === "anthropic" && m.id.includes("claude-3-5")
);

if (model) {
  // Type assertion needed because the element type is not automatically inferred
  (panel as any).setModel(model);
}
```

---

## Step 5: Attach a Custom AgentTool — a JavaScript REPL

`ChatPanel` accepts an array of `AgentTool` objects. The model can call these tools during a conversation, and the results appear inline in the chat.

Create a JavaScript REPL tool that evaluates code in a sandboxed context:

```typescript
import "@mariozechner/pi-web-ui";
import { Type } from "@sinclair/typebox";
import type { AgentTool } from "@mariozechner/pi-agent-core";

const replTool: AgentTool = {
  name: "javascript_repl",
  label: "JavaScript REPL",
  description:
    "Evaluates a JavaScript expression or statement and returns the result. " +
    "Use this to perform calculations, transform data, or demonstrate code. " +
    "The code runs in a restricted context — no DOM access, no network calls.",
  parameters: Type.Object({
    code: Type.String({
      description: "The JavaScript code to evaluate",
    }),
  }),
  execute: async (_id, params) => {
    let result: string;
    try {
      // eslint-disable-next-line no-new-func
      const fn = new Function(`"use strict"; return (${params.code})`);
      const raw = fn();
      result = typeof raw === "object" ? JSON.stringify(raw, null, 2) : String(raw);
    } catch (err) {
      result = `Error: ${err instanceof Error ? err.message : String(err)}`;
    }

    return {
      content: [{ type: "text", text: result }],
      details: { code: params.code, result },
    };
  },
};

// Attach tools to the panel after DOM is ready
document.addEventListener("DOMContentLoaded", () => {
  const panel = document.querySelector("pi-chat-panel") as any;
  if (panel) panel.setTools([replTool]);
});
```

Test the tool by sending: `"What is 2^32? Please use the JavaScript REPL to calculate it."`

The model should call `javascript_repl` with `code: "Math.pow(2, 32)"`, and the result should appear in the chat as a tool-call card.

> **Security note:** `new Function()` is not sandboxed from the rest of the page's JavaScript context. For production use, execute untrusted code inside a sandboxed `<iframe>` with no `src` and a strict Content Security Policy.

---

## Step 6: Upload a PDF Attachment and Observe Context Injection

The ChatPanel has a file attachment button (paperclip icon). Supported formats include PDF, DOCX, plain text, and images (PNG, JPEG, WebP).

1. Download any short PDF (for example, a one-page research abstract)
2. Click the attachment icon and select the file
3. Send the message: `"Please summarize this document in three bullet points."`

Observe what happens in DevTools → Network. The ChatPanel extracts the document's text content client-side and injects it as a `UserMessage` content block before your prompt. The model never receives the raw binary — only the extracted text.

For image files, the raw image bytes are base64-encoded and sent as a vision content block, provided the selected model supports vision (most Claude, GPT-4o, and Gemini models do).

**Inspect the raw request:**

```typescript
// Intercept fetch to log the request body
const originalFetch = window.fetch;
window.fetch = async (input, init) => {
  if (typeof input === "string" && input.includes("anthropic")) {
    const body = JSON.parse(init?.body as string);
    console.log("LLM request:", JSON.stringify(body.messages, null, 2));
  }
  return originalFetch(input, init);
};
```

Look at the `messages` array. You will see the document content as a text block and your prompt as a separate text block in the same user message.

---

## Step 7 (Stretch): Theme the ChatPanel with CSS Custom Properties

The ChatPanel exposes its color scheme through CSS custom properties on the `:host` element. Override them from your page's stylesheet:

```css
pi-chat-panel {
  /* Background colors */
  --pi-bg-primary: #1a1a2e;
  --pi-bg-secondary: #16213e;
  --pi-bg-input: #0f3460;

  /* Text colors */
  --pi-text-primary: #e0e0e0;
  --pi-text-secondary: #a0a0b0;
  --pi-text-accent: #e94560;

  /* Accent and border */
  --pi-accent: #e94560;
  --pi-border: #2a2a4a;

  /* Typography */
  --pi-font-size-base: 15px;
  --pi-font-family-mono: "JetBrains Mono", "Fira Code", monospace;
}
```

To discover all available custom properties, inspect the component in DevTools and look for `var(--pi-*)` references in the Shadow DOM styles.

For a dark-mode / light-mode toggle, add a class to the document and define two property sets:

```css
pi-chat-panel {
  --pi-bg-primary: #ffffff;
  --pi-text-primary: #1a1a1a;
}

.dark pi-chat-panel {
  --pi-bg-primary: #1a1a2e;
  --pi-text-primary: #e0e0e0;
}
```

---

## Expected Outcomes

By the end of this lab you should have:

- A running Vite project with `<pi-chat-panel>` embedded and functional
- A configured model and API key stored in IndexedDB
- A custom JavaScript REPL tool that the model can invoke mid-conversation
- An understanding of how file attachments inject content into the LLM context
- (Stretch) A custom color theme applied via CSS custom properties

---

## Check Yourself

1. Open DevTools → Application → IndexedDB and locate the stored API key. What are the security implications of this storage strategy for a production application?
2. Why does `inputTokens` jump significantly when you attach a PDF file? What is happening to your context?
3. The ChatPanel sends LLM requests directly from the browser. What CORS headers must an OpenAI-compatible self-hosted endpoint return to allow this?
4. Your `javascript_repl` tool uses `new Function()`. Name two attacks a malicious prompt could enable through this vector.
5. A user uploads a 50-page PDF. The selected model has a 128k token context window. What happens if the document alone exceeds the context window?

---

## What's Next

- **Lab 05** — [Deploying a Slack Bot with pi-mom](./lab-05-long-running-bot-mom.md): run an agent that lives in Slack and manages its own tools
- **`@mariozechner/pi-web-ui` README** — full component API, theming reference, and session management
- **`packages/web-ui/src/`** — component source; the `ChatPanel` class is the entry point
