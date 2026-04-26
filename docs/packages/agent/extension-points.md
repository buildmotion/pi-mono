# Extension Points

`pi-agent-core` is designed to be extended at well-defined seams without forking the source. This document catalogues every hook and strategy point, explains when to use each, and provides working examples.

See also the monorepo [glossary — Extension Point](../../glossary.md#extension-point).

---

## Overview

| Extension point | Interface | When called | Primary use case |
|---|---|---|---|
| `CustomAgentMessages` | Declaration merging | At compile time | Add custom message types to the conversation |
| `convertToLlm` | `(messages: AgentMessage[]) => Message[]` | Before every LLM call | Filter/convert custom messages to LLM format |
| `transformContext` | `(messages, signal?) => AgentMessage[]` | Before `convertToLlm` | Context window management, external injection |
| `streamFn` | `StreamFn` | Per LLM call | Replace the LLM backend (proxy, mock, custom) |
| `getApiKey` | `(provider) => string \| undefined` | Per LLM call | Dynamic / rotating credentials |
| `beforeToolCall` | `(context, signal?) => BeforeToolCallResult` | Before each tool executes | Approval gates, sandboxing, logging |
| `afterToolCall` | `(context, signal?) => AfterToolCallResult` | After each tool executes | Result transformation, audit logging |
| `getSteeringMessages` | `() => AgentMessage[]` | After each tool batch | Mid-run injection (via Agent's steer queue) |
| `getFollowUpMessages` | `() => AgentMessage[]` | When agent would stop | Queued follow-up (via Agent's followUp queue) |

---

## `CustomAgentMessages` — custom message types

The `AgentMessage` type is `Message | CustomAgentMessages[keyof CustomAgentMessages]`. By default `CustomAgentMessages` is empty, so `AgentMessage === Message`. You extend it via TypeScript declaration merging:

```typescript
// In your app's type declarations (e.g., types.d.ts or any .ts file)
import type { AgentMessage } from "@mariozechner/pi-agent-core";

declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    // Key is the role; value is the full message type
    notification: {
      role: "notification";
      text: string;
      level: "info" | "warn" | "error";
      timestamp: number;
    };
    artifact: {
      role: "artifact";
      kind: "file" | "image" | "chart";
      path: string;
      timestamp: number;
    };
  }
}

// Now these are valid AgentMessage values
const note: AgentMessage = { role: "notification", text: "Build started", level: "info", timestamp: Date.now() };
agent.state.messages.push(note);
```

Custom messages flow through the conversation history and are visible to event listeners. They are not sent to the LLM unless `convertToLlm` explicitly maps them to a `Message`.

---

## `convertToLlm`

```typescript
convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>
```

Called before every LLM call. Converts the full `AgentMessage[]` transcript to the `Message[]` that `pi-ai`'s `streamSimple` accepts.

**Default behaviour:** filters to messages with `role === "user" | "assistant" | "toolResult"`. Custom messages are silently dropped.

**Override to:**
- Map custom message types to user/assistant messages.
- Inject a formatted summary of custom messages as a user message.
- Apply provider-specific transformations.

```typescript
const agent = new Agent({
  convertToLlm: (messages) =>
    messages.flatMap((m) => {
      if (m.role === "user" || m.role === "assistant" || m.role === "toolResult") {
        return [m];
      }
      if (m.role === "notification") {
        // Inject notifications as system-like user messages
        return [{
          role: "user",
          content: [{ type: "text", text: `[System notification: ${m.text}]` }],
          timestamp: m.timestamp,
        }];
      }
      return []; // drop everything else
    }),
});
```

**Contract:** must not throw or reject. Throwing from `convertToLlm` bypasses the normal event sequence. Return a safe fallback (e.g., the unfiltered messages) instead.

---

## `transformContext`

```typescript
transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>
```

Called before `convertToLlm` on every turn. Receives the full `AgentMessage[]` and returns a (possibly modified) `AgentMessage[]`. This is the correct place for context window management.

**Use cases:**
- Prune old messages when the context grows too large.
- Replace a long tool result with a summarised version.
- Inject externally fetched context (RAG, memory, tool outputs from other systems).

```typescript
const agent = new Agent({
  transformContext: async (messages, signal) => {
    const estimated = estimateTokens(messages);
    if (estimated < MAX_CONTEXT_TOKENS) {
      return messages; // nothing to do
    }
    // Keep the system messages and the last N turns
    return pruneOldTurns(messages, TARGET_CONTEXT_TOKENS);
  },
});
```

**Contract:** must not throw or reject. Return the original messages or a safe fallback. The `signal` is the agent's abort signal; respect it in long-running async operations.

---

## `streamFn`

```typescript
streamFn?: StreamFn
// StreamFn = (...args: Parameters<typeof streamSimple>) => ReturnType<typeof streamSimple>
```

Replaces the default `streamSimple` from `pi-ai`. Every LLM call the loop makes goes through `streamFn`.

**Use cases:**
- Route calls through a backend proxy (see `streamProxy` in [proxy.ts](../../packages/ai/README.md)).
- Mock the LLM for testing.
- Add request tracing around every call.
- Use a custom streaming protocol.

```typescript
// Wrap streamSimple to add tracing
const agent = new Agent({
  streamFn: (model, context, options) => {
    const traceId = crypto.randomUUID();
    console.log(`[${traceId}] LLM call → ${model.id}`);
    const stream = streamSimple(model, context, options);
    // Wrap result() to log completion
    return stream;
  },
});
```

```typescript
// Proxy backend for browser deployments
import { streamProxy } from "@mariozechner/pi-agent-core";

const agent = new Agent({
  streamFn: (model, context, options) =>
    streamProxy(model, context, {
      ...options,
      authToken: await getShortLivedToken(),
      proxyUrl: "https://api.yourapp.com",
    }),
});
```

**Contract:** `StreamFn` must not throw or return a rejected promise. Failures must be encoded as a stream ending in an `error` event with `stopReason: "error"` or `"aborted"`.

---

## `getApiKey`

```typescript
getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined
```

Called before every LLM call. Returns the API key for the given provider name. If it returns `undefined`, the loop falls back to `config.apiKey` (from `SimpleStreamOptions`).

**Use case:** short-lived tokens that expire during long tool execution phases.

```typescript
const agent = new Agent({
  getApiKey: async (provider) => {
    if (provider === "github-copilot") {
      // Token expires every hour; refresh as needed
      return await copilotTokenCache.getValid();
    }
    return process.env[`${provider.toUpperCase()}_API_KEY`];
  },
});
```

**Contract:** must not throw or reject. Return `undefined` when no key is available (the provider may still succeed if it uses environment variable fallback or IAM credentials).

---

## `beforeToolCall`

```typescript
beforeToolCall?: (context: BeforeToolCallContext, signal?: AbortSignal) => Promise<BeforeToolCallResult | undefined>
```

Called after argument validation, before `tool.execute`. Return `{ block: true, reason?: string }` to prevent execution.

```typescript
const agent = new Agent({
  beforeToolCall: async ({ toolCall, args, context }, signal) => {
    // Require human approval for file writes
    if (toolCall.name === "write_file") {
      const approved = await requestHumanApproval(toolCall, args);
      if (!approved) {
        return { block: true, reason: "User declined the write operation." };
      }
    }
  },
});
```

See [tool-execution.md — beforeToolCall](./features/tool-execution.md#beforetoolcall-hook) for the full context object and merge semantics.

---

## `afterToolCall`

```typescript
afterToolCall?: (context: AfterToolCallContext, signal?: AbortSignal) => Promise<AfterToolCallResult | undefined>
```

Called after `tool.execute`, before `tool_execution_end`. Return an `AfterToolCallResult` to override parts of the result.

```typescript
const agent = new Agent({
  afterToolCall: async ({ toolCall, result, isError }, signal) => {
    // Sanitise file contents before sending to LLM
    if (toolCall.name === "read_file" && !isError) {
      const sanitised = stripSecrets(result.content);
      return { content: sanitised };
    }
  },
});
```

See [tool-execution.md — afterToolCall](./features/tool-execution.md#aftertoolcall-hook) for the full context object and merge semantics.

---

## `getSteeringMessages` and `getFollowUpMessages`

These are internally wired by `Agent` from the steering and follow-up queues. You interact with them via `agent.steer()` and `agent.followUp()` rather than configuring the callbacks directly.

If you use the low-level `runAgentLoop` directly (without the `Agent` class), you can supply these callbacks in `AgentLoopConfig`:

```typescript
import { runAgentLoop } from "@mariozechner/pi-agent-core";

const config: AgentLoopConfig = {
  model,
  convertToLlm: (msgs) => msgs.filter(m => m.role !== "notification"),
  getSteeringMessages: async () => {
    return steeringBuffer.drain(); // your own queue implementation
  },
  getFollowUpMessages: async () => {
    return followUpBuffer.drain();
  },
};
```

See [steering-and-followup.md](./features/steering-and-followup.md) for the full behavioural specification.

---

## Combining extension points

Extension points compose. Here is a production-style configuration combining several:

```typescript
const agent = new Agent({
  initialState: {
    systemPrompt: systemPrompt,
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
    thinkingLevel: "low",
    tools: appTools,
    messages: await loadMessagesFromDb(sessionId),
  },
  convertToLlm: convertAppMessages,
  transformContext: pruneContext,
  getApiKey: rotatingKeyProvider,
  beforeToolCall: approvalGate,
  afterToolCall: auditAndSanitise,
  sessionId: sessionId,
  toolExecution: "parallel",
});

agent.subscribe(persistEvents);
agent.subscribe(updateUI);
```
