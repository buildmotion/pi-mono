# Terminology — pi-pods

**GPU pod**
A remote compute instance equipped with one or more NVIDIA GPUs, running Ubuntu. pi-pods provisions and manages these instances via SSH.

**vLLM**
An open-source, high-throughput inference engine for LLMs. It uses PagedAttention for efficient GPU memory management and exposes an OpenAI-compatible HTTP API. pi-pods installs and manages vLLM on GPU pods.

**Tensor parallelism**
A technique for distributing a large model across multiple GPUs by splitting each tensor operation across the devices. Controlled by `--tensor-parallel-size N` in vLLM. Required for models larger than a single GPU's VRAM.

**PagedAttention**
vLLM's memory management algorithm that treats the KV cache like virtual memory pages, reducing fragmentation and enabling higher throughput.

**KV cache**
The key-value cache that vLLM maintains for each active request's attention computation. Limited by GPU VRAM. When the cache fills, throughput drops and requests may be queued.

**OpenAI-compatible endpoint**
An HTTP API that accepts requests in OpenAI's chat completions format (`POST /v1/chat/completions`) and returns responses in the same format. vLLM exposes this endpoint, allowing any OpenAI client (including pi-ai) to connect.

**Tool-call parser**
A vLLM component (`--tool-call-parser`) that parses the model's raw text output into structured tool call objects. Different model families encode tool calls in different formats; selecting the wrong parser breaks tool use.

**NFS (Network File System)**
A protocol for mounting a remote file system over the network. pi-pods uses NFS mounts (e.g., from cloud-provider network storage) so that large model weight files are stored once and shared across multiple pod restarts without re-downloading.

**HF_TOKEN**
A Hugging Face API token required to download model weights from gated repositories (models that require license acceptance). Set as an environment variable before running `pi pods setup`.

**PI_API_KEY**
A secret string set as an environment variable before `pi pods setup`. Becomes the Bearer token for the vLLM API endpoint. All clients (pi-ai, curl, etc.) must include it as `Authorization: Bearer <PI_API_KEY>`.

**Active pod**
The pod selected in `~/.pi/pods.json` under the `active` key. Commands like `pi pods stop` and `pi prompt` use the active pod when no pod name is specified.

**Quantization**
Reducing model weight precision (e.g., from FP16 to INT4) to decrease VRAM usage and increase throughput, at a potential cost in quality. vLLM supports quantization via `--quantization` arguments.

---

**Back to:** [README](./README.md) | [Global Glossary](../../glossary.md)
