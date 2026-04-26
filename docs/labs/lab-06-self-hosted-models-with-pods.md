# Lab 06: Deploying a Self-Hosted LLM on a GPU Pod with `pi-pods`

When you reach the limits of the hosted API — cost at scale, privacy requirements, fine-tuned weights, or models not available through any provider — you need to run the model yourself. This lab provisions a GPU instance, deploys a model with vLLM, and connects it to the `pi-ai` streaming API so every lab you have completed so far can use it without modification.

## Learning Objectives

- Provision a GPU pod and install the vLLM inference server using `pi pods setup`
- Deploy a large language model with `pi pods start`
- Verify the OpenAI-compatible endpoint with `curl`
- Connect the self-hosted endpoint to `pi-ai` as a custom `Model`
- Run a full multi-turn agent session against your own hardware
- Measure VRAM usage and understand tensor parallelism

## Prerequisites

- An account on [DataCrunch](https://datacrunch.io/) or [RunPod](https://www.runpod.io/) with GPU instances available
- An SSH key pair (`ssh-keygen -t ed25519 -C "lab-06"` if you need one)
- Node.js 20 or later
- `curl` available in your shell

---

## Background

### vLLM

[vLLM](https://docs.vllm.ai/) is a high-throughput inference engine for LLMs. Key properties:

- **Continuous batching**: handles multiple concurrent requests without stalling
- **PagedAttention**: manages KV-cache memory efficiently, dramatically reducing VRAM waste
- **OpenAI-compatible API**: exposes `/v1/chat/completions` and `/v1/models` endpoints, so any client written for OpenAI works out of the box
- **Tensor parallelism**: splits a model's weight matrices across multiple GPUs using `--tensor-parallel-size N`

### Why Self-Hosting?

| Concern | Hosted API | Self-Hosted |
|---|---|---|
| Per-token cost | Billed per token | GPU instance flat rate |
| Privacy | Data sent to provider | Stays on your hardware |
| Model choice | Provider's catalogue | Any HuggingFace model |
| Fine-tuned weights | Not supported | Full control |
| Rate limits | Provider-imposed | None |
| Cold-start latency | None | Model load time (~2 min) |

### OpenAI-Compatible Endpoints

vLLM's API surface is a superset of the OpenAI Chat Completions API. This means:
- Any client that speaks the OpenAI protocol works without modification
- `pi-ai` can connect by setting a custom `baseUrl` on a `Model` object
- Streaming, tool calls, and system prompts all behave identically

---

## Step 1: Install `pi-pods` Globally

```bash
npm install -g @mariozechner/pi
```

Verify:

```bash
pi --version
pi pods --help
```

You should see the `pods` subcommand listed with `setup`, `start`, `stop`, and `list` options.

---

## Step 2: Configure Pod Credentials

`pi pods` needs SSH access to the GPU instance and your cloud provider credentials to manage the lifecycle of pods.

**For DataCrunch:**

```bash
export DATACRUNCH_CLIENT_ID="your-client-id"
export DATACRUNCH_CLIENT_SECRET="your-client-secret"
```

**For RunPod:**

```bash
export RUNPOD_API_KEY="your-api-key"
```

Store your SSH private key in the default location (`~/.ssh/id_ed25519`) or specify it explicitly:

```bash
export PI_SSH_KEY="$HOME/.ssh/lab-06"
```

Test SSH access manually before proceeding. When `pi pods setup` runs, it connects over SSH to install software. If SSH fails, the setup will fail with an unclear error.

```bash
ssh -i ~/.ssh/lab-06 ubuntu@<pod-ip> "echo connected"
```

---

## Step 3: Provision a GPU Instance and Run `pi pods setup`

First, create a GPU pod through your provider's web UI:

- **Minimum spec for a 7B model**: 1× A10G (24 GB VRAM) or 1× RTX 4090 (24 GB VRAM)
- **For a 70B model**: 4× A100 (80 GB each) with NVLink
- **OS image**: Ubuntu 22.04 with CUDA 12.1+ pre-installed (most providers have a template)

Note the pod's public IP address. Then run:

```bash
pi pods setup <pod-ip>
```

This command connects over SSH and:
1. Updates system packages
2. Installs the CUDA toolkit (if not already present)
3. Installs Python 3.11 and pip
4. Creates a Python virtual environment at `/opt/vllm`
5. Installs `vllm` and its dependencies into the venv
6. Installs `huggingface_hub` for model downloads

Setup takes approximately 5–15 minutes depending on the instance's network speed and whether CUDA was pre-installed.

```
[pods] Connecting to <pod-ip>...
[pods] Installing system dependencies...
[pods] Creating Python venv at /opt/vllm...
[pods] Installing vllm (this takes a while)...
[pods] Setup complete.
```

---

## Step 4: Start a Model with `pi pods start`

Download and serve a model. Start with a 7B or 8B model — it fits on a single 24 GB GPU without tensor parallelism.

```bash
pi pods start <pod-ip> qwen2.5-7b-instruct
```

`pi pods start` knows a set of well-tested model identifiers and maps them to their HuggingFace repo paths. Under the hood it runs:

```bash
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-7B-Instruct \
  --host 0.0.0.0 \
  --port 8000 \
  --served-model-name qwen2.5-7b-instruct \
  --trust-remote-code
```

The first start downloads the model weights from HuggingFace. A 7B model is roughly 14 GB — expect 5–20 minutes depending on bandwidth. Subsequent starts use the cached weights and take about 90 seconds.

Watch the output for the ready line:

```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

For a 70B model on 4 GPUs, add the tensor parallelism flag:

```bash
pi pods start <pod-ip> qwen2.5-72b --tensor-parallel 4
```

---

## Step 5: Query the OpenAI-Compatible Endpoint with `curl`

```bash
POD_IP="<your-pod-ip>"

# List available models
curl -s "http://${POD_IP}:8000/v1/models" | python3 -m json.tool

# Send a chat completion request
curl -s "http://${POD_IP}:8000/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-7b-instruct",
    "messages": [
      {"role": "system", "content": "You are a concise assistant."},
      {"role": "user", "content": "What is tensor parallelism in one sentence?"}
    ],
    "stream": false,
    "max_tokens": 100
  }' | python3 -m json.tool
```

Expected response shape:

```json
{
  "id": "cmpl-abc123",
  "object": "chat.completion",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Tensor parallelism splits..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 32,
    "completion_tokens": 18,
    "total_tokens": 50
  }
}
```

Now test streaming:

```bash
curl -s "http://${POD_IP}:8000/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2.5-7b-instruct",
    "messages": [{"role": "user", "content": "Count from 1 to 10."}],
    "stream": true,
    "max_tokens": 50
  }'
```

You should see a sequence of `data: {"choices":[{"delta":{"content":"1"}},...]}` lines arriving incrementally.

---

## Step 6: Connect the Endpoint to `pi-ai` as a Custom Provider

`pi-ai` models can point at any OpenAI-compatible endpoint by setting `baseUrl` and `api: "openai-chat"`:

```typescript
import { stream } from "@mariozechner/pi-ai";
import type { Model, Context } from "@mariozechner/pi-ai";

const POD_IP = process.env.POD_IP ?? "localhost";

const selfHostedModel: Model = {
  id: "qwen2.5-7b-instruct",
  name: "Qwen 2.5 7B Instruct (self-hosted)",
  provider: "self-hosted",
  api: "openai-chat",
  // Point at your pod
  baseUrl: `http://${POD_IP}:8000/v1`,
  contextWindow: 131072,
  maxTokens: 4096,
  // No cost tracking for self-hosted (you pay a flat rate for the instance)
  cost: { input: 0, output: 0 },
};

const context: Context = {
  systemPrompt: "You are a helpful assistant.",
  messages: [
    { role: "user", content: "Explain the PagedAttention algorithm briefly." },
  ],
};

for await (const event of stream(selfHostedModel, context)) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
  if (event.type === "done") console.log("\n\nUsage:", event.usage);
  if (event.type === "error") console.error("Error:", event.error);
}
```

Run it:

```bash
POD_IP=<your-pod-ip> npx tsx query.ts
```

The response should stream from your GPU pod through your local machine's terminal — the same event model as any hosted provider.

---

## Step 7: Run a Full Agent Session Against Your Self-Hosted Model

Substitute the self-hosted `Model` object into the agent from Lab 02:

```typescript
import { Agent } from "@mariozechner/pi-agent-core";
import { Type } from "@sinclair/typebox";
import type { AgentTool } from "@mariozechner/pi-agent-core";
import type { Model } from "@mariozechner/pi-ai";

const POD_IP = process.env.POD_IP ?? "localhost";

const selfHostedModel: Model = {
  id: "qwen2.5-7b-instruct",
  name: "Qwen 2.5 7B Instruct (self-hosted)",
  provider: "self-hosted",
  api: "openai-chat",
  baseUrl: `http://${POD_IP}:8000/v1`,
  contextWindow: 131072,
  maxTokens: 4096,
  cost: { input: 0, output: 0 },
};

const calculatorTool: AgentTool = {
  name: "calculate",
  label: "Calculator",
  description: "Evaluates a mathematical expression and returns the result.",
  parameters: Type.Object({
    expression: Type.String({ description: "A JavaScript math expression, e.g. '2 ** 10'" }),
  }),
  execute: async (_id, params) => {
    // eslint-disable-next-line no-new-func
    const result = new Function(`return (${params.expression})`)();
    return {
      content: [{ type: "text", text: String(result) }],
      details: { expression: params.expression, result },
    };
  },
};

const agent = new Agent({
  model: selfHostedModel,
  systemPrompt: "You are a helpful math assistant. Use the calculator tool for arithmetic.",
  tools: [calculatorTool],
});

agent.subscribe((event) => {
  if (event.type === "message_update" && event.message.content) {
    process.stdout.write("\r" + event.message.content.slice(-80));
  }
  if (event.type === "tool_execution_start") {
    console.log(`\n[tool] ${event.toolCall.name}(${JSON.stringify(event.toolCall.input)})`);
  }
  if (event.type === "agent_end") {
    console.log("\n\nFinal:", event.message.content);
  }
});

await agent.prompt("What is 2 to the power of 32, and what is that divided by 1024?");
```

Run and verify that the self-hosted model calls `calculate` twice and produces the correct final answer.

> **Troubleshooting:** If the self-hosted model does not call tools, try a more explicit system prompt: `"When asked to calculate, you MUST call the calculate tool."` Smaller models are less reliable at tool-calling than the large hosted models. Qwen 2.5 7B is well-tuned for this; other 7B models may need prompt engineering.

---

## Step 8 (Stretch): Start a Second Model and Measure VRAM Usage

With a large GPU (e.g., 80 GB A100), you can run two models simultaneously:

```bash
# Start the first model on port 8000
pi pods start <pod-ip> qwen2.5-7b-instruct --port 8000

# SSH in and start the second model manually on a different port
ssh ubuntu@<pod-ip>
source /opt/vllm/bin/activate
python -m vllm.entrypoints.openai.api_server \
  --model mistralai/Mistral-7B-Instruct-v0.3 \
  --host 0.0.0.0 \
  --port 8001 \
  --served-model-name mistral-7b-instruct &
```

Check VRAM allocation:

```bash
nvidia-smi --query-gpu=name,memory.used,memory.free,memory.total --format=csv
```

Expected output for two 7B models on a 80 GB A100:

```
name, memory.used [MiB], memory.free [MiB], memory.total [MiB]
A100-SXM4-80GB, 29000 MiB, 50000 MiB, 81920 MiB
```

Each 7B model at float16 precision consumes approximately 14–15 GB of VRAM. The KV-cache for concurrent requests takes additional memory.

Now query both models and compare output quality on the same prompt:

```bash
for PORT in 8000 8001; do
  echo "=== Port $PORT ==="
  curl -s "http://${POD_IP}:${PORT}/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -d '{"model":"'$(curl -s http://${POD_IP}:${PORT}/v1/models | python3 -c "import sys,json; print(json.load(sys.stdin)[\"data\"][0][\"id\"])")'","messages":[{"role":"user","content":"Explain backpropagation in one paragraph."}],"max_tokens":150}' \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
done
```

---

## Expected Outcomes

By the end of this lab you should have:

- A GPU pod provisioned and configured with vLLM via `pi pods setup`
- A running model served on an OpenAI-compatible endpoint
- Verified the endpoint manually with `curl`
- Integrated the endpoint into `pi-ai` using a custom `Model` object
- Run a complete agent session with tool calls against self-hosted hardware
- (Stretch) Measured VRAM usage across two simultaneous models

---

## Check Yourself

1. A 7B model at float16 precision uses roughly 14 GB of VRAM. How much VRAM do you need for a 70B model at float16 without quantization? How does `--tensor-parallel-size 4` help?
2. vLLM's `--max-model-len` flag controls the maximum context window the server will allocate KV-cache for. What happens if a request's input exceeds this limit?
3. Your self-hosted endpoint has no authentication. List three ways to add access control without modifying vLLM's source code.
4. The `cost` field on your self-hosted `Model` is set to `{ input: 0, output: 0 }`. What is the actual cost per token? How would you calculate it from your instance's hourly rate and observed throughput?
5. A hosted model (e.g., Claude 3.5 Sonnet) and your self-hosted Qwen 2.5 7B produce different outputs for the same prompt. Name three reasons why, independent of model size.

---

## What's Next

- **`@mariozechner/pi` (pods) README** — full command reference, supported model list, and multi-GPU configuration
- **`packages/pods/src/`** — setup script source; read it to understand exactly what `pi pods setup` installs
- **vLLM documentation** — [docs.vllm.ai](https://docs.vllm.ai) — tensor parallelism, quantization (AWQ, GPTQ), and production deployment guides
- Return to any earlier lab and replace the hosted model with your self-hosted endpoint to compare quality and latency
