## C4 Component Diagram

```mermaid
graph TD
  MAIN["main.ts<br/>CLI arg parsing, startup"]
  SLACK["slack.ts — SlackBot<br/>Socket Mode client, event router"]
  AGENT["agent.ts — AgentRunner<br/>Per-channel Agent + AgentSession wrapper"]
  CTX["context.ts<br/>MomSettingsManager; channel context loading"]
  STORE["store.ts — ChannelStore<br/>session.json + memory.md per channel"]
  TOOLS["tools/index.ts<br/>bash, read, write, edit, attach, truncate"]
  SANDBOX["sandbox.ts — SandboxExecutor<br/>host exec or Docker exec"]
  LOG["log.ts<br/>Structured stderr logger"]
  EVENTS["events.ts<br/>Slack event types and watchers"]
  FSWATCH["fs-watch.ts<br/>File-system watcher for live memory reload"]
  DOWNLOAD["download.ts<br/>Slack file download utility"]

  MAIN --> SLACK
  MAIN --> STORE
  SLACK --> AGENT
  AGENT --> CTX
  AGENT --> STORE
  AGENT --> TOOLS
  TOOLS --> SANDBOX
  SLACK --> LOG
  AGENT --> LOG
  SLACK --> EVENTS
  STORE --> FSWATCH
```

---

## Component Responsibilities

| Module | Responsibility |
|--------|---------------|
| `main.ts` | Parse CLI args, validate env vars, start SlackBot |
| `slack.ts` | Maintain WebSocket connection to Slack; dispatch `app_mention` events to `AgentRunner` |
| `agent.ts` | Create `Agent` + `AgentSession`; inject memory file into system prompt; call `agent.prompt()` |
| `context.ts` | Build `SettingsManager` for mom; load channel-specific context (memory, history) |
| `store.ts` | Read/write `workspace/{channelId}/session.json` and `memory.md` |
| `tools/` | bash, read, write, edit, attach (uploads file to Slack), truncate |
| `sandbox.ts` | Execute bash commands either directly or via `docker exec` |
| `log.ts` | Structured JSON logging to stderr |
| `fs-watch.ts` | Watch memory.md for external edits; reload automatically |

---

**← [Container](./c4-02-container.md)** | **[Code Walkthrough →](./c4-04-code-walkthrough.md)**
