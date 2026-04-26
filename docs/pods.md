# `@mariozechner/pi` (pods) — GPU Pod Manager

## What is it?

`pods` is a CLI tool (`pi-pods`) that automates the full lifecycle of self-hosted LLM deployments on remote GPU cloud instances. It installs vLLM, configures models with the correct tool-calling parsers and GPU memory settings, manages multiple models on a single pod, and exposes each model as an OpenAI-compatible API endpoint.

**npm package:** `@mariozechner/pi`  
**CLI binary:** `pi-pods`  
**Version:** 0.70.2  
**Node requirement:** >= 20.0.0  
**License:** MIT

---

## The Problem It Solves

Running a self-hosted LLM requires:

1. Provisioning a GPU instance on a cloud provider
2. Installing CUDA drivers, Python, and vLLM
3. Mounting persistent storage (so models are not re-downloaded every restart)
4. Configuring vLLM with the right `--tensor-parallel-size`, `--tool-call-parser`, `--gpu-memory-utilization`, and dozens of other flags
5. Starting the model as a background process with a supervised restart policy
6. Remembering which port each model is on when running multiple models
7. Debugging GPU OOM errors and model compatibility issues

Each of these steps requires deep familiarity with vLLM internals and GPU arithmetic. `pods` automates all of it behind a handful of simple commands.

---

## Audience

- AI engineering teams that self-host large models for cost, latency, or data-privacy reasons
- Researchers running experiments on GPU cloud providers (DataCrunch, RunPod)
- Teams building on top of OpenAI-compatible APIs who want to swap in self-hosted models
- Anyone who wants to run Qwen, GLM, GPT-OSS, or other agentic models without deep vLLM expertise

---

## Supported Providers

| Provider | Storage type | Notes |
|----------|-------------|-------|
| **DataCrunch** | NFS (sharable across pods) | Best choice; models download once, used by all pods in a datacenter |
| **RunPod** | Network volumes | Persistent per-pod; cannot share between running pods simultaneously |
| Vast.ai | Instance volumes | Locked to specific machine |
| Prime Intellect | None | No persistent storage |
| AWS EC2 | EFS | Manual EFS setup required |
| Any Ubuntu machine | Any | Requires NVIDIA drivers, CUDA, SSH root access |

---

## How It Works: Step by Step

### 1. Pod Setup (`pi pods setup`)

```bash
pi pods setup dc1 "ssh root@1.2.3.4" \
  --mount "sudo mount -t nfs -o nconnect=16 nfs.fin-02.datacrunch.io:/hf-models /mnt/hf-models"
```

What happens:

1. `pi-pods` SSHs into the instance.
2. Runs the mount command to attach persistent NFS storage.
3. Installs vLLM (release, nightly, or gpt-oss build depending on `--vllm` flag).
4. Configures HuggingFace cache to point at the mounted path.
5. Saves the pod configuration to `~/.pi/pods.json`.

The models path is automatically extracted from the `--mount` command's target directory. You only need `--models-path` if you want to override it.

#### vLLM Build Variants

| `--vllm` flag | Use case |
|--------------|---------|
| `release` (default) | Stable; recommended for Qwen, Mistral, Llama |
| `nightly` | Required for newest models like GLM-4.5 |
| `gpt-oss` | Special build required for OpenAI's GPT-OSS models |

### 2. Starting a Model (`pi start`)

```bash
pi start Qwen/Qwen2.5-Coder-32B-Instruct --name qwen --context 64k --memory 70%
```

What happens:

1. `pi-pods` loads the pod configuration.
2. Looks up the model in `model-configs.ts` (the predefined model registry).
3. If found: retrieves the correct vLLM flags (`--tool-call-parser hermes --enable-auto-tool-choice`, `--tensor-parallel-size 1`, etc.).
4. If not found: requires `--vllm` flag for manual vLLM argument passthrough.
5. Checks GPU count and VRAM to verify the model fits.
6. SSHs into the pod and starts vLLM as a background process.
7. Records the model's name, HuggingFace ID, port, and GPU assignment in the pod's state file.
8. The model serves on a port derived from its position (e.g., model 0 → port 8001, model 1 → port 8002).

### 3. Listing and Monitoring

```bash
pi list        # shows running models, ports, status
pi logs qwen   # streams vLLM logs (tail -f)
pi ssh "nvidia-smi"  # run any command on the active pod
```

### 4. Stopping

```bash
pi stop qwen   # stop a specific model
pi stop        # stop all models
```

### 5. Using the Model

Every model exposes an OpenAI-compatible endpoint:

```python
from openai import OpenAI
client = OpenAI(base_url="http://1.2.3.4:8001/v1", api_key="your-pi-api-key")
response = client.chat.completions.create(model="Qwen/Qwen2.5-Coder-32B-Instruct", messages=[…])
```

Any tool that speaks OpenAI chat completions (LangChain, LlamaIndex, `pi-ai`, `pi-coding-agent`, etc.) works out of the box.

---

## Predefined Model Configurations

`pods` ships `model-configs.ts` with pre-baked configurations for popular agentic models. You do not have to know vLLM flags to run these:

### Qwen Models

| Model | GPU count | Notes |
|-------|-----------|-------|
| `Qwen/Qwen2.5-Coder-32B-Instruct` | 1× H100/H200 | Excellent coding; default parser: `hermes` |
| `Qwen/Qwen3-Coder-30B-A3B-Instruct` | 1× GPU | Advanced reasoning; parser: `qwen3_coder` |
| `Qwen/Qwen3-Coder-480B-A35B-Instruct-FP8` | 8× H200 | SOTA; data-parallel mode |

### GLM Models

| Model | GPU count | Notes |
|-------|-----------|-------|
| `zai-org/GLM-4.5` | 8–16× GPUs | Thinking mode; parser: `glm4_moe`; requires nightly vLLM |
| `zai-org/GLM-4.5-Air` | 1–2× GPUs | Smaller variant |

### GPT-OSS Models

| Model | VRAM requirement | Notes |
|-------|-----------------|-------|
| `openai/gpt-oss-20b` | 16 GB+ | Uses `/v1/responses` endpoint |
| `openai/gpt-oss-120b` | 60 GB+ | Requires gpt-oss vLLM build |

### Custom Models (`--vllm`)

For models not in the registry:

```bash
pi start deepseek-ai/DeepSeek-V3 --name deepseek --vllm \
  --tensor-parallel-size 4 --trust-remote-code
```

When `--vllm` is used, `--memory`, `--context`, and `--gpus` are ignored (you control everything via raw vLLM flags).

---

## Multi-GPU Allocation

When starting multiple models, `pi-pods` automatically assigns them to non-overlapping GPU slots:

```bash
pi start model1 --name m1   # → GPU 0
pi start model2 --name m2   # → GPU 1
pi start model3 --name m3   # → GPU 2
```

For large models that need multiple GPUs, `--gpus` (predefined models) or `--tensor-parallel-size` (`--vllm` mode) controls the allocation.

---

## Memory and Context Tuning

| Flag | Effect |
|------|--------|
| `--memory 30%` | Low `gpu_memory_utilization`; many concurrent requests, small context |
| `--memory 50%` | Balanced (default) |
| `--memory 90%` | Maximum context, few concurrent requests |
| `--context 4k` | 4,096 token context window |
| `--context 64k` | 65,536 token context window |
| `--context 128k` | 131,072 token context window |

---

## Tool Calling Parsers

Different model families require different vLLM tool-call parsers to emit properly formatted JSON tool calls:

| Model family | Parser |
|-------------|--------|
| Qwen2.5 | `hermes` |
| Qwen3-Coder | `qwen3_coder` |
| GLM-4.5 | `glm4_moe` |
| GPT-OSS | Uses `/v1/responses` endpoint instead of chat completions |
| Custom | `--vllm --tool-call-parser <name> --enable-auto-tool-choice` |

The predefined configurations handle this automatically.

---

## Standalone Agent CLI

`pods` bundles a `pi-agent` standalone CLI that talks to any deployed model:

```bash
# Interactive chat with a running model
pi agent qwen -i

# Single message
pi agent qwen "Explain tensor parallelism"

# Continue a previous session
pi agent qwen -i -c
```

The agent includes file-system tools (read, list, bash, glob, rg) for testing agentic capabilities against code.

Session history is stored in `~/.pi/sessions/`.

---

## Source Layout

```
packages/pods/src/
├── index.ts           Public API exports
├── cli.ts             CLI entry point (argument parsing, command dispatch)
├── config.ts          ~/.pi/pods.json load/save, active pod management
├── types.ts           PodConfig, ModelConfig, RunningModel interfaces
├── ssh.ts             SSH command execution helpers
├── model-configs.ts   Predefined vLLM configurations for known models
├── models.json        Model registry (HF IDs, GPU requirements, parsers)
└── commands/
    ├── setup.ts       pi pods setup — installs vLLM on a fresh pod
    ├── start.ts       pi start — starts a model
    ├── stop.ts        pi stop — stops model(s)
    ├── list.ts        pi list — shows running models
    ├── logs.ts        pi logs — streams model logs
    ├── shell.ts       pi shell — SSH into pod
    └── agent.ts       pi agent — runs interactive/single-shot agent
```

---

## Environment Variables

| Variable | Purpose |
|---------|---------|
| `HF_TOKEN` | HuggingFace token for downloading gated models (Llama, Mistral) |
| `PI_API_KEY` | API key used to authenticate requests to vLLM endpoints |
| `PI_CONFIG_DIR` | Override config directory (default: `~/.pi`) |
| `OPENAI_API_KEY` | Used by `pi-agent` when no `--api-key` is provided |

---

## Pros and Cons

**Pros**
- Abstracts nearly all vLLM complexity behind simple commands
- Predefined configurations for the most popular agentic models (correct parsers, GPU settings)
- Shared NFS storage on DataCrunch means models download once across all pods in a datacenter
- Automatic GPU assignment for multi-model pods
- OpenAI-compatible output works with any existing tooling
- JSON output mode on `pi-agent` for programmatic consumption

**Cons**
- Primarily tested on DataCrunch and RunPod; other providers may need more manual configuration
- No web UI for monitoring — entirely CLI-based
- `--vllm` passthrough bypasses all safety checks (memory, context, GPU count)
- vLLM nightly builds can be unstable
- No built-in SSL/TLS for model endpoints — relies on network-level security or a reverse proxy

---

## How to Use It (Junior Developer Walkthrough)

Think of `pi-pods` as a remote control for GPU servers running LLMs. You describe what model you want to run; it figures out all the knobs and dials.

1. **Install:**
   ```bash
   npm install -g @mariozechner/pi
   ```

2. **Set tokens:**
   ```bash
   export HF_TOKEN=hf_...        # for downloading models from HuggingFace
   export PI_API_KEY=mysecretkey  # protects your vLLM endpoint
   ```

3. **Provision a GPU pod** on DataCrunch, RunPod, or any Ubuntu+NVIDIA machine. Get the SSH command.

4. **Set up the pod** (one time per machine):
   ```bash
   pi pods setup mypod "ssh root@1.2.3.4" \
     --mount "sudo mount -t nfs … /mnt/models"
   ```
   This installs vLLM and configures everything. Takes ~10 minutes.

5. **Start a model:**
   ```bash
   pi start Qwen/Qwen2.5-Coder-32B-Instruct --name qwen
   ```
   The first run downloads the model (~60 GB). Subsequent starts reuse the cached weights.

6. **Verify it's running:**
   ```bash
   pi list
   pi logs qwen   # watch the startup output
   ```

7. **Chat with it:**
   ```bash
   pi agent qwen -i
   ```

8. **Use it from any OpenAI client:**
   ```python
   from openai import OpenAI
   client = OpenAI(base_url="http://1.2.3.4:8001/v1", api_key="mysecretkey")
   ```
