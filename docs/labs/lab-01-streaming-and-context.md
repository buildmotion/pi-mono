# Lab 01: Streaming LLM Responses and Managing Context with `pi-ai`

You are going to drive a language model directly — no SDK magic, no abstraction hiding the wire. By the end of this lab you will understand exactly what happens between `streamSimple()` and the moment text appears on screen, and you will be able to steer that interaction across multiple turns without losing the thread.

## Learning Objectives

- Understand the event-driven streaming model used by `@mariozechner/pi-ai`
- Construct a valid `Context` object and send it to an LLM
- Consume an `AssistantMessageEventStream` event by event
- Accumulate assistant replies into a multi-turn conversation history
- Read and interpret token-usage and cost data returned after each call

## Prerequisites

- Node.js 20 or later (`node --version`)
- An API key for at least one supported provider (Anthropic, OpenAI, Google, Mistral, or Groq)
- Basic TypeScript familiarity (`tsc --version` or `npx tsx`)

Set your key in the environment before starting:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# or
export OPENAI_API_KEY="sk-..."
```

---

## Background

### What Is Streaming?

When you call a non-streaming LLM endpoint, you wait for the entire response before seeing a single token. That latency is unacceptable in interactive applications and wastes time when you want to pipe output into downstream tools.

A streaming endpoint sends tokens as a server-sent event (SSE) stream. `pi-ai` wraps this as an `AssistantMessageEventStream` — an async iterable that yields typed events:

| Event type | When it fires |
|---|---|
| `start` | Immediately before the first token |
| `text_delta` | Each incremental text chunk |
| `toolcall_end` | When a complete tool call has been assembled |
| `done` | End of stream; carries the final `AssistantMessage` and `Usage` |
| `error` | Unrecoverable failure |

### Why Context Management Matters

An LLM is stateless. Every request must include the full conversation history — system prompt, prior user messages, prior assistant replies, tool results — to maintain coherence. Mismanaging this array is the most common source of "the model forgot what I said" bugs. This lab makes that array visible and explicit.

---

## Step 1: Install `pi-ai` and Select a Model

```bash
mkdir lab-01 && cd lab-01
npm init -y
npm install @mariozechner/pi-ai tsx typescript
```

Create `index.ts`:

```typescript
import { models } from "@mariozechner/pi-ai";

// Browse available models
console.log(`Total models available: ${models.length}`);

// Select a specific model
const model = models.find(
  (m) => m.provider === "anthropic" && m.id.includes("claude-3-5")
);

if (!model) {
  throw new Error("Model not found — check your provider and model ID filters");
}

console.log(`Selected model: ${model.name}`);
console.log(`  Provider:       ${model.provider}`);
console.log(`  API:            ${model.api}`);
console.log(`  Context window: ${model.contextWindow} tokens`);
console.log(`  Max output:     ${model.maxTokens} tokens`);
console.log(`  Input cost:     $${model.cost?.input ?? "unknown"} / 1M tokens`);
console.log(`  Output cost:    $${model.cost?.output ?? "unknown"} / 1M tokens`);
```

Run it:

```bash
npx tsx index.ts
```

You should see the model's metadata printed to the console. If you get "Model not found", try listing all providers:

```typescript
const providers = [...new Set(models.map((m) => m.provider))];
console.log("Available providers:", providers);
```

---

## Step 2: Build a Context Object and Call `streamSimple()`

The `Context` type has three fields: an optional system prompt, a required messages array, and an optional tools array. Start simple — no tools yet.

```typescript
import { models, streamSimple } from "@mariozechner/pi-ai";
import type { Context, UserMessage } from "@mariozechner/pi-ai";

const model = models.find(
  (m) => m.provider === "anthropic" && m.id.includes("claude-3-5")
)!;

const context: Context = {
  systemPrompt: "You are a concise assistant. Keep answers under three sentences.",
  messages: [
    {
      role: "user",
      content: "What is the difference between a process and a thread?",
    } satisfies UserMessage,
  ],
};

const stream = streamSimple(model, context);
console.log("Stream created — iterating events...");
```

`streamSimple()` returns immediately. It does not start the network request until you iterate the stream. Nothing has been sent yet.

---

## Step 3: Consume the Stream Event by Event

```typescript
for await (const event of stream) {
  switch (event.type) {
    case "start":
      process.stdout.write("[start]\n");
      break;

    case "text_delta":
      // Write each chunk directly — no newline — to simulate live typing
      process.stdout.write(event.delta);
      break;

    case "toolcall_end":
      // Not expected here, but good practice to handle
      console.log("\n[tool call]", event.toolCall.name);
      break;

    case "done":
      process.stdout.write("\n[done]\n");
      console.log("Stop reason:", event.message.stopReason);
      console.log("Usage:", event.usage);
      break;

    case "error":
      console.error("Stream error:", event.error);
      break;
  }
}
```

Run the full file. You should see the model's response printed incrementally, character by character, ending with the usage summary.

**Observe:** the `text_delta` events arrive as the model generates tokens, not all at once. This is the difference between streaming and batch.

---

## Step 4: Add a Second User Message — Multi-Turn Context

After the first turn completes, the assistant's reply lives only inside the `done` event. You must explicitly append it to your messages array before the next call.

```typescript
import { models, streamSimple } from "@mariozechner/pi-ai";
import type { Context, UserMessage, AssistantMessage } from "@mariozechner/pi-ai";

const model = models.find(
  (m) => m.provider === "anthropic" && m.id.includes("claude-3-5")
)!;

const context: Context = {
  systemPrompt: "You are a concise assistant. Keep answers under three sentences.",
  messages: [
    {
      role: "user",
      content: "What is the difference between a process and a thread?",
    } satisfies UserMessage,
  ],
};

async function runTurn(userText: string): Promise<AssistantMessage> {
  // Append the new user message
  context.messages.push({ role: "user", content: userText } satisfies UserMessage);

  let assistantMessage: AssistantMessage | undefined;

  for await (const event of streamSimple(model, context)) {
    if (event.type === "text_delta") process.stdout.write(event.delta);
    if (event.type === "done") {
      process.stdout.write("\n");
      assistantMessage = event.message;
    }
    if (event.type === "error") throw event.error;
  }

  if (!assistantMessage) throw new Error("Stream ended without a done event");

  // Append the assistant reply so the next turn has full history
  context.messages.push(assistantMessage);

  return assistantMessage;
}

// Turn 1
console.log("--- Turn 1 ---");
await runTurn("What is the difference between a process and a thread?");

// Turn 2 — the model should remember the prior question
console.log("--- Turn 2 ---");
await runTurn("Which one has its own memory space?");

console.log("\nFull message history length:", context.messages.length);
```

After both turns, `context.messages` should contain four entries: two user messages and two assistant messages, in order.

**Key insight:** the model does not "remember" anything. You are reconstructing its memory on every request by replaying the history.

---

## Step 5: Inspect Usage — Token Counts and Costs

The `done` event carries a `Usage` object. Extend `runTurn()` to accumulate totals:

```typescript
let totalInputTokens = 0;
let totalOutputTokens = 0;

async function runTurnWithUsage(userText: string): Promise<void> {
  context.messages.push({ role: "user", content: userText } satisfies UserMessage);

  let assistantMessage: AssistantMessage | undefined;
  let usage: { inputTokens: number; outputTokens: number } | undefined;

  for await (const event of streamSimple(model, context)) {
    if (event.type === "text_delta") process.stdout.write(event.delta);
    if (event.type === "done") {
      process.stdout.write("\n");
      assistantMessage = event.message;
      usage = event.usage;
    }
    if (event.type === "error") throw event.error;
  }

  if (!assistantMessage || !usage) throw new Error("Incomplete stream");

  context.messages.push(assistantMessage);

  totalInputTokens += usage.inputTokens;
  totalOutputTokens += usage.outputTokens;

  const inputCostPer1M = model.cost?.input ?? 0;
  const outputCostPer1M = model.cost?.output ?? 0;
  const turnCost =
    (usage.inputTokens / 1_000_000) * inputCostPer1M +
    (usage.outputTokens / 1_000_000) * outputCostPer1M;

  console.log(
    `  [usage] in=${usage.inputTokens} out=${usage.outputTokens} cost=$${turnCost.toFixed(6)}`
  );
}
```

Notice that `inputTokens` grows with each turn because the history grows. This is why long conversations become expensive and why context-window management (summarization, truncation) matters in production agents.

---

## Step 6 (Stretch): Switch Providers Without Changing Application Code

This is the payoff of a unified API. Change one line — the model selection — and the rest of your code is identical.

```typescript
// Swap to OpenAI
const model = models.find(
  (m) => m.provider === "openai" && m.id.includes("gpt-4o")
)!;

// Or Google
const model = models.find(
  (m) => m.provider === "google" && m.id.includes("gemini-2")
)!;
```

Run your multi-turn script against each provider and compare:
- Response latency (time to first token)
- Output quality for the same prompt
- Token counts and cost for the same conversation

---

## Expected Outcomes

By the end of this lab you should have:

- A working TypeScript file that streams responses and prints them incrementally
- A multi-turn loop that correctly maintains conversation history
- Token and cost tracking across turns
- At least one successful provider swap with zero application-code changes

---

## Check Yourself

1. What happens if you do not append the `AssistantMessage` to `context.messages` between turns? Try it and describe the model's behavior.
2. The `inputTokens` count grows with each turn even though your new question is short. Why?
3. What is the difference between `streamSimple()` and `stream()`? When would you use the lower-level `stream()` API?
4. If a network error interrupts the stream mid-response, which event type will you receive? How should you handle it in a production application?
5. A model's `contextWindow` is 200,000 tokens. Your conversation history is 195,000 tokens. What will happen on the next call, and how would you prevent it?

---

## What's Next

- **Lab 02** — [Tools and the Agent Loop](./lab-02-tools-and-agent-loop.md): add structured tool calls on top of the streaming foundation you built here
- **`@mariozechner/pi-ai` README** — full provider list, authentication options, and the `stream()` API
- **`packages/ai/src/types.ts`** — canonical type definitions for `Context`, `Model`, `AssistantMessageEventStream`, and `Usage`
