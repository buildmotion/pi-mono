# pi-mono: IT Stakeholder Brief

## What is pi-mono?

`pi-mono` is an open-source TypeScript monorepo that provides a complete, layered stack for building AI-powered developer tools. It covers everything from raw LLM API calls all the way up to finished user interfaces — both terminal and browser-based. Every package is independently published to npm, versioned in lockstep, and can be consumed on its own or composed together.

The project is authored by Mario Zechner ([@badlogic](https://github.com/badlogic)) and licensed under MIT.

---

## Package Inventory

| Package | npm name | Layer | Brief description |
|---------|----------|-------|-------------------|
| `packages/ai` | `@mariozechner/pi-ai` | Foundation | Unified streaming API for 20+ LLM providers |
| `packages/agent` | `@mariozechner/pi-agent-core` | Core | Stateful agent with tool execution and event streaming |
| `packages/tui` | `@mariozechner/pi-tui` | UI (terminal) | Flicker-free terminal UI framework |
| `packages/coding-agent` | `@mariozechner/pi-coding-agent` | Application | Terminal coding harness (`pi` CLI) |
| `packages/web-ui` | `@mariozechner/pi-web-ui` | UI (browser) | Reusable browser chat UI components |
| `packages/mom` | `@mariozechner/pi-mom` | Application | Self-managing Slack bot backed by an LLM agent |
| `packages/pods` | `@mariozechner/pi` | Infrastructure | CLI for deploying LLMs on GPU pods via vLLM |

---

## Dependency Graph

```
packages/ai          ← no internal deps; wraps provider SDKs
     ↑
packages/agent       ← depends on pi-ai
     ↑           ↑
packages/tui   packages/web-ui  (also depends on pi-agent-core)
     ↑
packages/coding-agent  ← depends on pi-ai, pi-agent-core, pi-tui
     ↑
packages/mom           ← depends on pi-ai, pi-agent-core, pi-coding-agent
packages/pods          ← depends on pi-agent-core
```

---

## Individual Package Summaries

### 1. `pi-ai` — LLM Provider Abstraction

**Problem it solves:** Every LLM provider has a different SDK, authentication scheme, model naming convention, streaming protocol, and tool-calling format. Developers who want to swap providers or support multiple providers end up writing adapter code for each one.

**What it does:** Presents a single streaming `stream()` / `complete()` API that works identically across Anthropic, OpenAI, Google Gemini, Mistral, AWS Bedrock, Azure OpenAI, and a dozen more. Models are discovered automatically at build time from provider APIs. Token counting, cost tracking, cross-provider context hand-offs, and OAuth login flows are built in.

**Audience:** Any TypeScript/Node.js developer building on top of LLMs.

**Pros:** Zero-friction provider switching; type-safe model selection; browser-compatible; OAuth for subscription accounts (Claude Pro, GitHub Copilot, etc.).

**Cons:** Tool-calling-only scope (non-tool models are excluded by design); requires understanding provider-specific options for advanced tuning.

---

### 2. `pi-agent-core` — Stateful Agent Engine

**Problem it solves:** A raw LLM streaming call gives you a single response. Real agentic workflows require maintaining conversation history, calling tools in a loop, reacting to tool results, and surfacing structured events to the UI — none of which the raw API provides.

**What it does:** Wraps `pi-ai` in a stateful `Agent` class that manages messages, runs the tool-call → result → LLM loop automatically, emits a rich event stream (`agent_start`, `turn_start`, `message_update`, `tool_execution_end`, …), and supports steering (interrupt mid-flight) and follow-up queuing.

**Audience:** Developers building any AI-powered application that needs multi-turn conversations with tools.

**Pros:** Clean event model suitable for both CLI and browser UIs; parallel tool execution; proxy support for browser deployments; extensible message types via TypeScript declaration merging.

**Cons:** No built-in persistence — callers are responsible for serializing `agent.state.messages`.

---

### 3. `pi-tui` — Terminal UI Framework

**Problem it solves:** Building interactive terminal applications (editors, chat interfaces, progress displays) is hard. Most approaches either flicker heavily, do not handle wide characters or images, or require a heavy ncurses dependency.

**What it does:** A lightweight, dependency-minimal TUI framework built entirely in TypeScript. Uses CSI 2026 synchronized output for flicker-free rendering, a differential three-strategy renderer that only repaints changed lines, and supports Kitty/iTerm2 inline image protocols. Ships built-in components: multi-line editor, Markdown renderer, autocomplete, fuzzy selector, progress indicators, and overlays.

**Audience:** Developers building terminal-first tools in Node.js.

**Pros:** No native dependencies (optional `koffi` only for clipboard); correct East Asian width handling; IME support; themeable.

**Cons:** Terminal-only (not web); relies on terminal emulator support for some features (inline images, bracketed paste).

---

### 4. `pi-coding-agent` — Terminal Coding Harness (`pi` CLI)

**Problem it solves:** Existing AI coding tools are opinionated monoliths that are hard to adapt to team-specific workflows without forking. Developers need a coding assistant they can extend without touching internals.

**What it does:** `pi` is the primary end-user application in the monorepo. It runs in four modes: interactive TUI, print/JSON, RPC (process integration), and SDK (embedded in other apps). Out of the box it gives the model `read`, `write`, `edit`, and `bash` tools. Everything else — sub-agents, plan mode, custom workflows — can be added via TypeScript Extensions, Skills (small CLI scripts the model can call), Prompt Templates, and Themes, all bundled as shareable Pi Packages published to npm or git.

**Audience:** Individual developers and engineering teams who want a customizable AI pair-programmer in the terminal.

**Pros:** Supports 20+ providers including subscription accounts; session branching and compaction; RPC mode for IDE/tool integration; fully extensible without forking; cross-platform (Linux, macOS, Windows, Termux/Android).

**Cons:** Terminal-only UI (browser users should look at `pi-web-ui`); some advanced features (sub-agents, plan mode) must be built as extensions rather than being shipped out of the box.

---

### 5. `pi-web-ui` — Browser Chat UI Components

**Problem it solves:** Building a polished AI chat interface for a web app requires integrating streaming, file attachments, code highlighting, sandboxed artifact execution, session persistence, model selection, API key management, and a CORS proxy — all from scratch.

**What it does:** A library of web components (built with `mini-lit` and Tailwind CSS v4) that snap together into a full chat UI. The top-level `ChatPanel` component wires an `Agent` to a complete interface with message history, streamed responses, tool-call cards, an artifacts panel (sandboxed HTML/SVG/Markdown execution), file attachment support (PDF, DOCX, XLSX, PPTX, images), a JavaScript REPL tool, and an IndexedDB-backed storage layer for sessions, API keys, and settings.

**Audience:** Web developers embedding an AI assistant into a product; teams building standalone browser-based coding or document-analysis tools.

**Pros:** Production-ready out of the box (used in the [sitegeist](https://sitegeist.ai) browser extension); custom provider support (Ollama, LM Studio, vLLM); internationalization API; fully themeable via Tailwind and CSS custom properties.

**Cons:** Requires peer dependencies (`mini-lit`, `lit`); PersistentStorageDialog is currently broken (noted in README); browser-only.

---

### 6. `pi-mom` — Self-Managing Slack Bot

**Problem it solves:** Setting up and maintaining an AI Slack bot that can actually do useful work (run commands, write files, create tools, remember context) requires significant DevOps effort and custom code. Most bots are read-only or operate in a heavily sandboxed way that limits utility.

**What it does:** `mom` is a Node.js process that connects to Slack via Socket Mode. Each channel gets its own isolated conversation history, memory (MARKDOWN files), and tool workspace. When mentioned, mom reads recent messages, loads channel-specific memory, invokes the LLM agent (defaulting to Anthropic), and responds — using `bash`, `read`, `write`, and any tools she has installed herself. She installs new tools autonomously (`npm`, `apk`, shell scripts) and stores everything in a workspace directory you control. A Docker sandbox mode isolates execution.

**Audience:** Engineering and DevOps teams that want an always-on AI assistant living in their Slack workspace, with the ability to run commands and manage files.

**Pros:** Self-managing (zero tool pre-configuration needed); per-channel memory and history; Docker sandbox for safety; artifacts server for sharing HTML/JS visualizations; scheduled events/reminders.

**Cons:** Anthropic-only LLM support (by default); requires a Slack App setup with Socket Mode; running on the host with `--sandbox=host` is a significant security risk.

---

### 7. `pods` — GPU Pod Manager

**Problem it solves:** Spinning up and managing self-hosted LLMs on GPU cloud instances (DataCrunch, RunPod, Vast.ai, EC2) involves manual SSH, vLLM installation, model downloads, GPU memory configuration, tool-call parser selection, and multi-GPU coordination — all error-prone and time-consuming.

**What it does:** The `pi-pods` CLI (installed globally as `pi`) automates the full lifecycle of self-hosted LLM deployments. One command provisions a fresh Ubuntu GPU pod (installs vLLM, configures NFS mounts, handles authentication); subsequent commands start, stop, list, and monitor models. Pre-baked configurations exist for popular agentic models (Qwen, GPT-OSS, GLM) with correct tool-call parsers and tensor parallelism settings. Models expose OpenAI-compatible endpoints. A bundled `pi-agent` CLI lets you chat with deployed models immediately.

**Audience:** AI engineering teams and researchers who need to self-host large models for cost, latency, or data-privacy reasons.

**Pros:** Abstracts nearly all vLLM complexity; shared NFS storage across pods (DataCrunch); automatic GPU assignment for multi-model pods; OpenAI-compatible output works with any downstream tooling.

**Cons:** Primarily tested on DataCrunch and RunPod; `--sandbox` (Docker) concept is separate from the `mom` sandbox and not present here; no web UI for monitoring.

---

## Architecture Summary

The monorepo is structured as a clean dependency pyramid:

- **Foundation** (`pi-ai`): Provider normalization, no business logic.
- **Core** (`pi-agent-core`): Business logic for agent loops and tool execution, depends only on `pi-ai`.
- **UI layer** (`pi-tui`, `pi-web-ui`): Rendering primitives with no AI logic of their own.
- **Applications** (`pi-coding-agent`, `pi-mom`, `pods`): Compose the layers above into complete, deployable tools.

All packages share a version number, build with TypeScript + `tsgo`, lint with Biome, and test with Vitest.

---

## Quick Reference: Which Package to Use?

| Goal | Use |
|------|-----|
| Call an LLM from TypeScript | `pi-ai` |
| Build a multi-turn AI app with tools | `pi-agent-core` |
| Add a terminal UI to a Node.js tool | `pi-tui` |
| Use a coding assistant in the terminal | `pi-coding-agent` (`pi` CLI) |
| Embed an AI chat in a web page | `pi-web-ui` |
| Add an AI assistant to Slack | `pi-mom` |
| Self-host LLMs on GPU cloud | `pods` (`pi-pods` CLI) |
