# pi-mono Documentation

Welcome to the pi-mono documentation site. This repo is a TypeScript monorepo that builds a complete, layered stack for AI-powered developer tools — from raw LLM streaming calls all the way up to a terminal coding assistant, a browser chat UI, and a self-managing Slack bot. These docs are written for AI engineering students who want to understand how real production agentic systems are designed and assembled from composable pieces.

---

## How These Docs Are Organized

```
docs/
├── index.md                    ← you are here
├── glossary.md                 ← terminology reference for every concept used across packages
├── packages/                   ← per-package deep-dives (READMEs, API surface, design rationale)
│   ├── pi-ai.md
│   ├── pi-agent-core.md
│   ├── pi-tui.md
│   ├── pi-coding-agent.md
│   ├── pi-web-ui.md
│   ├── pi-mom.md
│   └── pods.md
├── reference/                  ← cross-cutting architecture and engineering topics
│   ├── architecture-overview.md
│   ├── observability.md
│   └── multi-agent-orchestration.md
└── labs/                       ← hands-on exercises (coming soon)
```

Start with `glossary.md` to build shared vocabulary. Then read `reference/architecture-overview.md` to understand how the packages fit together before diving into individual package docs.

---

## Package Table

| Package | npm name | One-liner | Docs |
|---------|----------|-----------|------|
| `packages/ai` | `@mariozechner/pi-ai` | Unified streaming API for 20+ LLM providers | [pi-ai.md](packages/pi-ai.md) |
| `packages/agent` | `@mariozechner/pi-agent-core` | Stateful agent loop with tool execution and event streaming | [pi-agent-core.md](packages/pi-agent-core.md) |
| `packages/tui` | `@mariozechner/pi-tui` | Flicker-free terminal UI framework | [pi-tui.md](packages/pi-tui.md) |
| `packages/coding-agent` | `@mariozechner/pi-coding-agent` | Terminal coding harness (`pi` CLI) | [pi-coding-agent.md](packages/pi-coding-agent.md) |
| `packages/web-ui` | `@mariozechner/pi-web-ui` | Browser chat UI components | [pi-web-ui.md](packages/pi-web-ui.md) |
| `packages/mom` | `@mariozechner/pi-mom` | Self-managing Slack bot backed by an LLM agent | [pi-mom.md](packages/pi-mom.md) |
| `packages/pods` | `@mariozechner/pi` | GPU pod manager for self-hosted LLMs via vLLM | [pods.md](packages/pods.md) |

---

## Reference Docs

| Document | What you will learn |
|----------|---------------------|
| [Architecture Overview](reference/architecture-overview.md) | Layered architecture, dependency graph, runtime flow traces for every major scenario |
| [Observability](reference/observability.md) | Agent event taxonomy, structured logging, tracing, metrics, and telemetry hooks |
| [Multi-Agent Orchestration](reference/multi-agent-orchestration.md) | Sequential pipelines, supervisor/worker patterns, fan-out, state isolation, context hand-off |

---

## Labs

Hands-on exercises are located in `docs/labs/`. Each lab walks through building a working component of the stack from scratch, with guided checkpoints.

*(Labs are under active development.)*

---

## How to Navigate (For Students)

**If you are new to the codebase**, follow this path:

1. Read `glossary.md` end-to-end. Every term used in code comments and docs is defined there.
2. Read `reference/architecture-overview.md` to understand the layer model and data flows.
3. Pick the package that is closest to what you are building, and read its package doc.
4. Come back to `reference/observability.md` when you need to add logging or metrics.
5. Come back to `reference/multi-agent-orchestration.md` when you need agents to talk to each other.

**If you are looking for a specific concept**, check `glossary.md` first — definitions include pointers to the TypeScript source where each concept lives.

**If you are debugging a runtime issue**, the runtime flow traces in `reference/architecture-overview.md` show exactly which functions call which, in which order, for each major scenario.
