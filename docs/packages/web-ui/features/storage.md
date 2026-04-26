# Feature: Session Storage with IndexedDB

## Learning Objectives

- Understand the three IndexedDB object stores used by `pi-web-ui`.
- Load a past session and restore it to an Agent.
- Store and retrieve API keys safely.
- Replace the default IndexedDB backend with a custom storage implementation.

---

## Background

`pi-web-ui` ships a storage layer (`src/storage/`) that persists data to the browser's IndexedDB — a transactional key-value database available in every modern browser. No server is required.

Source: `src/storage/store.ts`, `src/storage/app-storage.ts`

---

## IndexedDB Schema

| Store | Key | Value |
|-------|-----|-------|
| `sessions` | `sessionId` (UUID) | `{ id, name, createdAt, messages: AgentMessage[] }` |
| `apiKeys` | `provider` string | `string` (the API key) |
| `settings` | `"settings"` | `{ theme, modelId, providerId, … }` |

---

## Step 1 — Access the Storage Layer

```typescript
import { AppStorage } from "@mariozechner/pi-web-ui/storage";

const storage = new AppStorage();
await storage.open(); // opens or creates the IndexedDB database
```

---

## Step 2 — Save a Session

```typescript
import { Agent } from "@mariozechner/pi-agent-core";

const agent = new Agent({ /* … */ });
// … run the agent …

await storage.saveSession({
  id: crypto.randomUUID(),
  name: "My first session",
  createdAt: Date.now(),
  messages: agent.state.messages,
});
```

---

## Step 3 — List and Restore Sessions

```typescript
const sessions = await storage.listSessions(); // [{id, name, createdAt}, …]

const session = await storage.loadSession(sessions[0].id);
if (session) {
  agent.state.messages = session.messages;
}
```

---

## Step 4 — Store and Retrieve an API Key

```typescript
await storage.setApiKey("anthropic", "sk-ant-...");

const key = await storage.getApiKey("anthropic"); // "sk-ant-..."
```

> **Security note:** Keys stored in IndexedDB are accessible to any JavaScript on the same origin. For production multi-user apps, store keys server-side and inject them via a proxy.

---

## Step 5 — Replace the Backend

The storage layer accepts a backend interface. To swap IndexedDB for, say, `localStorage` or a remote API:

```typescript
import type { StorageBackend } from "@mariozechner/pi-web-ui/storage";

class MyBackend implements StorageBackend {
  async saveSession(session) { /* … */ }
  async loadSession(id) { /* … */ }
  async listSessions() { /* … */ }
  async setApiKey(provider, key) { /* … */ }
  async getApiKey(provider) { /* … */ }
  async getSettings() { /* … */ }
  async saveSettings(s) { /* … */ }
}

const storage = new AppStorage(new MyBackend());
```

---

## Check-Yourself Questions

1. How many IndexedDB object stores does `pi-web-ui` use by default?
2. What is the risk of storing API keys in IndexedDB?
3. How would you migrate session history from one browser to another?
4. What interface must a custom backend implement to replace IndexedDB?
5. When does `ChatPanel` automatically save to storage — on every message, or at session end?

---

**What's Next:** [Artifacts](./artifacts.md) | [Extension Points](../extension-points.md)
