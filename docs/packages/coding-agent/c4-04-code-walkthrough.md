## Learning Objectives

- Trace a complete `pi` interactive session from startup to a tool-executing response.
- Understand how `AgentSession` wraps `Agent` and adds session persistence on top.
- See where extensions intercept the lifecycle.

---

## Step 1 — CLI Entry (`src/cli.ts`)

When you run `pi`, the CLI parses arguments and selects a mode:

```typescript
// src/cli.ts (simplified)
const mode = args.rpc ? "rpc" : args.print ? "print" : "interactive";
const session = await createAgentSession({ workingDir, mode, ... });
await session.start();
```

---

## Step 2 — AgentSession Setup (`src/core/agent-session.ts`)

`AgentSession` creates the `Agent`, registers tools, and loads extensions:

```typescript
// src/core/agent-session.ts (simplified)
this.agent = new Agent({
  initialState: {
    model: await this.modelResolver.resolve(),
    systemPrompt: await buildSystemPrompt(options),
    tools: createBuiltinTools(options),
  },
  getApiKey: (provider) => this.authStorage.getApiKey(provider),
});

// Fire session_start event so extensions can register additional tools
await this.extensionRunner.emitSessionStart(this);
```

---

## Step 3 — Interactive Mode Input (`src/modes/interactive/interactive-mode.ts`)

The TUI renders a multi-line editor. On submit:

```typescript
// src/modes/interactive/interactive-mode.ts (simplified)
editor.on("submit", async (text) => {
  await session.agent.prompt(text);
});
```

---

## Step 4 — Agent Loop Runs

`agent.prompt()` starts the loop:
1. Emits `agent_start`, `turn_start`.
2. Calls `streamSimple(model, context)` — the LLM responds with a tool call.
3. Emits `toolcall_end` — the tool is the `bash` tool.

---

## Step 5 — Bash Tool Executes (`src/core/tools/bash.ts`)

```typescript
// src/core/tools/bash.ts (simplified)
execute: async (toolCallId, { command }, signal, onUpdate) => {
  const result = await executeBash(command, { signal, onUpdate });
  return {
    content: [{ type: "text", text: result.output }],
    details: { exitCode: result.exitCode, command },
  };
},
```

---

## Step 6 — Extension Hooks

Before and after the tool runs, `ExtensionRunner` fires hooks:

```typescript
// src/core/extensions/runner.ts (simplified)
const beforeResult = await runner.emitBeforeToolCall(toolName, args, context);
if (beforeResult?.block) return blockedResult(beforeResult.reason);

const toolResult = await tool.execute(toolCallId, args, signal, onUpdate);

await runner.emitAfterToolCall(toolName, args, toolResult, context);
```

---

## Step 7 — Session Persistence

After `agent_end`, `AgentSession` writes messages to disk:

```typescript
agent.subscribe(async (event) => {
  if (event.type === "agent_end") {
    await this.sessionManager.saveMessages(
      this.sessionId,
      this.agent.state.messages,
    );
  }
});
```

---

**← [Component](./c4-03-component.md)** | **[Features →](./features/agent-session.md)**
