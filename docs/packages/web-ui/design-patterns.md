# Design Patterns — pi-web-ui

## 1. Web Components (Custom Elements)

**Pattern:** Encapsulate a reusable UI as a browser-native custom element with its own shadow DOM.

`ChatPanel` is registered as `<pi-chat-panel>`. Consumers do not need to know its internal structure — they set properties (`agent`, `strings`, `storage`) and the component handles rendering. Shadow DOM prevents style bleed-in from host pages.

**Why:** Framework-agnostic; works in React, Vue, Svelte, or plain HTML with identical semantics.

---

## 2. Observer (Agent Event Subscription)

**Pattern:** Components subscribe to a central event emitter and update themselves reactively.

`ChatPanel` calls `agent.subscribe()` once. When `message_update` arrives it patches the streaming bubble; when `agent_end` arrives it saves the session. No polling, no shared mutable state exposed to child components.

**Why:** Clean separation between business logic (Agent) and UI. Multiple subscribers (logging, storage, analytics) can coexist without coupling.

---

## 3. Adapter (File Ingestion Pipeline)

**Pattern:** Translate between incompatible interfaces — here, between browser `File` objects and `pi-ai` `ImageContent` / `TextContent`.

`src/utils/extract-document.ts` adapts PDF, DOCX, XLSX, PPTX, and image files into the message content format expected by `pi-ai`. Each file type has its own extraction path; the adapter hides this complexity from `ChatPanel`.

**Why:** Adding a new file type requires only a new adapter case, not changes to ChatPanel or Agent.

---

## 4. Sandbox (iframe Isolation)

**Pattern:** Execute untrusted code in a restricted container.

`SandboxedIframe` (`src/components/SandboxedIframe.ts`) gives AI-generated HTML its own JavaScript context without `allow-same-origin`. The artifact can render charts and interactive content without any access to the host page's credentials or DOM.

**Why:** AI-generated code cannot be fully trusted. The sandbox provides a safe execution environment without a server-side container.

---

## 5. Repository (Storage Layer)

**Pattern:** Abstract the data persistence mechanism behind a domain-oriented interface.

`AppStorage` (`src/storage/store.ts`) exposes `saveSession`, `loadSession`, `listSessions`, `setApiKey`, `getApiKey` — domain verbs, not IndexedDB-specific operations. Swapping to a remote API only requires a new `StorageBackend` implementation.

**Why:** Decouples UI code from storage technology. Enables testing with in-memory backends.

---

**What's Next:** [Terminology](./terminology.md) | [Extension Points](./extension-points.md)
