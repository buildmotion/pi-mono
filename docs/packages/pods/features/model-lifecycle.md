# Feature: Model Lifecycle (Start / Stop / List)

## Learning Objectives

- Start and stop a model on a configured pod.
- Understand how `models.json` drives optimal vLLM configuration.
- List running models and check endpoint health.

---

## Starting a Model

```bash
pi pods start my-pod qwen2.5-72b
```

This command:
1. Loads the pod config from `~/.pi/pods.json`.
2. Calls `getModelConfig("qwen2.5-72b", pod.gpus, gpuCount)` to get vLLM arguments.
3. SSHs into the pod and launches `vllm serve` as a background process.
4. Waits for the `/health` endpoint to return 200.
5. Prints the endpoint URL.

---

## Pre-baked Model Configs (`src/models.json`)

`models.json` maps model IDs to GPU-specific vLLM arguments:

```json
{
  "models": {
    "qwen2.5-72b": {
      "name": "Qwen/Qwen2.5-72B-Instruct",
      "configs": [
        {
          "gpuCount": 2,
          "gpuTypes": ["H100", "A100"],
          "args": ["--tensor-parallel-size", "2", "--tool-call-parser", "hermes"]
        },
        {
          "gpuCount": 4,
          "args": ["--tensor-parallel-size", "4", "--tool-call-parser", "hermes"]
        }
      ]
    }
  }
}
```

`getModelConfig` selects the best config by matching GPU count and type.

---

## Tensor Parallelism

For large models (70B+) that do not fit on a single GPU, vLLM uses tensor parallelism to split the model across multiple GPUs. The `--tensor-parallel-size N` argument sets how many GPUs to use. The pre-baked configs set this automatically.

---

## Tool-Call Parsers

Different model families use different tool-call output formats. `--tool-call-parser` tells vLLM which parser to apply. Common values:

| Value | Model families |
|-------|---------------|
| `hermes` | Qwen, Mistral, many open models |
| `granite` | IBM Granite models |
| `pythonic` | Google, some others |

If not set, tool calls may not be parsed correctly.

---

## Stopping a Model

```bash
pi pods stop my-pod
```

SSHs into the pod and sends `SIGTERM` to the `vllm serve` process.

---

## Listing Pods

```bash
pi pods list
```

Output:
```
* my-pod - 2x NVIDIA H100 (vLLM: release) - ssh ubuntu@203.0.113.10
    Models: /mnt/models
```

The `*` marks the active pod (used by `pi pods stop` and `pi prompt` when no pod name is given).

---

## Check-Yourself Questions

1. What does `--tensor-parallel-size 4` mean for a 70B model?
2. How does `getModelConfig` choose between multiple configs in `models.json`?
3. What command stops vLLM on the pod?
4. What is the purpose of `--tool-call-parser hermes`?
5. How would you run a model that is NOT in `models.json`?
