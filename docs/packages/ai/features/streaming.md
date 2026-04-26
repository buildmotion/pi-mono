# Feature — Streaming

Streaming is the core delivery mechanism in `pi-ai`. Instead of waiting for the model to finish generating its full response, you receive events incrementally as tokens arrive. This dramatically reduces time-to-first-token and lets your UI update in real time.

---

## Why streaming matters for UX

A typical LLM response takes 5–30 seconds to generate fully. Without streaming, the user stares at a blank screen until the entire response is ready. With streaming, text appears within the first 200–500 ms and flows token by token — matching the feel of a human typing.

`pi-ai` normalises the streaming formats of all supported providers into a single, provider-agnostic event protocol. You write one `for await` loop; pi-ai handles the translation.

---

## How `AssistantMessageEventStream` works

`AssistantMessageEventStream` is a push-based async queue built on `EventStream<AssistantMessageEvent, AssistantMessage>` (defined in `src/utils/event-stream.ts`).

It has two sides:

**Producer side (provider implementation)**
```typescript
const stream = new AssistantMessageEventStream();
stream.push({ type: "text_delta", contentIndex: 0, delta: "Hello", partial });
stream.push({ type: "done", reason: "stop", message: finalMessage });
stream.end();
```

**Consumer side (your application)**
```typescript
for await (const event of stream) {
  // handle each event
}
const message = await stream.result(); // AssistantMessage
```

The queue is backpressure-safe: if the provider is faster than the consumer, events accumulate in `queue[]`. If the consumer is faster than the provider (typical case), the consumer suspends at `await new Promise(...)` inside the async iterator until the next `push()` call.

`stream.result()` returns a `Promise<AssistantMessage>` that resolves the moment a `"done"` or `"error"` event is pushed.

---

## Event sequence — text-only response

For a simple user question with no tools:

```
start           — empty partial AssistantMessage is established
text_start      — a new TextContent block begins (contentIndex: 0)
text_delta      — "Hello" (partial updated)
text_delta      — ", world" (partial updated)
text_delta      — "!" (partial updated)
text_end        — TextContent block complete (contentIndex: 0)
done            — final AssistantMessage with stopReason "stop"
```

Every event except `done`/`error` carries a `partial` field — a snapshot of the in-progress `AssistantMessage` built up so far. You can render from `partial` at any point without waiting for `done`.

---

## Event sequence — tool-use response

When the model decides to call a tool:

```
start
text_start       — contentIndex: 0
text_delta       — "Let me check that for you."
text_end         — contentIndex: 0
toolcall_start   — contentIndex: 1, new ToolCall block begins
toolcall_delta   — streaming JSON arguments: '{"query":'
toolcall_delta   — '"latest news"}'
toolcall_end     — contentIndex: 1, final ToolCall with parsed arguments
done             — stopReason: "toolUse"
```

After `done` with `reason: "toolUse"`, the caller is expected to execute the tool, produce a `ToolResultMessage`, append both the assistant message and the tool result to the context, and make a follow-up call.

---

## Consuming the stream

### Minimal: only care about final result

```typescript
import { completeSimple } from "@mariozechner/pi-ai";
import { getModel }       from "@mariozechner/pi-ai/models";

const model   = getModel("openai", "gpt-4o");
const message = await completeSimple(model, {
  messages: [{ role: "user", content: "Explain entropy.", timestamp: Date.now() }],
});

console.log(message.content[0].text);
console.log("Cost: $", message.usage.cost.total.toFixed(6));
```

### Streaming with incremental rendering

```typescript
import { streamSimple } from "@mariozechner/pi-ai";
import { getModel }     from "@mariozechner/pi-ai/models";

const model  = getModel("anthropic", "claude-opus-4-5");
const stream = streamSimple(model, {
  messages: [{ role: "user", content: "Write a haiku about streaming.", timestamp: Date.now() }],
});

for await (const event of stream) {
  switch (event.type) {
    case "text_delta":
      process.stdout.write(event.delta);
      break;
    case "thinking_delta":
      // show thinking content if desired
      break;
    case "done":
      console.log("\n\nStop reason:", event.reason);
      console.log("Tokens used:", event.message.usage.totalTokens);
      break;
    case "error":
      console.error("Error:", event.error.errorMessage);
      break;
  }
}
```

### Using `partial` for live UI updates

```typescript
for await (const event of stream) {
  if ("partial" in event) {
    renderMessage(event.partial);   // re-render on every delta
  }
}
```

---

## Aborting mid-stream

Pass an `AbortSignal` via `options.signal`. When the signal fires, the provider cancels the HTTP request and emits an `error` event with `reason: "aborted"`.

```typescript
const controller = new AbortController();

const stream = streamSimple(model, context, {
  signal: controller.signal,
});

// Cancel after 3 seconds
setTimeout(() => controller.abort(), 3000);

for await (const event of stream) {
  if (event.type === "error" && event.reason === "aborted") {
    console.log("Stream cancelled.");
    break;
  }
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
```

The `AssistantMessage` returned by `stream.result()` will have `stopReason: "aborted"` and a non-empty `errorMessage`.

---

## Event type reference

| Event type | Key fields | When emitted |
|---|---|---|
| `start` | `partial` | Once, at the beginning of every stream |
| `text_start` | `contentIndex`, `partial` | When a new text content block begins |
| `text_delta` | `contentIndex`, `delta`, `partial` | For each text token |
| `text_end` | `contentIndex`, `content`, `partial` | When a text block is complete |
| `thinking_start` | `contentIndex`, `partial` | When a thinking block begins (reasoning models) |
| `thinking_delta` | `contentIndex`, `delta`, `partial` | For each thinking token |
| `thinking_end` | `contentIndex`, `content`, `partial` | When a thinking block is complete |
| `toolcall_start` | `contentIndex`, `partial` | When a tool call block begins |
| `toolcall_delta` | `contentIndex`, `delta`, `partial` | For each token of streaming tool arguments |
| `toolcall_end` | `contentIndex`, `toolCall`, `partial` | When a tool call block is complete with parsed arguments |
| `done` | `reason`, `message` | Terminal event on success |
| `error` | `reason`, `error` | Terminal event on failure or abort |

See `src/types.ts` for the full `AssistantMessageEvent` union type.

See also: [terminology.md](../terminology.md) for definitions of `AssistantMessageEvent`, `EventStream`, and `StopReason`.
