## C4 Context Diagram

```mermaid
C4Context
  title System Context — @mariozechner/pi (pods)

  Person(engineer, "AI Engineer", "Provisions and manages GPU pods from their laptop")

  System_Boundary(local, "Local machine") {
    System(pods_cli, "pi pods CLI", "Manages pod lifecycle: setup, start, stop, list")
  }

  System_Ext(gpu_cloud, "GPU Cloud Provider", "DataCrunch, RunPod, Vast.ai, AWS — bare Ubuntu GPU instances")
  System_Ext(vllm, "vLLM process", "OpenAI-compatible inference server running on the GPU pod")
  System_Ext(hf, "Hugging Face Hub", "Model weight repository")
  System_Ext(pi_ai, "pi-ai", "Consumer of the vLLM endpoint via openai-completions API")

  Rel(engineer, pods_cli, "Runs pi pods commands")
  Rel(pods_cli, gpu_cloud, "Provisions and configures via SSH")
  Rel(pods_cli, vllm, "Starts/stops vLLM process via SSH")
  Rel(vllm, hf, "Downloads model weights at startup")
  Rel(pi_ai, vllm, "Sends chat completions requests", "HTTPS/SSE")
```

---

## External Dependencies

| System | Role | Notes |
|--------|------|-------|
| GPU Cloud Provider | Provides the hardware | Any provider that gives you root SSH access to a GPU instance |
| vLLM | Inference server | Installed by `pi pods setup`; serves OpenAI-compatible `/v1/chat/completions` |
| Hugging Face Hub | Model weights | `HF_TOKEN` must be set for gated models |
| pi-ai | Consumes the endpoint | Uses `openai-completions` API with a custom `baseUrl` |

---

## Trust Boundaries

- SSH private key grants root access to the GPU pod. Protect it accordingly.
- `PI_API_KEY` is the secret that protects the vLLM endpoint. Set it in the environment before running `pi pods setup`.
- The vLLM endpoint is exposed on a public IP (or via SSH tunnel). Consider firewall rules.

---

**Back to:** [README](./README.md) | [Container View →](./c4-02-container.md)
