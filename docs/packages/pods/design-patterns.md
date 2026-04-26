# Design Patterns — pi-pods

## 1. Infrastructure-as-Code (SSH Automation)

**Pattern:** Define infrastructure setup as code that can be re-run idempotently.

`setupPod()` encodes the full provisioning sequence as TypeScript functions that issue SSH commands. Every step is explicit, version-controlled, and reproducible. Running `pi pods setup` again on a configured pod is safe.

**Why:** Eliminates manual, error-prone server setup. New team members can provision identical environments in minutes.

---

## 2. Registry Pattern (Model Configs)

**Pattern:** A central registry maps identifiers to configurations.

`models.json` maps model IDs to GPU-specific vLLM arguments. `getModelConfig()` queries the registry. Adding support for a new model requires only a new entry in `models.json`, not code changes.

**Why:** Separates configuration data from control flow. The registry can be updated (e.g., via a PR) without touching the launch logic.

---

## 3. Adapter (OpenAI-Compatible Endpoint)

**Pattern:** Translate between incompatible interfaces — here, vLLM's API format and pi-ai's `openai-completions` API.

vLLM speaks OpenAI's chat completions protocol. pi-ai's `openai-completions` provider speaks the same protocol. The "adapter" is the `Model` object with a custom `baseUrl` — no additional code needed.

**Why:** Allows any OpenAI-compatible server (not just vLLM) to be used with pi-ai and the entire pi-mono stack.

---

## 4. Strategy (GPU Tier Selection)

**Pattern:** Different implementations of the same operation selected at runtime.

`getModelConfig()` selects the best set of vLLM arguments based on the available GPU type and count. A model on 2x H100 gets different tensor parallelism settings than on 4x A100. The strategy is selected at runtime from the registry.

**Why:** One model can run on many GPU configurations. The optimal settings differ; encoding them in the registry avoids manual tuning.

---

**What's Next:** [Terminology](./terminology.md) | [Extension Points](./extension-points.md)
