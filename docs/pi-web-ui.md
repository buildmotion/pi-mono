# `@mariozechner/pi-web-ui` — Browser Chat UI Components

## What is it?

`pi-web-ui` is a TypeScript library of web components (built with `mini-lit` and Tailwind CSS v4) that provide a production-ready AI chat interface for browser applications. It wraps `pi-agent-core`'s `Agent` in a full chat UI complete with streaming responses, tool-call cards, file attachments, sandboxed artifact execution, session storage, API key management, and model selection — all composable and themeable.

**npm package:** `@mariozechner/pi-web-ui`  
**Version:** 0.70.2  
**License:** MIT  
**Real-world use:** [sitegeist.ai](https://sitegeist.ai) browser extension

---

## The Problem It Solves

Embedding an AI chat interface in a web application is not just about streaming text. A production chat UI needs:

- Message history with streamed assistant responses
- Tool-call progress cards (so users see what the model is doing)
- File attachment ingestion (PDF, DOCX, XLSX, images)
- Sandboxed execution of model-generated HTML, SVG, and Markdown artifacts
- Session persistence (IndexedDB)
- API key management across providers
- Model and provider selection UI
- CORS proxy handling for browser-to-LLM calls

Building all of this from scratch takes weeks. `pi-web-ui` ships it as composable components.

---

## Audience

- Web developers adding an AI assistant to a product
- Teams building standalone browser AI tools (document analysis, code review, data exploration)
- Browser extension developers
- Anyone who wants to use `pi-agent-core` in a browser context

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    ChatPanel                        │
│  ┌─────────────────────┐  ┌─────────────────────┐   │
│  │   AgentInterface    │  │   ArtifactsPanel    │   │
│  │  (messages, input)  │  │  (HTML, SVG, MD…)   │   │
│  └─────────────────────┘  └─────────────────────┘   │
└─────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│           Agent (from pi-agent-core)                │
│  - Messages, model, tools, event stream             │
└─────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│                  AppStorage                         │
│  SettingsStore  ProviderKeysStore  SessionsStore    │
│              IndexedDBStorageBackend                │
└─────────────────────────────────────────────────────┘
```

The `ChatPanel` is the top-level component most consumers use. It wires an `Agent` to two sub-panels:
- `AgentInterface` — message history, text input, model/thinking selectors
- `ArtifactsPanel` — sandboxed rendering of model-generated artifacts

---

## Key Components

### ChatPanel

The all-in-one entry point. Give it an `Agent` and it handles the rest.

```typescript
const chatPanel = new ChatPanel();
await chatPanel.setAgent(agent, {
  onApiKeyRequired: (provider) => ApiKeyPromptDialog.prompt(provider),
  toolsFactory: (agent, agentInterface, artifactsPanel, runtimeProvidersFactory) => {
    const replTool = createJavaScriptReplTool();
    replTool.runtimeProvidersFactory = runtimeProvidersFactory;
    return [replTool];
  },
});
document.body.appendChild(chatPanel);
```

### AgentInterface

Lower-level chat UI for custom layouts. Exposes individual control flags:

```typescript
const chat = document.createElement('agent-interface') as AgentInterface;
chat.session = agent;
chat.enableAttachments = true;
chat.enableModelSelector = true;
chat.enableThinkingSelector = true;
chat.showThemeToggle = false;
```

### ArtifactsPanel

Displays model-generated artifacts in sandboxed iframes:

- **HTML** — interactive pages (charts, games, forms)
- **SVG** — vector graphics
- **Markdown** — rendered documentation
- **Images, PDF, DOCX, XLSX** — file previews

Artifacts are created/updated/deleted by the model via a built-in tool that `ArtifactsPanel` exposes as `artifactsPanel.tool`.

---

## Message Types

`pi-web-ui` extends `pi-agent-core`'s message system with two additional types:

### UserMessageWithAttachments

User messages that include file attachments (processed and base64-encoded by `loadAttachment()`):

```typescript
const message: UserMessageWithAttachments = {
  role: 'user-with-attachments',
  content: 'Analyze this document',
  attachments: [await loadAttachment(file)],
  timestamp: Date.now(),
};
```

### ArtifactMessage

Represents artifact state changes for session persistence:

```typescript
const artifact: ArtifactMessage = {
  role: 'artifact',
  action: 'create',    // 'create' | 'update' | 'delete'
  filename: 'chart.html',
  content: '<canvas id="c">…</canvas>',
  timestamp: new Date().toISOString(),
};
```

Artifact messages are filtered out by `defaultConvertToLlm` — the LLM never sees them; they exist only to replay artifact state when loading a saved session.

---

## Built-in Tools

### JavaScript REPL

Executes JavaScript in a sandboxed browser environment. The model can generate charts (using Chart.js or D3), manipulate attachments, or produce artifacts.

```typescript
import { createJavaScriptReplTool } from '@mariozechner/pi-web-ui';

const replTool = createJavaScriptReplTool();
replTool.runtimeProvidersFactory = () => [
  new AttachmentsRuntimeProvider(attachments),
  new ArtifactsRuntimeProvider(artifactsPanel, agent, true),
];
agent.state.tools = [replTool];
```

### Extract Document

Fetches a URL and extracts its text content. Handles CORS via a configurable proxy URL.

```typescript
import { createExtractDocumentTool } from '@mariozechner/pi-web-ui';

const extractTool = createExtractDocumentTool();
extractTool.corsProxyUrl = 'https://corsproxy.io/?';
```

### Custom Tool Renderers

Register a custom UI card for any tool result:

```typescript
import { registerToolRenderer } from '@mariozechner/pi-web-ui';

registerToolRenderer('my_tool', {
  render(params, result, isStreaming) {
    return {
      content: html`<div>Custom result view</div>`,
      isCustom: false,   // false = wrapped in standard card; true = no wrapper
    };
  },
});
```

---

## Storage Layer

`pi-web-ui` ships a complete IndexedDB-backed storage system with four stores:

| Store | Contents |
|-------|----------|
| `SettingsStore` | Key-value app settings (proxy URL, etc.) |
| `ProviderKeysStore` | API keys per provider name |
| `SessionsStore` | Full session data (messages, attachments, artifacts) |
| `CustomProvidersStore` | User-defined Ollama/vLLM/LM Studio endpoints |

Setup:

```typescript
import { AppStorage, IndexedDBStorageBackend, SettingsStore, … } from '@mariozechner/pi-web-ui';

const backend = new IndexedDBStorageBackend({
  dbName: 'my-app',
  version: 1,
  stores: [ settings.getConfig(), providerKeys.getConfig(), sessions.getConfig(), … ],
});
const storage = new AppStorage(settings, providerKeys, sessions, customProviders, backend);
setAppStorage(storage);
```

All stores are wired to the global `AppStorage` singleton accessed via `getAppStorage()`.

---

## CORS Proxy

Browser apps cannot directly call most LLM provider APIs (CORS restrictions). `pi-web-ui` includes:

- `createStreamFn()` — wraps `pi-ai`'s stream function with automatic proxy injection
- `shouldUseProxyForProvider()` — tells you whether a given provider needs proxying
- `isCorsError()` — detects CORS errors in catch blocks

The `AgentInterface` component auto-configures the proxy from `SettingsStore` when mounted.

---

## Dialogs

| Dialog | Purpose |
|--------|---------|
| `SettingsDialog` | Opens a tabbed settings overlay (providers, proxy, API keys) |
| `SessionListDialog` | Browse, load, and delete saved sessions |
| `ApiKeyPromptDialog` | Prompt the user to enter an API key for a given provider |
| `ModelSelector` | Searchable model picker |

---

## Attachments

`loadAttachment()` ingests files and produces a normalized `Attachment` object:

```typescript
interface Attachment {
  id: string;
  type: 'image' | 'document';
  fileName: string;
  mimeType: string;
  size: number;
  content: string;         // base64-encoded raw bytes
  extractedText?: string;  // text extraction for PDF/DOCX/XLSX/PPTX
  preview?: string;        // base64 preview thumbnail
}
```

Supported formats: PDF (via `pdfjs-dist`), DOCX (via `docx-preview`), XLSX (via SheetJS), PPTX (zip-parsed), images, plain text.

---

## Internationalization

```typescript
import { i18n, setLanguage, translations } from '@mariozechner/pi-web-ui';

translations.de = { 'Loading...': 'Laden…', 'No sessions yet': 'Noch keine Sitzungen' };
setLanguage('de');
console.log(i18n('Loading...')); // "Laden…"
```

---

## Source Layout

```
packages/web-ui/src/
├── index.ts               Public re-exports
├── ChatPanel.ts           Top-level ChatPanel web component
├── app.css                Tailwind entry point (compiled to dist/app.css)
├── components/
│   ├── AgentInterface.ts  Chat message list + input
│   ├── ArtifactsPanel.ts  Sandboxed artifact viewer
│   └── …                 Individual message renderers, tool cards
├── dialogs/
│   ├── SettingsDialog.ts
│   ├── SessionListDialog.ts
│   ├── ApiKeyPromptDialog.ts
│   └── ModelSelector.ts
├── storage/
│   ├── AppStorage.ts
│   ├── IndexedDBStorageBackend.ts
│   ├── SettingsStore.ts
│   ├── ProviderKeysStore.ts
│   ├── SessionsStore.ts
│   └── CustomProvidersStore.ts
├── tools/
│   ├── javascript-repl/   JS REPL tool and sandbox iframe
│   ├── extract-document/  URL text extraction tool
│   └── artifacts/         Artifact create/update/delete tool
├── prompts/               Built-in system prompt helpers
└── utils/                 CORS detection, stream wrappers, attachment loading
```

---

## Key Dependencies

| Dependency | Purpose |
|-----------|---------|
| `@mariozechner/mini-lit` (peer) | Lightweight Lit-based web components |
| `lit` (peer) | Web component base |
| `@mariozechner/pi-ai` | LLM provider access |
| `pdfjs-dist` | PDF text extraction and rendering |
| `docx-preview` | DOCX rendering |
| `jszip` | ZIP/PPTX parsing |
| `xlsx` (SheetJS) | Spreadsheet parsing |
| `ollama` | Ollama local provider discovery |
| `@lmstudio/sdk` | LM Studio local provider discovery |
| `lucide` | Icon set |

---

## Pros and Cons

**Pros**
- Production-ready out of the box (already used in sitegeist.ai)
- Handles the full chat stack: streaming, tools, attachments, artifacts, storage, CORS
- Custom provider support (Ollama, LM Studio, vLLM)
- Sandboxed artifact execution (HTML/JS never runs in main page context)
- Extensible: custom tool renderers, custom message types, custom providers
- Internationalization API

**Cons**
- Browser-only (not for Node.js/terminal use)
- Requires `mini-lit` and `lit` as peer dependencies
- `PersistentStorageDialog` is currently broken (noted in README)
- Tailwind v4 peer dependency; may conflict with apps already using Tailwind v3

---

## How to Use It (Junior Developer Walkthrough)

`pi-web-ui` is like a LEGO kit for AI chat UIs. The main pieces are:
- `Agent` (from `pi-agent-core`) — the brain that talks to the LLM
- `ChatPanel` — the body that shows conversations
- `AppStorage` — the memory that persists sessions

1. **Install:**
   ```bash
   npm install @mariozechner/pi-web-ui @mariozechner/pi-agent-core @mariozechner/pi-ai
   npm install @mariozechner/mini-lit lit
   ```

2. **Import CSS:**
   ```typescript
   import '@mariozechner/pi-web-ui/app.css';
   ```

3. **Set up storage (runs once at app startup):**
   ```typescript
   import { AppStorage, IndexedDBStorageBackend, SettingsStore, ProviderKeysStore, SessionsStore, setAppStorage } from '@mariozechner/pi-web-ui';

   const settings = new SettingsStore();
   const providerKeys = new ProviderKeysStore();
   const sessions = new SessionsStore();

   const backend = new IndexedDBStorageBackend({
     dbName: 'my-app', version: 1,
     stores: [settings.getConfig(), providerKeys.getConfig(), sessions.getConfig(), SessionsStore.getMetadataConfig()],
   });
   settings.setBackend(backend);
   providerKeys.setBackend(backend);
   sessions.setBackend(backend);
   setAppStorage(new AppStorage(settings, providerKeys, sessions, undefined, backend));
   ```

4. **Create an agent and attach it to the chat UI:**
   ```typescript
   import { Agent } from '@mariozechner/pi-agent-core';
   import { getModel } from '@mariozechner/pi-ai';
   import { ChatPanel, ApiKeyPromptDialog, defaultConvertToLlm } from '@mariozechner/pi-web-ui';

   const agent = new Agent({
     initialState: {
       systemPrompt: 'You are a helpful assistant.',
       model: getModel('anthropic', 'claude-sonnet-4-5-20250929'),
       thinkingLevel: 'off',
       messages: [],
       tools: [],
     },
     convertToLlm: defaultConvertToLlm,
   });

   const chatPanel = new ChatPanel();
   await chatPanel.setAgent(agent, {
     onApiKeyRequired: (provider) => ApiKeyPromptDialog.prompt(provider),
   });

   document.body.appendChild(chatPanel);
   ```

5. **The user can now type messages, attach files, and the model responds in real time.** When the user sends a message, `ChatPanel` calls `agent.prompt()`, subscribes to the event stream, and renders each chunk as it arrives.
