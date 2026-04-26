# Feature: Pod Provisioning

## Learning Objectives

- Use `pi pods setup` to provision a fresh Ubuntu GPU instance for vLLM.
- Understand what the setup command installs and configures.
- Know the required environment variables and their purpose.

---

## Background

`pi pods setup` automates the tedious work of turning a bare Ubuntu GPU instance into a vLLM-ready server. It installs Python dependencies, CUDA libraries, vLLM, and configures the model storage path — all over SSH.

Source: `src/commands/pods.ts` → `setupPod()`

---

## Prerequisites

```bash
export HF_TOKEN="hf_..."          # Required: download gated models from Hugging Face
export PI_API_KEY="my-secret-key" # Required: becomes the Bearer token for the vLLM API
```

---

## Running Setup

```bash
pi pods setup my-pod "ssh ubuntu@203.0.113.10" \
  --models-path /mnt/models \
  --vllm release
```

| Argument | Description |
|----------|-------------|
| `my-pod` | Name for this pod in `~/.pi/pods.json` |
| `"ssh ubuntu@..."` | SSH command string (must connect without a password prompt) |
| `--models-path` | Remote path where model weights are stored or will be downloaded |
| `--vllm` | vLLM build variant: `release` (default), `nightly`, or `gpt-oss` |

---

## What Setup Installs

1. System packages: `python3-pip`, `nvme-cli`, CUDA toolkit
2. Python packages: `vllm` (version pinned per `--vllm` flag), `huggingface_hub`
3. Creates `~/.pi/pods.json` entry with the pod's SSH string, GPU inventory, and models path
4. Probes GPU count and type via `nvidia-smi` and stores it in the config

---

## Idempotency

`pi pods setup` is safe to re-run. It checks whether vLLM is already installed before reinstalling. Running it again on a configured pod will update the GPU inventory but not reinstall existing packages.

---

## Check-Yourself Questions

1. What two environment variables are required before running `pi pods setup`?
2. What does `--vllm gpt-oss` install compared to `--vllm release`?
3. What file on your local machine is updated by `pi pods setup`?
4. How does `pi pods setup` discover how many GPUs the pod has?
5. Is `pi pods setup` safe to run on a pod that was already set up?
