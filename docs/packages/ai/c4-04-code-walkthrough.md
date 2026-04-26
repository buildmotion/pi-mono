# Code Walkthrough — From `streamSimple()` to Event Emission

This walkthrough traces a single `streamSimple()` call from your application code through the package internals, all the way to the first `text_delta` event arriving in your `for await` loop. Each step includes annotated source excerpts.

---

## The call you write

```typescript
import { streamSimple } from "@mariozechner/pi-ai";
import { getModel }     from "@mariozechner/pi-ai/models";

const model = getModel("anthropic", "claude-opus-4-5");

const stream = streamSimple(model, {
  messages: [{ role: "user", content: "What is 2 + 2?", timestamp: Date.now() }],
});

for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}

const message = await stream.result();
console.log("Stop reason:", message.stopReason);
```

---

## Step 1 — Model lookup (`src/models.ts`)

```typescript
// src/models.ts
export function getModel<TProvider extends KnownProvider, TModelId extends keyof (typeof MODELS)[TProvider]>(
  provider: TProvider,
  modelId: TModelId,
): Model<ModelApi<TProvider, TModelId>> {
  const providerModels = modelRegistry.get(provider);         // 1. look up provider bucket
  return providerModels?.get(modelId as string) as ...;       // 2. look up model by id
}
```

`getModel("anthropic", "claude-opus-4-5")` performs two `Map` lookups and returns a `Model<"anthropic-messages">` object that looks like:

```typescript
{
  id: "claude-opus-4-5",
  name: "Claude Opus 4.5",
  api: "anthropic-messages",       // <-- this is the dispatch key
  provider: "anthropic",
  baseUrl: "https://api.anthropic.com",
  reasoning: true,
  input: ["text", "image"],
  cost: { input: 15, output: 75, cacheRead: 1.5, cacheWrite: 18.75 },
  contextWindow: 200000,
  maxTokens: 32000,
}
```

The `api` field is what the Stream Engine uses to choose the right provider implementation.

---

## Step 2 — Entry point (`src/stream.ts`)

`import { streamSimple } from "@mariozechner/pi-ai"` pulls in `src/index.ts`, which re-exports from `src/stream.ts`. The first thing `stream.ts` does at module load time is:

```typescript
import "./providers/register-builtins.js";   // side-effect: registers all built-in providers
```

Then `streamSimple` itself:

```typescript
// src/stream.ts
export function streamSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions,
): AssistantMessageEventStream {
  const provider = resolveApiProvider(model.api);   // 1. look up "anthropic-messages"
  return provider.streamSimple(model, context, options);  // 2. delegate
}

function resolveApiProvider(api: Api) {
  const provider = getApiProvider(api);
  if (!provider) throw new Error(`No API provider registered for api: ${api}`);
  return provider;
}
```

`resolveApiProvider` calls into the API Registry.

---

## Step 3 — API Registry dispatch (`src/api-registry.ts`)

```typescript
// src/api-registry.ts
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

export function getApiProvider(api: Api): ApiProviderInternal | undefined {
  return apiProviderRegistry.get(api)?.provider;
}
```

The registry was populated at module load time by `register-builtins.ts`. For `"anthropic-messages"` it returns an `ApiProviderInternal` whose `streamSimple` is a lazy wrapper function.

---

## Step 4 — Lazy provider loading (`src/providers/register-builtins.ts`)

Built-in providers are loaded lazily to keep startup time low and avoid pulling Node.js-only modules into browser bundles.

```typescript
// src/providers/register-builtins.ts

// createLazySimpleStream returns a StreamFunction that:
// 1. Creates an AssistantMessageEventStream immediately (returned to the caller)
// 2. Dynamically imports the real provider module asynchronously
// 3. Forwards events from the real stream into the outer stream
function createLazySimpleStream<TApi, TOptions, TSimpleOptions>(
  loadModule: () => Promise<LazyProviderModule<TApi, TOptions, TSimpleOptions>>,
): StreamFunction<TApi, TSimpleOptions> {
  return (model, context, options) => {
    const outer = new AssistantMessageEventStream();  // returned immediately

    loadModule()                                       // async import
      .then((module) => {
        const inner = module.streamSimple(model, context, options);
        forwardStream(outer, inner);                   // pipe inner → outer
      })
      .catch((error) => {
        outer.push({ type: "error", reason: "error", error: createLazyLoadErrorMessage(model, error) });
        outer.end();
      });

    return outer;   // caller gets this before the import resolves
  };
}
```

Because `AssistantMessageEventStream` is a push queue, your `for await` loop simply waits for events. The dynamic import runs in the background and events start flowing once the module is loaded.

---

## Step 5 — Option normalisation (`src/providers/simple-options.ts`)

Inside `streamSimpleAnthropic`, the first thing that happens is option conversion:

```typescript
// src/providers/simple-options.ts
export function buildBaseOptions(model, options?, apiKey?): StreamOptions {
  return {
    temperature:    options?.temperature,
    maxTokens:      options?.maxTokens ?? (model.maxTokens > 0 ? Math.min(model.maxTokens, 32000) : undefined),
    signal:         options?.signal,
    apiKey:         apiKey || options?.apiKey,
    cacheRetention: options?.cacheRetention,
    sessionId:      options?.sessionId,
    headers:        options?.headers,
    onPayload:      options?.onPayload,
    onResponse:     options?.onResponse,
    timeoutMs:      options?.timeoutMs,
    maxRetries:     options?.maxRetries,
    maxRetryDelayMs: options?.maxRetryDelayMs,
    metadata:       options?.metadata,
  };
}
```

If `reasoning` is set, `adjustMaxTokensForThinking` computes the thinking budget and inflates `maxTokens` to accommodate it.

---

## Step 6 — Provider makes the HTTP request (`src/providers/anthropic.ts`)

The Anthropic provider creates an `AssistantMessageEventStream`, fires the HTTP request via the Anthropic SDK, and processes SSE chunks in a background async IIFE:

```typescript
// src/providers/anthropic.ts (simplified)
export function streamAnthropic(model, context, options?): AssistantMessageEventStream {
  const stream = new AssistantMessageEventStream();
  const client = new Anthropic({ apiKey: options?.apiKey ?? getEnvApiKey("anthropic"), ... });

  (async () => {
    // Emit start event with an empty partial message
    stream.push({ type: "start", partial: buildEmptyMessage(model) });

    // Open SSE connection to api.anthropic.com
    const response = await client.messages.stream({ model: model.id, messages, ... });

    for await (const chunk of response) {
      // Translate Anthropic SDK events into pi-ai AssistantMessageEvents
      switch (chunk.type) {
        case "content_block_start":
          if (chunk.content_block.type === "text") {
            stream.push({ type: "text_start", contentIndex, partial });
          }
          break;
        case "content_block_delta":
          if (chunk.delta.type === "text_delta") {
            partial.content[contentIndex].text += chunk.delta.text;
            stream.push({ type: "text_delta", contentIndex, delta: chunk.delta.text, partial });
          }
          break;
        case "message_stop":
          stream.push({ type: "done", reason: stopReason, message: finalMessage });
          break;
      }
    }
    stream.end();
  })();

  return stream;   // returned before the IIFE resolves
}
```

---

## Step 7 — EventStream delivers events (`src/utils/event-stream.ts`)

```typescript
// src/utils/event-stream.ts
export class EventStream<T, R> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: ((value: IteratorResult<T>) => void)[] = [];
  private done = false;

  push(event: T): void {
    if (this.done) return;
    if (this.isComplete(event)) {
      this.done = true;
      this.resolveFinalResult(this.extractResult(event));
    }
    const waiter = this.waiting.shift();
    if (waiter) {
      waiter({ value: event, done: false });   // deliver to waiting consumer
    } else {
      this.queue.push(event);                  // buffer until consumer is ready
    }
  }

  async *[Symbol.asyncIterator](): AsyncIterator<T> {
    while (true) {
      if (this.queue.length > 0) {
        yield this.queue.shift()!;
      } else if (this.done) {
        return;
      } else {
        // Suspend the generator until push() delivers the next event
        const result = await new Promise<IteratorResult<T>>(resolve => this.waiting.push(resolve));
        if (result.done) return;
        yield result.value;
      }
    }
  }
}
```

Your `for await` loop suspends at the `await new Promise(...)` line. When the Anthropic provider calls `stream.push(event)`, it finds your waiting resolver, calls it, and your loop resumes.

---

## Full call sequence summary

```
getModel("anthropic", "claude-opus-4-5")
  → Map lookup in models.generated.ts → returns Model<"anthropic-messages">

streamSimple(model, context)
  → stream.ts: resolveApiProvider("anthropic-messages")
  → api-registry.ts: getApiProvider("anthropic-messages") → ApiProviderInternal
  → register-builtins.ts: lazy wrapper creates AssistantMessageEventStream (A)
      [async] dynamically imports anthropic.ts
  → returns (A) immediately

for await (const event of A)
  → EventStream suspends, awaiting first push()

  [async, once import resolves]
  → anthropic.ts: streamSimple → buildBaseOptions() → new AssistantMessageEventStream (B)
  → anthropic.ts: makes HTTPS request to api.anthropic.com
  → forwardStream(A, B): pipes B → A
  → SSE chunk arrives → B.push({ type: "text_delta", delta: "4" })
  → forwardStream copies to A.push(...)
  → EventStream resolves waiting Promise → your loop yields { type: "text_delta", delta: "4" }

  [on message_stop]
  → B.push({ type: "done", message: finalMessage })
  → A.push({ type: "done", message: finalMessage })
  → EventStream marks done, resolves finalResultPromise

stream.result()
  → returns the resolved AssistantMessage promise
```
