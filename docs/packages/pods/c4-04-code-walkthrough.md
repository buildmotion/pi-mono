## Learning Objectives

Trace `pi pods start my-pod qwen2.5-72b` from CLI entry to a live vLLM endpoint.

---

## Step 1 — CLI Entry (`src/cli.ts`)

```bash
pi pods start my-pod qwen2.5-72b
```

The CLI parser dispatches to `startModel("my-pod", "qwen2.5-72b", options)` in `src/commands/pods.ts`.

---

## Step 2 — Load Pod Config (`src/config.ts`)

```typescript
// src/commands/pods.ts (simplified)
const config = loadConfig();           // reads ~/.pi/pods.json
const pod = config.pods[podName];      // { ssh: "ssh user@...", gpus: [...], ... }
```

---

## Step 3 — Resolve Model Config (`src/model-configs.ts`)

```typescript
// src/commands/pods.ts (simplified)
const modelCfg = getModelConfig(modelId, pod.gpus, requestedGpuCount);
// Returns: { args: ["--tensor-parallel-size", "2", "--tool-call-parser", "hermes"], ... }
```

`getModelConfig` reads `models.json` to find the best vLLM launch arguments for the given model and GPU combination.

---

## Step 4 — SSH to Pod and Launch vLLM (`src/ssh.ts`)

```typescript
// src/commands/pods.ts (simplified)
const vllmCmd = [
  "vllm", "serve", modelId,
  "--api-key", process.env.PI_API_KEY,
  "--port", "8000",
  ...modelCfg.args,
].join(" ");

await sshExecStream(pod.ssh, `nohup ${vllmCmd} > /var/log/vllm.log 2>&1 &`);
```

`sshExecStream` streams the SSH output to stdout so you see startup logs in real time.

---

## Step 5 — Health Check

```typescript
// src/commands/pods.ts (simplified)
console.log("Waiting for vLLM to become ready...");
await pollUntilHealthy(`http://<pod-ip>:8000/health`, { timeoutMs: 120_000 });
console.log("vLLM is ready. Endpoint: http://<pod-ip>:8000");
```

---

## Step 6 — Endpoint Is Live

The endpoint is now accepting requests at `http://<pod-ip>:8000/v1/chat/completions` with `Authorization: Bearer <PI_API_KEY>`.

Connect to it from pi-ai — see [features/connecting-to-pi-ai.md](./features/connecting-to-pi-ai.md).

---

**← [Component](./c4-03-component.md)** | **[Features →](./features/pod-provisioning.md)**
