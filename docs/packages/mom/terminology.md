# Terminology — pi-mom

**AgentRunner**
A per-channel object that owns an `Agent` and `AgentSession`. Created once per channel on first mention; reused for all subsequent messages in that channel.

**app_mention event**
The Slack event type fired when a user mentions a bot's display name (e.g., `@mom`). This is the trigger that starts a mom conversation turn.

**Artifact**
A file produced by mom during a run (HTML report, CSV, image) that is uploaded to Slack via the `attach` tool so users can view or download it.

**Channel memory**
The `memory.md` file stored at `workspace/{channelId}/memory.md`. Contains free-form Markdown that mom and operators maintain. Injected into the system prompt on every turn to provide persistent context.

**ChannelStore**
The class (`src/store.ts`) that reads and writes session history and memory files for a single channel.

**Docker sandbox**
The `--sandbox=docker` execution mode. All bash commands run inside a Docker container, isolating the agent from the host file system and process namespace.

**Host sandbox**
The `--sandbox=host` default execution mode. Bash commands run directly on the host machine with the same permissions as the user running mom.

**Memory-augmented agent**
An agent that reads from and writes to a persistent external memory store (here: `memory.md`) to maintain context across sessions and process restarts.

**MomHandler**
The function type (`src/slack.ts`) that receives Slack event objects and routes them to the appropriate `AgentRunner`.

**Self-installing tool**
A capability that mom acquires at runtime by executing `npm install`, `apt-get install`, or similar commands and recording the installation in `memory.md`.

**Socket Mode**
A Slack delivery mechanism that pushes events to a bot over a persistent WebSocket connection. Does not require a public HTTP endpoint. Uses an `xapp-` app-level token.

**Workspace**
The operator-configured directory (e.g., `/home/user/mom-workspace`) that holds all per-channel subdirectories. Persistent across process restarts.

---

**Back to:** [README](./README.md) | [Global Glossary](../../glossary.md)
