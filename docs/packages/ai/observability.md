# Observability

`pi-ai` exposes three hooks for observability: `onPayload`, `onResponse`, and the `Usage` object on every completed message. Together they give you full visibility into every LLM call without modifying provider code.

---

## `onPayload` — log the raw request

`onPayload` fires after `pi-ai` has built the provider-specific request body but before it sends the HTTP request. It receives the payload as a plain JavaScript object and the `Model` that is being called.

```typescript
const stream = streamSimple(model, context, {
  onPayload: (payload, model) => {
    console.log(JSON.stringify({
      event: "llm_request",
      provider: model.provider,
      model: model.id,
      api: model.api,
      payload,
    }));
    return undefined;   // return undefined to leave payload unchanged
  },
});
```

**Return value**: if `onPayload` returns a non-undefined value, that value replaces the payload. Use this for mutation carefully — invalid payloads cause provider errors. For logging-only use cases, always return `undefined`.

---

## `onResponse` — log response headers and status

`onResponse` fires immediately after the HTTP response arrives (headers only; the body stream has not been consumed yet). It receives a `ProviderResponse` object and the `Model`.

```typescript
const stream = streamSimple(model, context, {
  onResponse: (response, model) => {
    console.log(JSON.stringify({
      event: "llm_response_headers",
      provider: model.provider,
      model: model.id,
      httpStatus: response.status,
      rateLimitRemainingRequests: response.headers["x-ratelimit-remaining-requests"],
      rateLimitRemainingTokens: response.headers["x-ratelimit-remaining-tokens"],
      requestId: response.headers["request-id"] ?? response.headers["x-request-id"],
    }));
  },
});
```

`onResponse` is not available for the AWS Bedrock provider (which uses the AWS SDK rather than a raw `fetch` response).

---

## `Usage` — structured token and cost logging

Every `AssistantMessage` (including error messages) carries a `Usage` object:

```typescript
interface Usage {
  input: number;       // prompt tokens consumed
  output: number;      // completion tokens generated
  cacheRead: number;   // tokens served from provider cache
  cacheWrite: number;  // tokens written to provider cache
  totalTokens: number; // input + output (not including cache tokens)
  cost: {
    input: number;     // USD cost of input tokens
    output: number;    // USD cost of output tokens
    cacheRead: number; // USD cost of cache-read tokens
    cacheWrite: number;// USD cost of cache-write tokens
    total: number;     // sum of all cost fields
  };
}
```

Log it after `stream.result()` or `completeSimple()`:

```typescript
const message = await stream.result();

console.log(JSON.stringify({
  event: "llm_done",
  provider: message.provider,
  model: message.model,
  stopReason: message.stopReason,
  inputTokens: message.usage.input,
  outputTokens: message.usage.output,
  cacheReadTokens: message.usage.cacheRead,
  cacheWriteTokens: message.usage.cacheWrite,
  totalTokens: message.usage.totalTokens,
  costUsd: message.usage.cost.total,
  responseId: message.responseId,
}));
```

---

## Suggested log schema for each LLM call

The following schema captures everything needed for cost tracking, latency analysis, and debugging:

```typescript
interface LlmCallLog {
  // Identity
  callId: string;           // your generated UUID per call
  sessionId?: string;       // from options.sessionId

  // Timing
  startedAt: number;        // Unix ms before streamSimple()
  firstTokenAt?: number;    // Unix ms when first text_delta arrives
  completedAt: number;      // Unix ms when done/error fires

  // Model
  provider: string;
  model: string;
  api: string;

  // Outcome
  stopReason: string;
  errorMessage?: string;

  // Tokens and cost
  inputTokens: number;
  outputTokens: number;
  cacheReadTokens: number;
  cacheWriteTokens: number;
  totalTokens: number;
  costUsd: number;

  // HTTP
  httpStatus?: number;
  requestId?: string;
}
```

---

## Wrapping `streamSimple` for telemetry

Create a thin wrapper that injects the hooks and emits your log entries:

```typescript
import { streamSimple } from "@mariozechner/pi-ai";
import type { Model, Context, SimpleStreamOptions, Api } from "@mariozechner/pi-ai";

export function observedStreamSimple<TApi extends Api>(
  model: Model<TApi>,
  context: Context,
  options?: SimpleStreamOptions & { callId?: string },
): ReturnType<typeof streamSimple> {
  const callId    = options?.callId ?? crypto.randomUUID();
  const startedAt = Date.now();
  let httpStatus: number | undefined;
  let requestId: string | undefined;
  let firstTokenAt: number | undefined;

  const stream = streamSimple(model, context, {
    ...options,

    onPayload: (payload, m) => {
      log({ event: "llm_request", callId, provider: m.provider, model: m.id, payload });
      return options?.onPayload?.(payload, m);
    },

    onResponse: (response, m) => {
      httpStatus = response.status;
      requestId  = response.headers["x-request-id"] ?? response.headers["request-id"];
      log({ event: "llm_response_headers", callId, httpStatus, requestId });
      return options?.onResponse?.(response, m);
    },
  });

  // Attach a side-channel listener for the first text_delta
  (async () => {
    for await (const event of stream) {
      if (!firstTokenAt && event.type === "text_delta") firstTokenAt = Date.now();
      if (event.type === "done" || event.type === "error") {
        const msg = event.type === "done" ? event.message : event.error;
        log({
          event:            "llm_done",
          callId,
          provider:         msg.provider,
          model:            msg.model,
          stopReason:       msg.stopReason,
          errorMessage:     msg.errorMessage,
          startedAt,
          firstTokenAt,
          completedAt:      Date.now(),
          inputTokens:      msg.usage.input,
          outputTokens:     msg.usage.output,
          cacheReadTokens:  msg.usage.cacheRead,
          cacheWriteTokens: msg.usage.cacheWrite,
          totalTokens:      msg.usage.totalTokens,
          costUsd:          msg.usage.cost.total,
          httpStatus,
          requestId,
        });
      }
    }
  })();

  return stream;
}

function log(entry: Record<string, unknown>): void {
  // Replace with your structured logger (pino, winston, etc.)
  console.log(JSON.stringify(entry));
}
```

The wrapper is transparent to the caller — it returns the same `AssistantMessageEventStream` and the `for await` loop works identically. The telemetry side-channel iterates the stream independently via the same async-iterable interface.

**Note**: `AssistantMessageEventStream` delivers each event to exactly one consumer. The wrapper above consumes the stream itself for telemetry, which means the calling code would not receive events. For production use, implement a proper fan-out or use `onPayload`/`onResponse` + `stream.result()` instead of iterating inside the wrapper.

See also: [extension-points.md](./extension-points.md) for `onPayload` and `onResponse` details. [features/streaming.md](./features/streaming.md) for the full event type reference.
