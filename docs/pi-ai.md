# `@mariozechner/pi-ai` — Unified LLM API

## What is it?

`pi-ai` is the lowest-level package in the pi-mono stack. It is a TypeScript library that provides a single, unified streaming API for interacting with large language models (LLMs) from over 20 different providers. Think of it as an adapter layer: you write one piece of code, and `pi-ai` talks to whichever provider (OpenAI, Anthropic, Google, AWS, etc.) you point it at.

**npm package:** `@mariozechner/pi-ai`  
**Version:** 0.70.2  
**Node requirement:** >= 20.0.0  
**License:** MIT

---

## The Problem It Solves

Every LLM provider ships its own SDK with its own:

- Authentication scheme (API keys, OAuth tokens, AWS credentials)
- Request/response format
- Streaming protocol (SSE, WebSocket, server-side events)
- Tool-calling (function calling) schema
- Model naming and discovery

A developer who wants to support multiple providers — or simply swap providers in the future — has to write a custom adapter for each one. `pi-ai` does that work once, exposing a clean, typed API surface that behaves identically regardless of the underlying provider.

---

## Audience

- TypeScript/Node.js developers building LLM-powered features
- Teams that want provider flexibility (e.g., use Claude in production but Gemini in CI)
- Developers building libraries or frameworks on top of LLMs (like `pi-agent-core` does)

---

## Supported Providers (at a glance)

- OpenAI, Azure OpenAI, OpenAI Codex (ChatGPT Plus/Pro via OAuth)
- Anthropic (API key and Claude Pro/Max via OAuth)
- Google Gemini, Vertex AI, Gemini CLI (OAuth)
- AWS Bedrock (Converse streaming)
- Mistral, Groq, Cerebras, xAI, DeepSeek, MiniMax
- OpenRouter, Vercel AI Gateway, Fireworks, Kimi For Coding
- GitHub Copilot (OAuth), Google Antigravity (OAuth)
- OpenCode Zen / Go
- Any OpenAI-compatible API: Ollama, vLLM, LM Studio, etc.

---

## Core Concepts

### Models and Providers

A **provider** is a service (e.g., `"anthropic"`). A **model** is a specific LLM offered by that provider (e.g., `"claude-sonnet-4-20250514"`). `pi-ai` maintains a generated model registry (`models.generated.ts`) built at compile time by querying provider APIs.

```typescript
import { getModel } from '@mariozechner/pi-ai';

// Fully typed — TypeScript autocompletes both provider and model names
const model = getModel('anthropic', 'claude-sonnet-4-20250514');
```

### Context

A `Context` is a plain, serializable object representing a conversation. It is the unit of state that travels between calls.

```typescript
interface Context {
  systemPrompt: string;
  messages: Message[];   // user, assistant, toolResult messages
  tools?: Tool[];        // function definitions the model can call
}
```

Because `Context` is a plain object, it can be serialized to JSON, stored in a database, and handed off to a different model mid-session (cross-provider hand-off).

### Streaming

The primary API is `stream()`, which returns an async iterable of typed events. Every event has a `type` field:

| Event type | Meaning |
|-----------|---------|
| `start` | Stream has begun; includes partial message |
| `text_start` | Model started emitting text |
| `text_delta` | A new chunk of text (call `process.stdout.write(event.delta)`) |
| `text_end` | Text block complete |
| `thinking_start/delta/end` | Model extended thinking (Anthropic, Google) |
| `toolcall_start/delta/end` | Model is calling a tool; args stream as partial JSON |
| `done` | Stream complete; includes stop reason |
| `error` | Provider returned an error |
| `usage` | Token counts and cost |

```typescript
const s = stream(model, context);
for await (const event of s) {
  if (event.type === 'text_delta') process.stdout.write(event.delta);
  if (event.type === 'done') console.log('Stop reason:', event.reason);
}
```

### Tools

Tools (function calling) use [TypeBox](https://github.com/sinclairzx81/typebox) schemas, which produce JSON Schema at runtime and TypeScript types at compile time.

```typescript
import { Type } from '@mariozechner/pi-ai';

const tools = [{
  name: 'read_file',
  description: 'Read a file from disk',
  parameters: Type.Object({
    path: Type.String({ description: 'Absolute file path' })
  })
}];
```

When the model calls a tool, you receive `toolcall_end` events containing the tool name and validated arguments. You execute the tool yourself and push a `toolResult` message back into the context before calling `stream()` again.

---

## Architecture and Source Layout

```
packages/ai/src/
├── index.ts              Re-exports public API
├── types.ts              All TypeScript interfaces (Model, Context, Message, Tool, events…)
├── stream.ts             Core stream() / complete() dispatcher
├── models.ts             Model registry helpers (getModel, listModels, etc.)
├── models.generated.ts   Auto-generated list of all known provider/model pairs
├── api-registry.ts       Maps Api strings to provider implementations
├── env-api-keys.ts       Reads API keys from environment variables
├── oauth.ts              OAuth flow helpers (device code, PKCE)
├── cli.ts                pi-ai CLI binary (login, list-models, etc.)
├── bedrock-provider.ts   Separate entry point for AWS Bedrock (heavy SDK)
└── providers/
    ├── anthropic.ts
    ├── openai-completions.ts
    ├── openai-responses.ts
    ├── google.ts
    ├── google-vertex.ts
    ├── google-gemini-cli.ts
    ├── mistral.ts
    ├── azure-openai-responses.ts
    ├── openai-codex-responses.ts
    ├── register-builtins.ts  Lazy-loads provider modules on demand
    └── …
```

### How a Call Flows Through the Code

1. **Consumer** calls `stream(model, context, options)`.
2. `stream.ts` looks up the model's `api` field (e.g., `"anthropic"`) in `api-registry.ts`.
3. The registry lazy-loads the matching provider module (e.g., `providers/anthropic.ts`) via `register-builtins.ts`.
4. The provider module:
   - Converts `Context.messages` into the provider's native format
   - Applies authentication (API key from env, OAuth token from `auth.json`, or caller-supplied key)
   - Opens a streaming HTTP connection to the provider API
   - Parses the provider's streaming format (SSE lines, JSON chunks, etc.)
   - Emits normalized events (`text_delta`, `toolcall_end`, `usage`, etc.)
5. `stream.ts` yields these events to the consumer.

No provider SDK is imported until it is actually needed (lazy loading), keeping startup time minimal.

---

## Cross-Provider Hand-offs

Because `Context` is plain data, you can switch models mid-conversation:

```typescript
const context: Context = { systemPrompt: '…', messages: [] };
// Start with GPT-4o
await runTurn(getModel('openai', 'gpt-4o'), context);
// Continue with Claude (messages carry over)
await runTurn(getModel('anthropic', 'claude-sonnet-4-20250514'), context);
```

`pi-ai` normalizes message formats so the hand-off is transparent.

---

## OAuth Providers

For providers backed by user subscriptions (Claude Pro, ChatGPT Plus, GitHub Copilot, Gemini CLI), `pi-ai` implements OAuth device-code and PKCE flows. Tokens are stored in `~/.pi/agent/auth.json`. The `pi-ai` CLI exposes a `login` subcommand that drives the flow in a terminal.

---

## Token and Cost Tracking

Every completed stream emits a `usage` event:

```typescript
{ type: 'usage', inputTokens: 120, outputTokens: 45, cost: 0.000217 }
```

Cost calculations use the per-model pricing data baked into `models.generated.ts`.

---

## Browser Compatibility

`pi-ai` is designed to work in browsers. It avoids Node.js-only APIs in the hot path, uses `undici` only where needed (for HTTP/2 or proxy support on Node), and falls back to `fetch` in browser contexts. The Bedrock provider is a separate entry point because the AWS SDK is Node.js-only.

---

## Key Dependencies

| Dependency | Purpose |
|-----------|---------|
| `@anthropic-ai/sdk` | Anthropic API client |
| `openai` | OpenAI / OpenAI-compatible API client |
| `@google/genai` | Google Gemini client |
| `@mistralai/mistralai` | Mistral client |
| `@aws-sdk/client-bedrock-runtime` | AWS Bedrock (separate entry point) |
| `typebox` | JSON Schema / TypeScript type generation for tools |
| `partial-json` | Parse streaming partial JSON for tool arguments |
| `proxy-agent` | HTTP/HTTPS proxy support |

---

## Pros and Cons

**Pros**
- Single API for 20+ providers — swap without rewriting call sites
- Type-safe model and provider names (autocomplete in IDE)
- Built-in OAuth for subscription accounts
- Streaming-first design; `complete()` is a thin wrapper over `stream()`
- Zero tree-shaking friction (providers are lazy-loaded)
- Browser-compatible

**Cons**
- Tool-calling models only — non-tool models are excluded by design
- Advanced per-provider options require knowing provider-specific option types
- Model list is generated at build time; very new models may not appear until next release
- AWS Bedrock requires a separate import due to heavy SDK size

---

## How to Use It (Junior Developer Walkthrough)

1. **Install:**
   ```bash
   npm install @mariozechner/pi-ai
   ```

2. **Set your API key** (e.g., for Anthropic):
   ```bash
   export ANTHROPIC_API_KEY=sk-ant-...
   ```

3. **Write your first stream:**
   ```typescript
   import { getModel, stream } from '@mariozechner/pi-ai';

   const model = getModel('anthropic', 'claude-haiku-4-5');
   const context = {
     systemPrompt: 'You are a helpful assistant.',
     messages: [{ role: 'user', content: 'What is 2 + 2?' }],
   };

   for await (const event of stream(model, context)) {
     if (event.type === 'text_delta') process.stdout.write(event.delta);
   }
   ```

4. **Add tool calling:** Define a tool with `Type.Object(...)`, include it in `context.tools`, and loop: stream → handle `toolcall_end` → push `toolResult` message → stream again.

5. **Switch providers:** Change the first argument to `getModel`. Everything else stays the same.
