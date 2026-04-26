# Feature — Model Registry

The model registry is pi-ai's typed catalogue of every supported LLM. It lets you look up models by provider and ID with full TypeScript inference, calculate costs from usage data, and define your own custom models for self-hosted or unlisted endpoints.

---

## How `models.generated.ts` is produced

`src/models.generated.ts` is auto-generated. Do not edit it manually.

**To regenerate:**
```bash
npm run generate-models
```

This runs `packages/ai/scripts/generate-models.ts`, which:

1. Fetches model lists from each provider's public API or documentation source.
2. Maps each model to the `Model<TApi>` interface — normalising cost fields to $/million-tokens, filling context window and max token limits, and assigning the correct `api` string.
3. Writes the `MODELS` constant to `src/models.generated.ts`.

The generated file is a plain TypeScript object literal with `satisfies Model<TApi>` assertions that catch regressions at compile time.

---

## The `Model<TApi>` interface

```typescript
// src/types.ts
export interface Model<TApi extends Api> {
  id: string;           // provider's canonical model identifier, e.g. "claude-opus-4-5"
  name: string;         // human-readable display name, e.g. "Claude Opus 4.5"
  api: TApi;            // wire protocol — determines which provider implementation is used
  provider: Provider;   // company/service hosting the model, e.g. "anthropic"
  baseUrl: string;      // API endpoint base URL
  reasoning: boolean;   // true if the model supports extended thinking / chain-of-thought
  input: ("text" | "image")[];  // supported input modalities
  cost: {
    input: number;      // $/million input tokens
    output: number;     // $/million output tokens
    cacheRead: number;  // $/million cache-read tokens
    cacheWrite: number; // $/million cache-write tokens
  };
  contextWindow: number;  // max total tokens (input + output) per request
  maxTokens: number;      // max output tokens per response (0 = use provider default)
  headers?: Record<string, string>;  // provider-specific headers sent with every request
  compat?: ...;  // API-specific compatibility overrides (see OpenAICompletionsCompat, etc.)
}
```

### Field notes

- **`api` vs `provider`**: `api` picks the wire protocol (e.g., `"openai-completions"`). `provider` identifies who hosts it (e.g., `"groq"`). Many providers share the same `api` value because they speak the same protocol.
- **`reasoning: true`** means `options.reasoning` (a `ThinkingLevel`) is meaningful for this model. Setting it on a non-reasoning model is silently ignored by the provider.
- **`cost` fields use $/million tokens** as the unit. `calculateCost()` divides by 1,000,000 when computing actual dollar amounts.
- **`maxTokens: 0`** means the provider default is used. Most providers default this to their internal limit.

---

## Looking up a model programmatically

### By provider + model ID (fully typed)

```typescript
import { getModel } from "@mariozechner/pi-ai/models";

// TypeScript infers Model<"anthropic-messages"> from the literal arguments
const model = getModel("anthropic", "claude-opus-4-5");
console.log(model.contextWindow);  // 200000
console.log(model.cost.input);     // $/million input tokens
```

`getModel` performs two `Map` lookups and returns `undefined` if either provider or model ID is unknown. Both arguments are constrained to string literals present in `MODELS` — typos are caught at compile time.

### List all models for a provider

```typescript
import { getModels, getProviders } from "@mariozechner/pi-ai/models";

const providers = getProviders();              // KnownProvider[]
const anthropicModels = getModels("anthropic"); // Model<"anthropic-messages">[]

for (const m of anthropicModels) {
  console.log(m.id, m.cost.input, m.contextWindow);
}
```

### Filter by capability

```typescript
const reasoningModels = getModels("openai").filter(m => m.reasoning);
const visionModels    = getModels("google").filter(m => m.input.includes("image"));
```

---

## Defining a custom model

To target a self-hosted or unlisted endpoint, create a `Model` object directly. No registration step is needed — the model is passed directly to `streamSimple()`.

```typescript
import type { Model } from "@mariozechner/pi-ai";
import { streamSimple } from "@mariozechner/pi-ai";

// A self-hosted Llama 3 instance behind an OpenAI-compatible endpoint
const myModel: Model<"openai-completions"> = {
  id: "llama-3.3-70b-instruct",
  name: "Llama 3.3 70B Instruct (self-hosted)",
  api: "openai-completions",
  provider: "my-org",            // any string; used for display and auth lookup
  baseUrl: "https://llm.my-org.internal/v1",
  reasoning: false,
  input: ["text"],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 4096,
  compat: {
    supportsStore: false,
    supportsDeveloperRole: false,
    supportsReasoningEffort: false,
    supportsUsageInStreaming: true,
  },
};

const stream = streamSimple(myModel, context, { apiKey: "your-key" });
```

The `compat` field fine-tunes protocol behaviour for providers that don't support every OpenAI extension. See `OpenAICompletionsCompat` in `src/types.ts` for all available flags.

---

## Cost calculation from `Usage`

Every `AssistantMessage` carries a `Usage` object with raw token counts. Cost fields in `usage.cost` are pre-populated by the provider using `calculateCost()`:

```typescript
// src/models.ts
export function calculateCost<TApi extends Api>(model: Model<TApi>, usage: Usage): Usage["cost"] {
  usage.cost.input     = (model.cost.input     / 1_000_000) * usage.input;
  usage.cost.output    = (model.cost.output    / 1_000_000) * usage.output;
  usage.cost.cacheRead = (model.cost.cacheRead / 1_000_000) * usage.cacheRead;
  usage.cost.cacheWrite= (model.cost.cacheWrite/ 1_000_000) * usage.cacheWrite;
  usage.cost.total     = usage.cost.input + usage.cost.output
                       + usage.cost.cacheRead + usage.cost.cacheWrite;
  return usage.cost;
}
```

You can call this yourself to estimate cost from a partial usage object:

```typescript
import { calculateCost } from "@mariozechner/pi-ai/models";
import { getModel }      from "@mariozechner/pi-ai/models";

const model = getModel("anthropic", "claude-opus-4-5");
const fakeUsage = {
  input: 1000, output: 500, cacheRead: 0, cacheWrite: 0,
  totalTokens: 1500,
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
};
const cost = calculateCost(model, fakeUsage);
console.log(`Estimated cost: $${cost.total.toFixed(6)}`);
```

### Reading cost after a completed call

```typescript
const message = await completeSimple(model, context);
console.log("Input tokens:", message.usage.input);
console.log("Output tokens:", message.usage.output);
console.log("Cache read tokens:", message.usage.cacheRead);
console.log("Total cost USD:", message.usage.cost.total);
```

---

## Model equality

When you hold two model references and want to check if they refer to the same model:

```typescript
import { modelsAreEqual } from "@mariozechner/pi-ai/models";

if (!modelsAreEqual(previousModel, currentModel)) {
  console.log("Model changed, clearing context.");
}
```

`modelsAreEqual` compares both `id` and `provider` — two models with the same ID on different providers are not equal.

See also: [terminology.md](../terminology.md) for `Model`, `Api`, `Provider`, `Usage`. [extension-points.md](../extension-points.md) for registering custom providers.
