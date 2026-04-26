# @mariozechner/pi-coding-agent — Course Introduction

## What You Will Learn

- How `pi` (the CLI) orchestrates an AI agent loop for software engineering tasks.
- The four run modes: interactive TUI, print/JSON, RPC, and SDK.
- How the built-in tools (bash, read, write, edit, find, grep, ls) are composed into a coding workflow.
- How to extend `pi` with TypeScript Extensions, Skills, Prompt Templates, and Pi Packages.
- Session management, context compaction, and session branching.
- How to instrument the agent for telemetry and observability.

## Prerequisites

- Completion of [pi-ai](../ai/README.md) and [pi-agent-core](../agent/README.md) labs.
- Node.js 20+ and npm.
- Basic shell and TypeScript familiarity.

## Quick Start

```bash
# Install globally
npm install -g @mariozechner/pi-coding-agent

# Run in interactive mode
pi

# Run a one-shot command
pi --print "Summarize the files in this directory"

# Run with a specific model
pi --provider anthropic --model claude-sonnet-4-5
```

## Documentation Map

| File | Topic |
|------|-------|
| [c4-01-context.md](./c4-01-context.md) | System context and boundaries |
| [c4-02-container.md](./c4-02-container.md) | Runtime container view |
| [c4-03-component.md](./c4-03-component.md) | Internal module breakdown |
| [c4-04-code-walkthrough.md](./c4-04-code-walkthrough.md) | Annotated code trace |
| [features/agent-session.md](./features/agent-session.md) | AgentSession lifecycle |
| [features/tools.md](./features/tools.md) | Built-in coding tools |
| [features/extensions-and-skills.md](./features/extensions-and-skills.md) | Extensions, Skills, Pi Packages |
| [extension-points.md](./extension-points.md) | How to extend pi |
| [observability.md](./observability.md) | Telemetry and logging hooks |
| [design-patterns.md](./design-patterns.md) | Patterns and rationale |
| [terminology.md](./terminology.md) | Package-specific terms |
