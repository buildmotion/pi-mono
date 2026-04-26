## Learning Objectives

- Trace a complete round-trip from user input to streamed assistant response in `pi-web-ui`.
- Understand how `AgentEvent` types map to UI state changes.
- See where file attachments and artifact rendering are inserted into the flow.

---

## Step 1 — User Types and Submits

The `AgentInterface` component (input bar at the bottom) fires a `submit` event when the user presses Enter or clicks Send.

```typescript
// src/components/AgentInterface.ts (simplified)
this.sendButton.addEventListener("click", () => {
  const text = this.editor.value.trim();
  if (!text) return;
  this.dispatchEvent(new CustomEvent("submit", { detail: { text, attachments: this.attachments } }));
  this.editor.value = "";
  this.attachments = [];
});
```

---

## Step 2 — ChatPanel Handles the Submit

`ChatPanel` listens for the `submit` event, converts attachments to `ImageContent`, and calls `agent.prompt()`.

```typescript
// src/ChatPanel.ts (simplified)
this.agentInterface.addEventListener("submit", async (e: CustomEvent) => {
  const { text, attachments } = e.detail;

  // Convert uploaded files to base64 ImageContent
  const images = await Promise.all(
    attachments.map(file => extractDocument(file))
  );

  await this.agent.prompt(text, images.flat());
});
```

---

## Step 3 — Agent Runs and Emits Events

`agent.prompt()` starts the agent loop. `ChatPanel` subscribed to events via `agent.subscribe()` during setup:

```typescript
// src/ChatPanel.ts (simplified setup)
this.agent.subscribe((event) => {
  switch (event.type) {
    case "message_start":
      this.messageList.startStreaming(event.message);
      break;
    case "message_update":
      this.messageList.updateStreaming(event.message);
      break;
    case "message_end":
      this.messageList.finalizeMessage(event.message);
      break;
    case "tool_execution_start":
      this.messageList.showToolCard(event.toolCallId, event.toolName, event.args);
      break;
    case "tool_execution_end":
      this.messageList.updateToolCard(event.toolCallId, event.result, event.isError);
      break;
    case "agent_end":
      this.agentInterface.setEnabled(true);
      break;
  }
});
```

---

## Step 4 — Streaming Renders Incrementally

Each `message_update` event carries the latest partial `AssistantMessage`. The `StreamingMessageContainer` re-renders only the changed text delta — no full DOM replacement:

```typescript
// src/components/StreamingMessageContainer.ts (simplified)
update(message: AssistantMessage) {
  for (const block of message.content) {
    if (block.type === "text") {
      this.textNode.textContent = block.text; // incremental update
    }
  }
}
```

---

## Step 5 — Tool Call Result Becomes an Artifact

If the JavaScript REPL tool returns HTML, `ChatPanel` injects it into the `SandboxedIframe`:

```typescript
// src/ChatPanel.ts (simplified)
case "tool_execution_end":
  if (event.toolName === "javascript_repl" && !event.isError) {
    this.artifactsPanel.setContent(event.result.details.html);
  }
  break;
```

```typescript
// src/components/SandboxedIframe.ts (simplified)
setContent(html: string) {
  this.iframe.srcdoc = html; // iframe is sandbox="allow-scripts"
}
```

---

## Step 6 — Session Saved

On `agent_end`, `ChatPanel` serializes `agent.state.messages` to IndexedDB via the `StorageLayer`:

```typescript
this.agent.subscribe(async (event) => {
  if (event.type === "agent_end") {
    await this.storage.saveSession(this.sessionId, this.agent.state.messages);
  }
});
```

---

**← [Component](./c4-03-component.md)** | **[Features →](./features/chat-panel.md)**
