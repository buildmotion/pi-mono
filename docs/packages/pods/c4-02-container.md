## C4 Container Diagram

```mermaid
C4Container
  title Container View — @mariozechner/pi (pods)

  Container_Boundary(local, "Local machine") {
    Container(cli, "CLI (src/cli.ts)", "Node.js process", "Argument parsing, command dispatch")
    Container(cfg, "Config store (src/config.ts)", "JSON file (~/.pi/pods.json)", "Persists pod definitions and active pod selection")
  }

  Container_Boundary(pod, "Remote GPU Pod (Ubuntu)") {
    Container(vllm_proc, "vLLM process", "Python process", "Serves OpenAI-compatible API on port 8000")
    Container(nfs, "NFS / model storage", "Mounted filesystem", "Stores downloaded model weights")
  }

  System_Ext(hf, "Hugging Face Hub")
  System_Ext(pi_ai, "pi-ai / your application")

  Rel(cli, cfg, "Read/write pod config")
  Rel(cli, vllm_proc, "Setup / start / stop via SSH commands", "SSH")
  Rel(vllm_proc, nfs, "Reads model weights from NFS mount")
  Rel(vllm_proc, hf, "Downloads weights if not cached")
  Rel(pi_ai, vllm_proc, "Chat completions", "HTTPS/SSE → port 8000")
```

---

## Config File (`~/.pi/pods.json`)

`pi pods` persists all pod definitions locally:

```json
{
  "active": "my-pod",
  "pods": {
    "my-pod": {
      "ssh": "ssh user@203.0.113.10",
      "gpus": [{ "name": "NVIDIA H100", "count": 2 }],
      "vllmVersion": "release",
      "modelsPath": "/mnt/models"
    }
  }
}
```

---

**← [Context](./c4-01-context.md)** | **[Component View →](./c4-03-component.md)**
