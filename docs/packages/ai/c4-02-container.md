# C4 Level 2 — Container

This diagram zooms into the `@mariozechner/pi-ai` package and shows the major sub-systems (containers) that exist inside it at runtime.

---

## Diagram

```mermaid
C4Container
  title pi-ai — Container View (Node.js process)

  Person(caller, "Caller", "Application code or higher-level pi package")

  Container_Boundary(piAiProcess, "Node.js Process (@mariozechner/pi-ai)") {

    Container(streamEngine, "Stream Engine", "src/stream.ts", "Entry point: stream(), streamSimple(), complete(), completeSimple(). Dispatches to the correct API provider.")

    Container(apiRegistry, "API Registry", "src/api-registry.ts", "Runtime map from Api string to ApiProvider. Supports register, unregister, and clear operations.")

    Container(providerRegistry, "Provider Implementations", "src/providers/", "One module per wire protocol: anthropic.ts, openai-completions.ts, google.ts, etc. Each exports stream() and streamSimple().")

    Container(modelRegistry, "Model Registry", "src/models.ts + src/models.generated.ts", "Typed catalogue of every supported model with cost, context window, and API assignment. Loaded at module import time.")

    Container(oauthModule, "OAuth Module", "src/utils/oauth/", "Login flows and token refresh for Anthropic, GitHub Copilot, Google Gemini CLI, Antigravity, OpenAI Codex.")

    Container(envKeys, "Env Key Resolver", "src/env-api-keys.ts", "Maps provider names to env var names (ANTHROPIC_API_KEY, etc.) and reads them from process.env.")

    Container(eventStream, "EventStream", "src/utils/event-stream.ts", "Generic async-iterable push/end queue. AssistantMessageEventStream specialises it for the LLM event protocol.")

    Container(cli, "CLI", "src/cli.ts", "Command-line interface for interactive model queries. Not required for library usage.")
  }

  System_Ext(providers, "LLM Provider APIs", "Anthropic, OpenAI, Google, Bedrock, etc.")
  System_Ext(oauthEndpoints, "OAuth Endpoints", "GitHub, Google, Anthropic, OpenAI")

  Rel(caller, streamEngine, "streamSimple(model, context, options)", "TypeScript")
  Rel(streamEngine, apiRegistry, "getApiProvider(model.api)")
  Rel(apiRegistry, providerRegistry, "Returns registered ApiProvider")
  Rel(streamEngine, eventStream, "Returns AssistantMessageEventStream")
  Rel(providerRegistry, envKeys, "getEnvApiKey(provider)")
  Rel(providerRegistry, eventStream, "push() events as they arrive")
  Rel(providerRegistry, providers, "HTTPS SSE / AWS SDK")
  Rel(oauthModule, oauthEndpoints, "PKCE OAuth 2.0")
  Rel(providerRegistry, modelRegistry, "Reads Model fields (cost, contextWindow)")
  Rel(caller, modelRegistry, "getModel(provider, modelId)")
  Rel(caller, oauthModule, "loginAnthropic(), loginGitHubCopilot(), etc.")
```

---

## Runtime interaction sequence

```mermaid
sequenceDiagram
  participant App as Application
  participant SE as Stream Engine
  participant AR as API Registry
  participant PV as Provider Implementation
  participant ES as EventStream
  participant API as LLM Provider API

  App->>SE: streamSimple(model, context, options)
  SE->>AR: getApiProvider(model.api)
  AR-->>SE: ApiProvider
  SE->>PV: provider.streamSimple(model, context, options)
  PV->>ES: new AssistantMessageEventStream()
  PV->>API: HTTPS SSE request (async, non-blocking)
  SE-->>App: returns EventStream immediately
  API-->>PV: SSE chunks
  PV->>ES: push({ type: "text_delta", ... })
  App->>ES: for await (const event of stream)
  ES-->>App: yields events as they arrive
  API-->>PV: [stream end]
  PV->>ES: push({ type: "done", message })
  PV->>ES: end()
  ES-->>App: stream exhausted
  App->>ES: stream.result()
  ES-->>App: AssistantMessage
```

---

## Container responsibilities

| Container | Source path | Responsibility |
|---|---|---|
| Stream Engine | `src/stream.ts` | Public entry points. Resolves provider, delegates, returns stream. |
| API Registry | `src/api-registry.ts` | Runtime map from `Api` string to `ApiProvider`. Supports dynamic registration. |
| Provider Implementations | `src/providers/*.ts` | Wire-protocol adapters. One per `KnownApi` value. Lazy-loaded on first use. |
| Model Registry | `src/models.ts` + `src/models.generated.ts` | Typed model catalogue. `getModel()`, `getModels()`, `calculateCost()`. |
| OAuth Module | `src/utils/oauth/` | PKCE login flows, token refresh, credential storage for subscription providers. |
| Env Key Resolver | `src/env-api-keys.ts` | Reads API keys from environment variables. Falls back to ambient credentials for Bedrock and Vertex. |
| EventStream | `src/utils/event-stream.ts` | Backpressure-safe async-iterable queue with a terminal result promise. |
| CLI | `src/cli.ts` | `pi-ai` binary for ad-hoc queries. Not used in library mode. |

---

## Browser compatibility

`pi-ai` is designed to work in modern browsers for providers that expose standard HTTPS APIs. Providers that rely on Node.js-only SDKs are **excluded from browser bundles**:

| Provider | Browser compatible? | Reason |
|---|---|---|
| Anthropic | Yes | Uses `fetch` + `@anthropic-ai/sdk` |
| OpenAI (completions / responses) | Yes | Uses `fetch` + `openai` SDK |
| Google Generative AI | Yes | Uses `fetch` + `@google/generative-ai` |
| Mistral | Yes | Uses `fetch` |
| AWS Bedrock | No | `@aws-sdk/client-bedrock-runtime` requires Node.js |
| Google Vertex | No | Requires `gcloud` ADC and Node.js filesystem |
| Google Gemini CLI | No | Requires filesystem OAuth token storage |

The lazy-loading architecture in `src/providers/register-builtins.ts` ensures that Node.js-only provider modules are never evaluated during browser bundle analysis.
