# Extension Points — pi-pods

---

## 1. Custom Model Configs

Add entries to `src/models.json` (or fork the file) to support new models:

```json
{
  "models": {
    "my-custom-model": {
      "name": "myorg/MyModel-70B",
      "configs": [
        {
          "gpuCount": 1,
          "args": ["--max-model-len", "32768", "--tool-call-parser", "hermes"]
        }
      ]
    }
  }
}
```

Then start it with `pi pods start my-pod my-custom-model`.

For models not in `models.json`, vLLM is launched without special args — it uses its own defaults.

---

## 2. Custom Provisioning Scripts

`setupPod()` in `src/commands/pods.ts` runs a sequence of SSH commands. To add custom provisioning steps (install monitoring agents, configure NFS, set up cron jobs), extend the function:

```typescript
// src/commands/pods.ts
async function postSetupHooks(ssh: string) {
  await sshExec(ssh, "curl -fsSL https://my-company.com/setup.sh | bash");
}
```

---

## 3. Alternative GPU Clouds

`pi pods` only needs SSH access to a Ubuntu GPU instance. It works with any cloud that provides this:

| Cloud | Notes |
|-------|-------|
| DataCrunch | Tested; recommended for H100 access |
| RunPod | Works; use the SSH command from the pod page |
| Vast.ai | Works; enable SSH in instance settings |
| AWS EC2 | Works with `p3.8xlarge` or `p4d` instances |
| Bare metal | Works; any Ubuntu GPU server with SSH access |

---

## 4. Custom `baseUrl` Paths

If your vLLM is behind a reverse proxy that changes the path:

```typescript
const model: Model<"openai-completions"> = {
  ...qwen72b,
  baseUrl: "https://my-gateway.example.com/pods/pod1/v1",
};
```

---

## 5. Multiple Models on One Pod

vLLM supports running multiple model instances on different ports. Launch a second instance on port 8001:

```bash
pi pods start my-pod mistral-7b --port 8001
```

(Requires adding `--port` support to `startModel()`.) Then create two separate `Model` objects with different `baseUrl` ports.

---

**What's Next:** [Observability](./observability.md) | [Design Patterns](./design-patterns.md)
