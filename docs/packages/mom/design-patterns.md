# Design Patterns — pi-mom

## 1. Memory-Augmented Agent

**Pattern:** Persist context outside the LLM's in-flight context window and inject it at the start of each turn.

mom reads `memory.md` and prepends it to the system prompt on every turn. This gives the agent long-term memory that survives process restarts and context compaction, without storing all prior messages in the live context.

**Why:** The LLM context window is finite and expensive. A concise, human-curated markdown file is far more token-efficient than replaying a full conversation history.

---

## 2. Workspace Pattern

**Pattern:** Every actor in a multi-tenant system gets its own isolated directory.

Each Slack channel maps to `workspace/{channelId}/`. Sessions, memory, and installed tools are channel-scoped. mom cannot accidentally mix up state between channels.

**Why:** Simple, filesystem-native isolation. Easy to inspect, backup, and restore without a database.

---

## 3. Self-Installing Plugin System

**Pattern:** An agent can extend its own capabilities at runtime by installing software.

Rather than shipping every possible tool, mom installs tools when she needs them (via `npm install`, `apt-get`, etc.) and records what she installed in `memory.md`. The next time she needs the same tool, she checks memory first.

**Why:** Keeps the base image/package small. Allows the agent to adapt to team-specific tooling without human intervention.

---

## 4. Sandbox / Execution Backend Strategy

**Pattern:** Abstract the execution environment behind an interface; swap implementations without changing callers.

`SandboxExecutor` (`src/sandbox.ts`) exposes a single `exec(command)` method. `--sandbox=host` and `--sandbox=docker` are two strategies. All tools call `executor.exec()` without knowing which backend is active.

**Why:** Allows security posture to change (from dev to production) without modifying tool implementations.

---

## 5. Event-Driven Architecture (Socket Mode)

**Pattern:** The system reacts to events rather than polling.

mom receives a Slack WebSocket push event for every mention. It does not poll the Slack API. This minimises latency and API rate-limit consumption.

**Why:** Real-time responsiveness with zero unnecessary network traffic.

---

**What's Next:** [Terminology](./terminology.md) | [Extension Points](./extension-points.md)
