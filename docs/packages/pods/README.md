# @mariozechner/pi (pods) — Course Introduction

## What You Will Learn

- How to provision a GPU cloud instance for self-hosted LLM inference.
- How `pi pods` configures and launches [vLLM](https://github.com/vllm-project/vllm) on a remote GPU pod.
- How the exposed OpenAI-compatible endpoint connects to `pi-ai` as a custom model.
- How pre-baked model configs select optimal tensor parallelism and tool-call parsers.

## Prerequisites

- A GPU cloud account (DataCrunch, RunPod, Vast.ai, or AWS EC2 with GPU instances).
- SSH key pair registered with your cloud provider.
- Node.js 20+.
- Basic familiarity with [pi-ai](../ai/README.md).

## Quick Start

```bash
# Install globally
npm install -g @mariozechner/pi

# Set required environment variables
export HF_TOKEN="hf_..."          # Hugging Face token for model downloads
export PI_API_KEY="my-secret-key" # API key for the vLLM endpoint

# Provision a fresh GPU instance
pi pods setup my-pod "ssh user@<pod-ip>" --models-path /mnt/models

# Start a model
pi pods start my-pod qwen2.5-72b

# List all pods
pi pods list
```

## Documentation Map

| File | Topic |
|------|-------|
| [c4-01-context.md](./c4-01-context.md) | System context |
| [c4-02-container.md](./c4-02-container.md) | Container view |
| [c4-03-component.md](./c4-03-component.md) | Component breakdown |
| [c4-04-code-walkthrough.md](./c4-04-code-walkthrough.md) | Code trace |
| [features/pod-provisioning.md](./features/pod-provisioning.md) | Provisioning a pod |
| [features/model-lifecycle.md](./features/model-lifecycle.md) | Starting/stopping models |
| [features/connecting-to-pi-ai.md](./features/connecting-to-pi-ai.md) | Using the endpoint with pi-ai |
| [extension-points.md](./extension-points.md) | Customization |
| [observability.md](./observability.md) | Monitoring |
| [design-patterns.md](./design-patterns.md) | Design patterns |
| [terminology.md](./terminology.md) | Terminology |
