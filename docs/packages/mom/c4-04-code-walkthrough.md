## Learning Objectives

Trace a Slack mention through the complete mom stack, from the WebSocket event to the posted reply.

---

## Step 1 — Slack Event Arrives (`src/slack.ts`)

mom maintains a persistent WebSocket connection to Slack via Socket Mode. When a user types `@mom install jq`, Slack delivers an `app_mention` event:

```typescript
// src/slack.ts (simplified)
this.socket.on("message", async (raw) => {
  const event = JSON.parse(raw);
  if (event.payload?.event?.type === "app_mention") {
    await this.handler(event.payload.event, this.client);
  }
  this.socket.send(JSON.stringify({ envelope_id: event.envelope_id })); // ACK
});
```

---

## Step 2 — Handler Dispatches to AgentRunner (`src/main.ts`)

```typescript
// src/main.ts (simplified)
const handler: MomHandler = async (event, client) => {
  const runner = await getOrCreateRunner(event.channel, store, sandbox);
  const ctx: SlackContext = { channel: event.channel, client };
  await runner.run(ctx, store);
};
```

`getOrCreateRunner` creates a new `AgentRunner` for the channel if one doesn't exist.

---

## Step 3 — Context Loaded, Agent Prompted (`src/agent.ts`)

```typescript
// src/agent.ts (simplified)
async run(ctx: SlackContext, store: ChannelStore) {
  // Load channel memory and recent messages
  const memory = await store.loadMemory(ctx.channel);
  const systemPrompt = buildSystemPrompt(memory);

  this.agent.state.systemPrompt = systemPrompt;

  // Convert pending Slack messages into a user message
  const text = pendingMessages.map(m => `${m.userName}: ${m.text}`).join("\n");
  await this.agent.prompt(text);
}
```

---

## Step 4 — Tool Call: `bash` executes `apt-get install jq`

The LLM decides to install `jq` via bash. `SandboxExecutor` routes the command:

```typescript
// src/sandbox.ts (simplified)
async exec(command: string): Promise<ExecResult> {
  if (this.config.type === "docker") {
    return dockerExec(this.config.containerId, command);
  }
  return hostExec(command);
}
```

---

## Step 5 — Reply Sent to Slack

After `agent_end`, the AgentRunner collects the assistant's text content and posts it:

```typescript
agent.subscribe(async (event) => {
  if (event.type === "message_end" && event.message.role === "assistant") {
    const text = event.message.content
      .filter(c => c.type === "text")
      .map(c => c.text)
      .join("");
    await ctx.client.chat.postMessage({ channel: ctx.channel, text });
  }
});
```

---

**← [Component](./c4-03-component.md)** | **[Features →](./features/per-channel-memory.md)**
