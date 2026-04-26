# Design Patterns

`pi-ai` uses a small set of well-known patterns. Understanding them helps you read the source, extend the package, and reason about its behaviour under load.

---

## Adapter pattern — per-provider stream adapters

**What it is**: The Adapter pattern converts an incompatible interface into a target interface. Here, each LLM provider speaks a different wire protocol. The Adapter translates each protocol into the uniform `AssistantMessageEvent` sequence that callers expect.

**Where it lives**: `src/providers/*.ts` — one adapter per `KnownApi` value.

**Why it was chosen**: Every provider (Anthropic, OpenAI, Google, Bedrock, Mistral) has a different SSE format, different event names, different tool-call serialisation, and different stop conditions. The Adapter pattern keeps all that provider-specific ugliness inside the provider module. The rest of the system never sees raw SSE chunks — only normalised events.

```typescript
// Anthropic adapter: translates Anthropic SDK events → AssistantMessageEvents
for await (const chunk of response) {
  if (chunk.type === "content_block_delta" && chunk.delta.type === "text_delta") {
    stream.push({ type: "text_delta", contentIndex, delta: chunk.delta.text, partial });
  }
}

// OpenAI adapter: translates OpenAI SSE events → the same AssistantMessageEvents
for await (const chunk of response) {
  const delta = chunk.choices[0]?.delta?.content ?? "";
  if (delta) {
    stream.push({ type: "text_delta", contentIndex, delta, partial });
  }
}
```

Both adapters emit identical `text_delta` events. The caller sees one consistent interface regardless of which provider is under the hood.

---

## Registry pattern — API registry and model registry

**What it is**: The Registry pattern maintains a well-known collection of objects keyed by identifier, allowing dynamic lookup and registration.

**Where it lives**:
- `src/api-registry.ts` — `Map<Api, ApiProvider>` for stream implementations
- `src/models.ts` — `Map<Provider, Map<ModelId, Model>>` for model definitions

**Why it was chosen**: Providers and models need to be discoverable at runtime without the caller needing to know implementation details. The registry also supports dynamic extension — third-party code can call `registerApiProvider()` to add new wire protocols at runtime without modifying the library. This is essential for the testing infrastructure (`src/providers/faux.ts`) and for enterprise deployments with custom internal endpoints.

```typescript
// Registration (done once at startup by register-builtins.ts)
registerApiProvider({ api: "anthropic-messages", stream: ..., streamSimple: ... });

// Lookup (done on every streamSimple() call)
const provider = getApiProvider(model.api);
```

---

## Event stream / observable — `AssistantMessageEventStream`

**What it is**: The Observable pattern (or push-based iterator) allows a producer to emit values over time that a consumer receives asynchronously. `AssistantMessageEventStream` implements this as a standard JavaScript `AsyncIterable`.

**Where it lives**: `src/utils/event-stream.ts`

**Why it was chosen**: LLM streaming responses are inherently push-based — the server sends chunks as they are generated. JavaScript's `AsyncIterable` / `for await` idiom is the natural fit. It composes well with existing async patterns, requires no third-party reactive library, and gives consumers control over iteration speed (backpressure). The `result()` escape hatch provides a simple way to skip streaming and get the final message when incremental rendering is not needed.

```typescript
// Producer side (provider implementation)
stream.push({ type: "text_delta", delta: "Hello", ... });

// Consumer side (application code)
for await (const event of stream) { ... }
const message = await stream.result();
```

---

## Strategy pattern — provider selection

**What it is**: The Strategy pattern selects an algorithm (or implementation) at runtime based on a key. Here, `model.api` is the key; the corresponding `ApiProvider` is the strategy.

**Where it lives**: `src/stream.ts` — `resolveApiProvider(model.api)` selects the provider.

**Why it was chosen**: The choice of wire protocol is a runtime property of the model object, not a compile-time decision. The Strategy pattern lets the same `streamSimple()` call route to Anthropic, OpenAI, Google, or any custom provider purely based on the `api` field. Adding a new protocol requires only a new strategy (a new provider module) and registration — the dispatch logic never changes.

```typescript
// Strategy selection
const provider = resolveApiProvider(model.api);  // picks "anthropic-messages" → AnthropicProvider

// Strategy execution
return provider.streamSimple(model, context, options);
```

---

## Factory pattern — lazy provider instantiation

**What it is**: The Factory pattern encapsulates object creation. Here, provider modules are expensive to load (they import large SDKs). Lazy factories defer that cost until the first call.

**Where it lives**: `src/providers/register-builtins.ts` — `createLazyStream()` and `createLazySimpleStream()`.

**Why it was chosen**: `pi-ai` supports 10 built-in wire protocols. Eagerly importing all provider SDKs at startup would add hundreds of milliseconds to startup time and pull Node.js-only modules into browser bundles. The lazy factory pattern solves both problems: each provider module is imported exactly once, the first time a model that needs it is called. Subsequent calls reuse the cached import promise.

```typescript
// Factory registration (at startup — no import happens yet)
registerApiProvider({
  api: "anthropic-messages",
  stream: createLazyStream(() => import("./anthropic.js").then(...)),  // factory
  streamSimple: createLazySimpleStream(() => import("./anthropic.js").then(...)),
});

// First call to streamSimple() with an Anthropic model:
// → factory fires → import("./anthropic.js") → module cached → stream created
// Subsequent calls: module already cached, no import overhead
```

---

## Summary

| Pattern | Location | Problem solved |
|---|---|---|
| Adapter | `src/providers/*.ts` | Translate 10 different wire formats into one event protocol |
| Registry | `src/api-registry.ts`, `src/models.ts` | Runtime lookup and dynamic extension of providers and models |
| Event Stream / Observable | `src/utils/event-stream.ts` | Push-based async delivery matching LLM SSE streaming |
| Strategy | `src/stream.ts` | Select provider implementation at runtime from `model.api` |
| Factory (lazy) | `src/providers/register-builtins.ts` | Defer provider SDK loading for startup performance and browser safety |
