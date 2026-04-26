# Feature — Provider Authentication

`pi-ai` supports four authentication mechanisms. They apply in the following priority order within any given call:

1. `options.apiKey` — explicit key passed per call
2. Environment variable — read by `getEnvApiKey(provider)` at call time
3. OAuth token — stored in `auth.json` on disk, refreshed automatically
4. Ambient credentials — AWS credential chain and Google ADC (no key required)

---

## API key via environment variable

Set the provider-specific env var before starting your process. `pi-ai` reads it automatically on every call via `src/env-api-keys.ts`.

| Provider | Environment variable(s) |
|---|---|
| Anthropic | `ANTHROPIC_OAUTH_TOKEN` (preferred), `ANTHROPIC_API_KEY` |
| OpenAI | `OPENAI_API_KEY` |
| Azure OpenAI | `AZURE_OPENAI_API_KEY` |
| Google Generative AI | `GEMINI_API_KEY` |
| Google Vertex | `GOOGLE_CLOUD_API_KEY` |
| AWS Bedrock | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`, `AWS_PROFILE`, or others (see below) |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN` |
| DeepSeek | `DEEPSEEK_API_KEY` |
| xAI | `XAI_API_KEY` |
| Groq | `GROQ_API_KEY` |
| Cerebras | `CEREBRAS_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` |
| Mistral | `MISTRAL_API_KEY` |
| MiniMax | `MINIMAX_API_KEY` |
| MiniMax CN | `MINIMAX_CN_API_KEY` |
| HuggingFace | `HF_TOKEN` |
| Fireworks | `FIREWORKS_API_KEY` |
| Kimi Coding | `KIMI_API_KEY` |

```bash
# Example — set before running your script
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
```

How it works internally:

```typescript
// src/env-api-keys.ts
export function getEnvApiKey(provider: string): string | undefined {
  const envKeys = findEnvKeys(provider);   // maps provider name → env var names
  if (envKeys?.[0]) return process.env[envKeys[0]];
  // ... Vertex ADC and Bedrock ambient credential checks
}
```

---

## API key via `options.apiKey`

Pass the key directly when calling `streamSimple()` or `completeSimple()`. This overrides the environment variable for that call only.

```typescript
const stream = streamSimple(model, context, {
  apiKey: "sk-ant-your-runtime-key",
});
```

Use this pattern when your application manages multiple tenants with different keys, or when you retrieve keys from a secrets manager at call time.

---

## OAuth flows (subscription accounts)

Some providers grant access via OAuth rather than static API keys. Supported OAuth providers:

| Provider | OAuth provider ID | Covers |
|---|---|---|
| Anthropic | `"anthropic"` | Claude Pro / Max subscription |
| GitHub Copilot | `"github-copilot"` | Copilot Business / Enterprise subscription |
| Google Gemini CLI | `"gemini-cli"` | Google Cloud Code Assist |
| Google Antigravity | `"google-antigravity"` | Gemini 3 / Claude / GPT-OSS via Google Cloud |
| OpenAI Codex | `"openai-codex"` | ChatGPT subscription |

### Login flow (interactive)

```typescript
import { loginAnthropic, loginGitHubCopilot } from "@mariozechner/pi-ai/oauth";

// Opens a browser and performs PKCE OAuth. Stores credentials in auth.json.
await loginAnthropic();
await loginGitHubCopilot();
```

The login function:
1. Opens a local HTTP server on `127.0.0.1` for the redirect callback.
2. Launches the browser to the provider's authorization URL.
3. Receives the authorization code, exchanges it for tokens via PKCE.
4. Stores the token and expiry in `auth.json` (path managed by the calling application).

### Token refresh

```typescript
import { getOAuthApiKey } from "@mariozechner/pi-ai/oauth";

const result = await getOAuthApiKey("anthropic", credentials);
if (result) {
  // result.apiKey is a fresh bearer token, ready for options.apiKey
  // result.newCredentials has updated expiry — store it back to auth.json
  const stream = streamSimple(model, context, { apiKey: result.apiKey });
}
```

`getOAuthApiKey` automatically refreshes if `Date.now() >= credentials.expires`. The coding agent uses a `getApiKey` callback in `AgentLoopConfig` to feed fresh tokens into every LLM call without the caller needing to manage this explicitly.

### Custom OAuth provider

```typescript
import { registerOAuthProvider } from "@mariozechner/pi-ai/oauth";

registerOAuthProvider({
  id: "my-provider",
  name: "My Provider",
  login: async (options) => { /* open browser, exchange code */ },
  refreshToken: async (credentials) => { /* refresh and return new OAuthCredentials */ },
  getApiKey: (credentials) => credentials.accessToken,
});
```

---

## AWS Bedrock — ambient credential chain

Bedrock does not use API keys. Instead, `pi-ai` delegates entirely to the AWS SDK credential resolution chain. Set any of the following:

| Method | Environment variables or files |
|---|---|
| IAM access keys | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` (+ optional `AWS_SESSION_TOKEN`) |
| Named profile | `AWS_PROFILE` (reads `~/.aws/credentials`) |
| Bedrock bearer token | `AWS_BEARER_TOKEN_BEDROCK` |
| ECS task role | `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` or `AWS_CONTAINER_CREDENTIALS_FULL_URI` |
| IRSA (Kubernetes) | `AWS_WEB_IDENTITY_TOKEN_FILE` |

`getEnvApiKey("amazon-bedrock")` returns the sentinel string `"<authenticated>"` when any of these are set — this is pi-ai's signal that Bedrock is configured even though no literal API key string exists.

---

## Google Vertex — Application Default Credentials

```bash
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT="my-project"
export GOOGLE_CLOUD_LOCATION="us-central1"
```

`pi-ai` checks for `~/.config/gcloud/application_default_credentials.json` (or `GOOGLE_APPLICATION_CREDENTIALS` env var) and returns `"<authenticated>"` when ADC credentials, project, and location are all configured.

---

## Custom HTTP headers

Add headers to every request to a provider — useful for enterprise proxies, API gateways, or observability middleware:

```typescript
const stream = streamSimple(model, context, {
  headers: {
    "X-Request-Id": requestId,
    "X-Tenant-Id": tenantId,
  },
});
```

Custom headers are merged with provider defaults. Provider-required headers (e.g., `anthropic-version`) always take precedence. Not supported by the AWS Bedrock provider (which uses SDK-level auth, not HTTP headers).

---

## `getApiKey` callback pattern (agent usage)

The coding agent and other higher-level packages use a `getApiKey` callback in `AgentLoopConfig` to supply a fresh token before each LLM call. This avoids hardcoding tokens into long-lived state:

```typescript
// Pattern used by the agent layer
const agentConfig = {
  getApiKey: async (provider: string) => {
    const result = await getOAuthApiKey(provider, loadCredentials());
    if (result) {
      saveCredentials(result.newCredentials);
      return result.apiKey;
    }
    return getEnvApiKey(provider);  // fall back to env var
  },
};
```

See also: [terminology.md](../terminology.md), [extension-points.md](../extension-points.md).
