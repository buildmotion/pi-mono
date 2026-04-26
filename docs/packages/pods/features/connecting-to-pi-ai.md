# Feature: Connecting to pi-ai

## Learning Objectives

- Create a `Model` object that points at your pod's vLLM endpoint.
- Call `streamSimple()` against a self-hosted model.
- Run a full agent session using a pod as the LLM backend.

---

## Background

vLLM exposes an OpenAI-compatible HTTP API. `pi-ai` supports any OpenAI-compatible endpoint via the `openai-completions` API identifier and a custom `baseUrl`. You define a `Model` object, and everything else — streaming, tool calls, usage tracking — works identically to a cloud model.

---

## Step 1 — Define a Custom Model

```typescript
import type { Model } from "@mariozechner/pi-ai";

const qwen72b: Model<"openai-completions"> = {
  id: "Qwen/Qwen2.5-72B-Instruct",
  name: "Qwen 2.5 72B (self-hosted)",
  api: "openai-completions",
  provider: "pods",
  baseUrl: "http://203.0.113.10:8000/v1",
  contextWindow: 131072,
  maxTokens: 4096,
  input: 0,     // cost per token — 0 for self-hosted
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
};
```

Set `baseUrl` to your pod's IP and port.

---

## Step 2 — Stream a Response

```typescript
import { streamSimple } from "@mariozechner/pi-ai";

const context = {
  messages: [{ role: "user", content: [{ type: "text", text: "Explain vLLM in one sentence." }] }],
};

const stream = streamSimple(qwen72b, context, {
  apiKey: process.env.PI_API_KEY, // the key you set during pi pods setup
  maxTokens: 256,
});

for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
```

---

## Step 3 — Use with an Agent

The pod model works as a drop-in replacement for any cloud model:

```typescript
import { Agent } from "@mariozechner/pi-agent-core";

const agent = new Agent({
  initialState: {
    model: qwen72b,
    systemPrompt: "You are a helpful assistant.",
  },
  getApiKey: async () => process.env.PI_API_KEY,
});

await agent.prompt("What files are in the current directory?");
```

---

## Step 4 — Using SSH Tunnel for Private Pods

If your pod is not publicly accessible, create an SSH tunnel:

```bash
ssh -L 8000:localhost:8000 ubuntu@<pod-ip> -N &
```

Then use `baseUrl: "http://localhost:8000/v1"` in your `Model` definition.

---

## Check-Yourself Questions

1. Which `api` identifier does pi-ai use for OpenAI-compatible endpoints?
2. What should you set `cost.input` and `cost.output` to for a self-hosted model?
3. How do you authenticate requests to the vLLM endpoint?
4. Why would you use an SSH tunnel instead of the pod's public IP?
5. Does changing from a cloud model to a pod model require any changes to the agent loop or tool configuration?
