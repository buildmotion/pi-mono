# Observability — pi-pods

Pod deployments combine local CLI observability (the `pi` process) with remote server observability (the vLLM process on the pod).

---

## 1. SSH Command Logging

`src/ssh.ts` streams SSH output to stdout. Capture it:

```bash
pi pods start my-pod qwen2.5-72b 2>&1 | tee /var/log/pods-start.log
```

This gives you a complete log of the setup/start commands and their outputs.

---

## 2. vLLM Server Logs

vLLM writes detailed logs including model load time, request latency, and errors. SSH into the pod and tail the log:

```bash
ssh ubuntu@<pod-ip> "tail -f /var/log/vllm.log"
```

Key log events to watch for:

| Log message | Meaning |
|-------------|---------|
| `Avg prompt throughput: ... tokens/s` | Inference throughput |
| `KV cache usage: ...%` | Memory pressure — if >90%, consider shorter contexts |
| `CUDA out of memory` | Model does not fit; reduce `--max-model-len` or add GPUs |
| `Engine ready` | vLLM finished loading; ready to accept requests |

---

## 3. Health Endpoint

vLLM exposes `GET /health` — poll it to check readiness:

```bash
curl http://<pod-ip>:8000/health
# 200 OK when ready
```

---

## 4. Metrics Endpoint

vLLM exposes Prometheus metrics at `/metrics`:

```bash
curl http://<pod-ip>:8000/metrics
```

Key metrics:

| Metric | Description |
|--------|-------------|
| `vllm:num_requests_running` | Active concurrent requests |
| `vllm:gpu_cache_usage_perc` | KV cache utilization |
| `vllm:time_to_first_token_seconds` | TTFT histogram |
| `vllm:e2e_request_latency_seconds` | End-to-end latency histogram |

Scrape with Prometheus and visualize in Grafana for production monitoring.

---

## 5. pi-ai Usage Tracking

When you use your pod model via pi-ai, `AssistantMessage.usage` contains token counts:

```typescript
const result = await completeSimple(podModel, context);
console.log("Tokens:", result.usage.input, result.usage.output);
// Note: cost will be 0 for self-hosted models unless you set input/output prices on the Model object
```

---

**What's Next:** [Design Patterns](./design-patterns.md) | [Terminology](./terminology.md)
