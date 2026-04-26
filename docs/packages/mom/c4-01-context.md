## C4 Context Diagram

```mermaid
C4Context
  title System Context — @mariozechner/pi-mom

  Person(slackUser, "Slack User", "Team member who mentions @mom in a channel")
  System_Ext(slack, "Slack", "Chat platform; delivers events via Socket Mode WebSocket")
  System_Ext(llm, "Anthropic API", "LLM provider (claude-sonnet-4-5 by default)")
  System_Ext(docker, "Docker daemon", "Optional sandbox for bash command execution")
  System_Ext(npm, "npm Registry", "mom installs tools with npm at runtime")
  System_Ext(fs, "Workspace File System", "Directory on the host for sessions, memory files, and installed tools")

  System_Boundary(mom_process, "pi-mom Node.js process") {
    System(mom, "mom", "Self-managing Slack bot backed by a pi-agent-core Agent")
  }

  Rel(slackUser, slack, "Mentions @mom in a channel", "Slack UI")
  Rel(slack, mom, "Delivers message events", "WebSocket / Socket Mode")
  Rel(mom, slack, "Sends replies, uploads artifacts", "Slack Web API")
  Rel(mom, llm, "Streams LLM completions", "HTTPS/SSE")
  Rel(mom, docker, "Executes bash in container (--sandbox=docker)", "Docker API")
  Rel(mom, npm, "npm install when installing new tools")
  Rel(mom, fs, "Reads/writes session files, memory files, tools")
```

---

## External Dependencies

| System | Role | Trust |
|--------|------|-------|
| Slack | Event delivery and reply sending | External; authenticated via bot/app tokens |
| Anthropic API | LLM inference | External; authenticated via API key |
| Docker daemon | Bash sandbox | Local; requires Docker socket access |
| npm Registry | Tool installation at runtime | Semi-trusted; mom can `npm install` arbitrary packages |
| Workspace FS | Persistence | Trusted; operator-controlled directory |

---

## Security Considerations

Running mom with `--sandbox=host` (the default) gives the agent **unrestricted bash access on the host machine**. Use `--sandbox=docker` in production or shared environments to isolate execution.

---

**Back to:** [README](./README.md) | [Container View →](./c4-02-container.md)
