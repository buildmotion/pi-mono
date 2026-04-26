## C4 Container Diagram

```mermaid
C4Container
  title Container View — @mariozechner/pi-mom

  Container_Boundary(mom, "pi-mom Node.js process") {
    Container(slackClient, "SlackBot (src/slack.ts)", "WebSocket client", "Connects to Slack via Socket Mode; dispatches events to handlers")
    Container(agentFactory, "AgentRunner factory (src/agent.ts)", "Per-channel factory", "Creates an AgentRunner for each channel on first mention")
    Container(agentRunner, "AgentRunner", "Stateful per-channel agent", "Owns Agent + AgentSession; executes one conversation turn at a time")
    Container(channelStore, "ChannelStore (src/store.ts)", "File-system store", "Reads/writes session JSON and memory.md per channel")
    Container(toolset, "MomTools (src/tools/)", "AgentTool[]", "bash, read, write, edit, attach, truncate")
    Container(sandbox, "SandboxExecutor (src/sandbox.ts)", "Execution backend", "host exec or Docker container exec depending on --sandbox flag")
    Container(artifactServer, "Artifacts HTTP server", "Express-like server", "Serves HTML artifacts via public URL for Slack to preview")
  }

  System_Ext(slack, "Slack")
  System_Ext(llm, "Anthropic API")
  System_Ext(docker, "Docker daemon")
  System_Ext(fs, "Workspace FS")

  Rel(slackClient, agentFactory, "Routes app_mention events")
  Rel(agentFactory, agentRunner, "Creates one runner per channel")
  Rel(agentRunner, toolset, "Registers tools with Agent")
  Rel(agentRunner, channelStore, "Loads/saves messages and memory")
  Rel(toolset, sandbox, "Executes bash via sandbox")
  Rel(sandbox, docker, "--sandbox=docker only")
  Rel(sandbox, fs, "Reads/writes workspace files")
  Rel(slackClient, slack, "Sends replies")
  Rel(agentRunner, llm, "Streams completions")
  Rel(artifactServer, slack, "Slack unfurls artifact URLs")
```

---

## Key Runtime Facts

- **One `AgentRunner` per channel.** Channels are isolated; a conversation in `#engineering` cannot see `#marketing` history.
- **Sequential per-channel processing.** If a second mention arrives while the agent is running, mom queues it and processes it after the current run completes.
- **Persistent workspace.** All files written during a session survive process restarts.

---

**← [Context](./c4-01-context.md)** | **[Component View →](./c4-03-component.md)**
