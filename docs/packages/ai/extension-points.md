# Extension Points

`pi-ai` is designed to be extended without forking. This page covers every supported extension mechanism.

---

## Registering a custom provider

A custom provider lets you add support for a new wire protocol or a new API endpoint not covered by the built-in providers.

### 1. Implement `ApiProvider`

```typescript
import type { ApiProvider, AssistantMessageEvent } from "@mariozechner/pi-ai";
import { AssistantMessageEventStream } from "@mariozechner/pi-ai";

const myProvider: ApiProvider<"my-custom-api"> = {
  api: "my-custom-api",

  stream(model, context, options) {
    const stream = new AssistantMessageEventStream();
    (async () => {
      const partial = buildEmptyMessage(model);
      stream.push({ type: "start", partial });

      // Make your HTTP request here
      const response = await fetch(model.baseUrl, {
        method: "POST",
        headers: { "Authorization": `Bearer ${options?.apiKey}`, "Content-Type": "application/json" },
        body: JSON.stringify(buildPayload(model, context, options)),
        signal: options?.signal,
      });

      // Parse SSE or JSON and push events
      const text = await response.text();
      partial.content.push({ type: "text", text });
      stream.push({ type: "text_start", contentIndex: 0, partial });
      stream.push({ type: "text_end", contentIndex: 0, content: text, partial });

      const final = { ...partial, stopReason: "stop" as const, timestamp: Date.now() };
      stream.push({ type: "done", reason: "stop", message: final });
      stream.end(final);
    })();
    return stream;
  },

  streamSimple(model, context, options) {
    // Convert SimpleStreamOptions to your provider-specific options here
    return this.stream(model, context, options);
  },
};
```

### 2. Register the provider

```typescript
import { registerApiProvider } from "@mariozechner/pi-ai";

registerApiProvider(myProvider);
```

Registration is global and takes effect immediately. Call this once at application startup, before any `streamSimple()` calls that use your custom `api` string.

### 3. Use it with a custom model

```typescript
import type { Model } from "@mariozechner/pi-ai";
import { streamSimple } from "@mariozechner/pi-ai";

const model: Model<"my-custom-api"> = {
  id: "my-model-v1",
  name: "My Model V1",
  api: "my-custom-api",
  provider: "my-org",
  baseUrl: "https://api.my-org.com/v1/chat",
  reasoning: false,
  input: ["text"],
  cost: { input: 1, output: 3, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 32000,
  maxTokens: 2048,
};

const stream = streamSimple(model, context, { apiKey: "my-key" });
```

### Scoped registration and unregistration

Pass a `sourceId` to `registerApiProvider` so you can unregister all providers from a given source in one call:

```typescript
registerApiProvider(myProvider, "my-plugin");

// Later, to remove:
import { unregisterApiProviders } from "@mariozechner/pi-ai";
unregisterApiProviders("my-plugin");
```

---

## Adding a custom model

If you want to use an existing wire protocol (e.g., `"openai-completions"`) with a custom endpoint, you only need a model object — no new provider registration required:

```typescript
import type { Model } from "@mariozechner/pi-ai";

const customEndpointModel: Model<"openai-completions"> = {
  id: "my-fine-tune",
  name: "My Fine-Tuned Model",
  api: "openai-completions",
  provider: "my-org",
  baseUrl: "https://inference.my-org.com/v1",
  reasoning: false,
  input: ["text"],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 16384,
  maxTokens: 4096,
  compat: {
    supportsStore: false,
    supportsDeveloperRole: false,
    supportsReasoningEffort: false,
  },
};
```

Pass this model to `streamSimple()` directly. The built-in `openai-completions` provider implementation handles it.

---

## Intercepting payloads via `onPayload`

`onPayload` is called with the raw request body before it is sent to the provider. Return the payload unchanged (`undefined`) or return a modified copy.

```typescript
const stream = streamSimple(model, context, {
  onPayload: (payload, model) => {
    console.log(`[${model.id}] Request payload:`, JSON.stringify(payload, null, 2));
    return undefined;   // pass through unchanged
  },
});
```

Modify the payload (use carefully — invalid payloads cause provider errors):

```typescript
onPayload: (payload, model) => {
  const p = payload as Record<string, unknown>;
  return { ...p, user: "my-user-id" };  // inject a field
},
```

---

## Intercepting responses via `onResponse`

`onResponse` fires after the HTTP response headers arrive but before the body is consumed. Use it to inspect status codes and headers.

```typescript
const stream = streamSimple(model, context, {
  onResponse: (response, model) => {
    console.log(`[${model.id}] HTTP ${response.status}`);
    console.log("Rate limit remaining:", response.headers["x-ratelimit-remaining-requests"]);
  },
});
```

`onResponse` cannot modify the response. Its return value is ignored.

---

## Adding custom HTTP headers

Merge additional headers into every request made for a specific call:

```typescript
const stream = streamSimple(model, context, {
  headers: {
    "X-Request-Id": generateRequestId(),
    "X-Tenant-Id": tenantId,
  },
});
```

Custom headers are merged with provider defaults. Provider-required headers always win. Not available for the AWS Bedrock provider.

To inject headers on every call globally, wrap `streamSimple`:

```typescript
function myStreamSimple(model, context, options) {
  return streamSimple(model, context, {
    ...options,
    headers: { "X-App-Name": "my-app", ...options?.headers },
  });
}
```

---

## Custom `ThinkingLevel` budgets

The default token budgets for thinking levels are:

| Level | Default budget |
|---|---|
| `minimal` | 1,024 tokens |
| `low` | 2,048 tokens |
| `medium` | 8,192 tokens |
| `high` | 16,384 tokens |

Override them per call:

```typescript
const stream = streamSimple(model, context, {
  reasoning: "high",
  thinkingBudgets: {
    high: 32768,   // override "high" budget to 32k tokens
  },
});
```

`thinkingBudgets` is merged with the defaults — only the levels you specify are overridden. The `xhigh` level is handled separately (it maps to provider-specific "max effort" modes and does not have a token budget).

See also: [features/provider-auth.md](./features/provider-auth.md) for the `getApiKey` callback. [observability.md](./observability.md) for wrapping `streamSimple` for telemetry.
