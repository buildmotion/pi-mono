# @mariozechner/pi-mom — Course Introduction

## What You Will Learn

- How `pi-mom` acts as a self-managing Slack bot backed by a pi-agent-core `Agent`.
- How Slack Socket Mode delivers events to a Node.js process without exposing a public HTTP endpoint.
- How per-channel memory files give the agent persistent, channel-scoped context.
- How mom autonomously installs new CLI tools at runtime to extend her own capabilities.
- How Docker sandboxing isolates bash execution from the host.

## Prerequisites

- Completed [pi-agent-core](../agent/README.md) and [pi-coding-agent](../coding-agent/README.md) labs.
- A Slack workspace where you have admin rights.
- Node.js 20+, an Anthropic API key, and (optionally) Docker.

## Quick Start

```bash
# 1. Create a Slack app with Socket Mode (see c4-01-context.md for full steps)
# 2. Set environment variables
export MOM_SLACK_APP_TOKEN="xapp-..."
export MOM_SLACK_BOT_TOKEN="xoxb-..."
export ANTHROPIC_API_KEY="sk-ant-..."

# 3. Run mom
npx @mariozechner/pi-mom /path/to/workspace
```

Mom listens for Slack mentions and responds using the LLM agent.

## Documentation Map

| File | Topic |
|------|-------|
| [c4-01-context.md](./c4-01-context.md) | System context |
| [c4-02-container.md](./c4-02-container.md) | Container view |
| [c4-03-component.md](./c4-03-component.md) | Component breakdown |
| [c4-04-code-walkthrough.md](./c4-04-code-walkthrough.md) | Code trace |
| [features/per-channel-memory.md](./features/per-channel-memory.md) | Per-channel memory |
| [features/self-extending-tools.md](./features/self-extending-tools.md) | Self-extending tools |
| [features/docker-sandbox.md](./features/docker-sandbox.md) | Docker sandbox |
| [extension-points.md](./extension-points.md) | Extension points |
| [observability.md](./observability.md) | Observability |
| [design-patterns.md](./design-patterns.md) | Design patterns |
| [terminology.md](./terminology.md) | Terminology |
