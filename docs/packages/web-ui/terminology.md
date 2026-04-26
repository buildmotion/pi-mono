# Terminology — pi-web-ui

**Artifact**
A piece of self-contained visual content (HTML, SVG, Markdown) produced by a tool call and rendered in the artifact side panel. Artifacts execute in a sandboxed iframe.

**Artifact sandbox**
An `<iframe sandbox="allow-scripts">` element that isolates AI-generated HTML from the host page. Scripts run but have no access to the parent DOM, cookies, IndexedDB, or network credentials.

**AttachmentTile**
A UI chip that represents an uploaded file before submission. Clicking it removes the attachment from the pending message.

**ChatPanel**
The root `<pi-chat-panel>` custom element. Owns and configures the `Agent`, renders the full chat UI, and manages session persistence.

**CORS proxy**
An optional Node.js reverse proxy server that forwards browser requests to LLM providers. Solves `Access-Control-Allow-Origin` restrictions without exposing API keys in client-side code.

**Custom element**
A browser-native web component (`customElements.define(…)`). `ChatPanel` is registered as `<pi-chat-panel>`.

**IndexedDB**
A transactional, asynchronous key-value database built into every modern browser. pi-web-ui uses it to persist sessions, API keys, and settings without a server.

**MessageBubble / StreamingMessageContainer**
The component that renders a single assistant message. During streaming it updates incrementally; after `message_end` it displays the final, fully rendered content.

**mini-lit**
A lightweight reactive element library (similar to Lit) used to author pi-web-ui's components. Provides reactive property binding and efficient DOM updates.

**Session**
A named, persisted conversation: an `id`, `name`, `createdAt` timestamp, and serialized `AgentMessage[]` array stored in IndexedDB.

**Shadow DOM**
The encapsulated DOM tree inside a custom element. Styles from the host page do not leak in; internal styles do not leak out.

**StorageBackend**
The interface (`src/storage/backends/`) that backs `AppStorage`. The default implementation uses IndexedDB; you can swap in any implementation (remote API, localStorage, etc.).

**Tailwind CSS**
A utility-first CSS framework used to style pi-web-ui components. Version 4 is used, configured via `src/app.css`.

**ToolCallCard**
A collapsible UI card that shows a tool call's name, arguments, and result (or error). Rendered inside the message list after each `tool_execution_end` event.

---

**Back to:** [README](./README.md) | [Global Glossary](../../glossary.md)
