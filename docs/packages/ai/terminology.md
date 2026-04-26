# Terminology — `@mariozechner/pi-ai`

This is the pi-ai–specific vocabulary reference. For cross-package terms (Agent Loop, Steering Message, C4 Model, etc.) see the monorepo [glossary](../../glossary.md).

---

## Api

`Api` is a string identifier for a **wire protocol** — the specific HTTP request/response format used to talk to an LLM endpoint. It is distinct from the provider (the company) and the model (the specific checkpoint).

```typescript
// src/types.ts
type KnownApi =
  | "openai-completions"       // OpenAI Chat Completions, used by many providers
  | "mistral-conversations"    // Mistral's native API
  | "openai-responses"         // OpenAI Responses API (newer, stateful)
  | "azure-openai-responses"   // Azure-specific Responses variant
  | "openai-codex-responses"   // ChatGPT OAuth via Responses API
  | "anthropic-messages"       // Anthropic Messages API
  | "bedrock-converse-stream"  // AWS Bedrock Converse streaming
  | "google-generative-ai"     // Google Gemini via @google/generative-ai
  | "google-gemini-cli"        // Gemini via Google Cloud Code Assist OAuth
  | "google-vertex";           // Google Vertex AI

type Api = KnownApi | (string & {});  // open for custom protocols
```

`model.api` is the dispatch key that the API Registry uses to select the correct provider implementation.

---

## Provider

`Provider` is a string identifier for the **company or service** hosting an LLM. Examples: `"anthropic"`, `"openai"`, `"google"`, `"amazon-bedrock"`, `"groq"`.

`Provider` is used for authentication lookup (`getEnvApiKey(provider)`) and display. Multiple providers can share the same `Api` value — for example Groq, Cerebras, and OpenAI all use `"openai-completions"`.

```typescript
type KnownProvider = "amazon-bedrock" | "anthropic" | "google" | "openai" | "groq" | ... | string;
type Provider = KnownProvider | string;
```

---

## Model

A `Model<TApi>` is a typed descriptor for one deployable LLM checkpoint. It holds everything needed to make a call: the provider endpoint, cost rates, capability flags, and the wire protocol assignment.

```typescript
interface Model<TApi extends Api> {
  id: string;             // provider's canonical model id
  name: string;           // human-readable name
  api: TApi;              // wire protocol (dispatch key)
  provider: Provider;     // hosting company
  baseUrl: string;        // API endpoint base URL
  reasoning: boolean;     // supports extended thinking
  input: ("text" | "image")[];
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number }; // $/million tokens
  contextWindow: number;  // max total tokens per request
  maxTokens: number;      // max output tokens per response
  headers?: Record<string, string>;
  compat?: ...;           // API-specific compatibility flags
}
```

See [features/model-registry.md](./features/model-registry.md) for registry functions.

---

## Context

`Context` is the snapshot handed to the LLM on every call. It groups the system prompt, the conversation history, and the tool definitions.

```typescript
interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

Do not confuse this with the broader notion of "context window" (the token limit). `Context` is a data structure; context window is a hardware constraint.

---

## AssistantMessageEvent

`AssistantMessageEvent` is a discriminated union of every event that a provider emits during streaming. Consumers pattern-match on `event.type`.

```typescript
type AssistantMessageEvent =
  | { type: "start";          partial: AssistantMessage }
  | { type: "text_start";     contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta";     contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end";       contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end";   contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end";   contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "done";  reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
  | { type: "error"; reason: "aborted" | "error"; error: AssistantMessage };
```

Every non-terminal event includes a `partial` field — the in-progress `AssistantMessage` built up so far, suitable for live rendering. The stream always starts with `start` and ends with exactly one `done` or `error`.

---

## EventStream

`EventStream<T, R>` is the generic async-iterable push queue that backs all streaming in pi-ai.

- **Producer** calls `push(event)` to enqueue and `end()` to signal completion.
- **Consumer** iterates with `for await (const event of stream)`.
- `stream.result()` returns a `Promise<R>` that resolves on the terminal event.

`AssistantMessageEventStream` specialises `EventStream<AssistantMessageEvent, AssistantMessage>` and treats `"done"` and `"error"` as terminal events. Source: `src/utils/event-stream.ts`.

---

## stream / streamSimple

`stream()` is the low-level entry point. It accepts provider-specific options typed to the exact `Api`.

`streamSimple()` is the high-level entry point. It accepts the normalised `SimpleStreamOptions` (which includes `reasoning: ThinkingLevel`) and internally translates them to provider-specific options. This is the function most callers should use.

Both return an `AssistantMessageEventStream` immediately (before the HTTP request completes).

```typescript
// src/stream.ts
function stream<TApi extends Api>(model, context, options?: ProviderStreamOptions): AssistantMessageEventStream
function streamSimple<TApi extends Api>(model, context, options?: SimpleStreamOptions): AssistantMessageEventStream
```

---

## complete / completeSimple

Convenience wrappers that call `stream()` or `streamSimple()` and await `stream.result()`. They return a `Promise<AssistantMessage>` and discard all intermediate events.

Use `completeSimple()` when you only need the final message and do not need incremental rendering.

---

## StopReason

`StopReason` is the terminal condition of an assistant message:

| Value | Meaning |
|---|---|
| `"stop"` | Model finished naturally |
| `"length"` | Response truncated at `maxTokens` |
| `"toolUse"` | Model emitted one or more tool calls |
| `"error"` | Unrecoverable error (network, invalid response, etc.) |
| `"aborted"` | Caller cancelled via `AbortSignal` |

Source: `src/types.ts`.

---

## ThinkingLevel

`ThinkingLevel` controls how much token budget is allocated for extended thinking (chain-of-thought reasoning) on models that support it.

```typescript
type ThinkingLevel = "minimal" | "low" | "medium" | "high" | "xhigh";
```

Default budgets: `minimal` = 1,024, `low` = 2,048, `medium` = 8,192, `high` = 16,384 tokens. `xhigh` maps to provider-specific "max effort" modes (supported by GPT-5.2/5.3/5.4, DeepSeek V4 Pro, and Claude Opus 4.6+). Budgets can be overridden via `options.thinkingBudgets`. Source: `src/types.ts`, `src/providers/simple-options.ts`.

---

## CacheRetention

`CacheRetention` controls how aggressively pi-ai requests prompt caching from providers that support it.

```typescript
type CacheRetention = "none" | "short" | "long";
```

- `"none"` — no cache-control markers sent.
- `"short"` — default; provider ephemeral cache (Anthropic: ~5 minutes TTL).
- `"long"` — provider long-retention cache where available (Anthropic: 1 hour TTL via `cache_control.ttl: "1h"`).

Not all providers honour all retention values; unsupported values fall back to `"short"`.

---

## Transport

`Transport` hints at the preferred connection type for providers that support multiple transports.

```typescript
type Transport = "sse" | "websocket" | "auto";
```

Most providers use SSE only. `"auto"` lets the provider choose.

---

## Usage

`Usage` is the structured token and cost summary attached to every `AssistantMessage`.

```typescript
interface Usage {
  input: number;        // prompt tokens consumed
  output: number;       // completion tokens generated
  cacheRead: number;    // tokens served from provider cache
  cacheWrite: number;   // tokens written to provider cache
  totalTokens: number;  // input + output
  cost: {
    input: number;      // USD
    output: number;     // USD
    cacheRead: number;  // USD
    cacheWrite: number; // USD
    total: number;      // USD sum
  };
}
```

Cost fields are computed by `calculateCost(model, usage)` in `src/models.ts` using the $/million-token rates from the `Model.cost` field.

---

## ToolCall

`ToolCall` is a structured request inside an `AssistantMessage` for the caller to invoke a named function.

```typescript
interface ToolCall {
  type: "toolCall";
  id: string;                       // provider-generated unique ID
  name: string;                     // name of the tool to invoke
  arguments: Record<string, any>;  // parsed JSON arguments
  thoughtSignature?: string;        // Google-specific: opaque signature for thought reuse
}
```

After receiving a `done` event with `reason: "toolUse"`, the caller executes the tool(s) and appends a `ToolResultMessage` for each call. See the monorepo [glossary](../../glossary.md#tool-call) for the broader agent context.

---

## ToolResultMessage

`ToolResultMessage` carries the output of a tool execution back to the model.

```typescript
interface ToolResultMessage<TDetails = any> {
  role: "toolResult";
  toolCallId: string;                        // links to ToolCall.id
  toolName: string;                          // name of the tool that was called
  content: (TextContent | ImageContent)[];  // tool output (text and/or images)
  details?: TDetails;                        // arbitrary structured data for UI rendering
  isError: boolean;                          // true if the tool execution failed
  timestamp: number;                         // Unix ms
}
```

---

## AssistantMessage

`AssistantMessage` is the complete, final output from one LLM call.

```typescript
interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api;
  provider: Provider;
  model: string;                  // model ID as returned by the provider
  responseId?: string;            // provider-specific response identifier
  usage: Usage;
  stopReason: StopReason;
  errorMessage?: string;          // set when stopReason is "error" or "aborted"
  timestamp: number;              // Unix ms
}
```

---

## UserMessage

`UserMessage` is a turn from the human side of the conversation.

```typescript
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;  // Unix ms
}
```

Content can be a plain string (shorthand) or an array of content blocks mixing text and images.
