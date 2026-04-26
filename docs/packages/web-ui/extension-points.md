# Extension Points — pi-web-ui

`pi-web-ui` is designed to be composed. The four primary extension axes are: custom tools, custom theming, custom storage, and internationalization.

---

## 1. Adding Custom Tools

Attach any `AgentTool` to the agent before binding it to ChatPanel:

```typescript
import { Type } from "typebox";

const myTool = {
  name: "fetch_docs",
  label: "Fetch Docs",
  description: "Fetches a documentation page and returns its text.",
  parameters: Type.Object({ url: Type.String() }),
  execute: async (_id, { url }) => {
    const res = await fetch(url);
    const text = await res.text();
    return { content: [{ type: "text", text }], details: {} };
  },
};

agent.state.tools = [...agent.state.tools, myTool];
panel.agent = agent;
```

To produce artifacts from your tool, return `details: { html: "<html>…</html>" }`.

---

## 2. Theming with CSS Custom Properties

`ChatPanel` exposes a set of CSS custom properties for theming:

```css
pi-chat-panel {
  --pi-bg: #0d1117;
  --pi-surface: #161b22;
  --pi-text: #e6edf3;
  --pi-accent: #238636;
  --pi-border: #30363d;
  --pi-code-bg: #1c2128;
  --pi-user-bubble-bg: #1c2128;
  --pi-assistant-bubble-bg: #0d1117;
}
```

All Tailwind classes inside the shadow DOM inherit these properties.

---

## 3. Custom Storage Backend

Implement `StorageBackend` and pass it to `AppStorage`:

```typescript
import type { StorageBackend } from "@mariozechner/pi-web-ui/storage";
import { AppStorage } from "@mariozechner/pi-web-ui/storage";

class RemoteStorage implements StorageBackend {
  async saveSession(s) { await fetch("/api/sessions", { method: "POST", body: JSON.stringify(s) }); }
  async loadSession(id) { return (await fetch(`/api/sessions/${id}`)).json(); }
  async listSessions() { return (await fetch("/api/sessions")).json(); }
  async setApiKey(p, k) { await fetch("/api/keys", { method: "POST", body: JSON.stringify({ provider: p, key: k }) }); }
  async getApiKey(p) { return (await fetch(`/api/keys/${p}`)).text(); }
  async getSettings() { return (await fetch("/api/settings")).json(); }
  async saveSettings(s) { await fetch("/api/settings", { method: "PUT", body: JSON.stringify(s) }); }
}

panel.storage = new AppStorage(new RemoteStorage());
```

---

## 4. Internationalization

`ChatPanel` emits locale strings through a `strings` property. Provide a partial override:

```typescript
panel.strings = {
  send: "Enviar",
  placeholder: "Escribe tu mensaje…",
  newSession: "Nueva sesión",
};
```

---

## 5. Embedding in Frameworks

**React:**
```tsx
import "@mariozechner/pi-web-ui";
import { useEffect, useRef } from "react";

export function AiChat({ agent }: { agent: Agent }) {
  const ref = useRef<HTMLElement>(null);
  useEffect(() => { if (ref.current) (ref.current as any).agent = agent; }, [agent]);
  return <pi-chat-panel ref={ref} />;
}
```

**Vue:**
```vue
<template>
  <pi-chat-panel ref="panel" />
</template>
<script setup>
import "@mariozechner/pi-web-ui";
import { onMounted, ref } from "vue";
const panel = ref(null);
onMounted(() => { panel.value.agent = props.agent; });
</script>
```

---

**What's Next:** [Observability](./observability.md) | [Design Patterns](./design-patterns.md)
