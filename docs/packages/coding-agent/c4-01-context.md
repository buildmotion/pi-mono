## Learning Objectives

- Identify who uses `pi` and what external systems it interacts with.
- Understand the trust boundary between the agent process and the local file system.
- Know which external services `pi` can reach and under what circumstances.

---

## C4 Context Diagram

```mermaid
C4Context
  title System Context — @mariozechner/pi-coding-agent

  Person(dev, "Developer", "Uses pi in the terminal to get AI assistance with coding tasks")

  System_Boundary(pi_process, "pi process") {
    System(pi, "pi (coding-agent)", "AI coding assistant CLI with tool execution and session management")
  }

  System_Ext(llm, "LLM Provider API", "Anthropic, OpenAI, Google, etc.")
  System_Ext(fs, "Local File System", "Working directory; pi reads, writes, and executes files here")
  System_Ext(shell, "Shell / Bash", "pi executes bash commands on the developer's machine")
  System_Ext(npm, "npm Registry", "Pi Packages (Extensions, Skills, Templates) installed from npm or git")
  System_Ext(ide, "IDE / Editor (RPC mode)", "External tools integrate with pi via JSON-RPC over stdio")

  Rel(dev, pi, "Types prompts, views responses, configures settings")
  Rel(pi, llm, "Streams LLM completions", "HTTPS/SSE")
  Rel(pi, fs, "Reads and writes source files", "Node.js fs")
  Rel(pi, shell, "Executes bash commands", "child_process")
  Rel(pi, npm, "Installs Pi Packages", "npm/npx")
  Rel(ide, pi, "Sends prompts, receives events", "JSON-RPC over stdio (RPC mode)")
```

---

## External Dependencies

| System | Role | Trust |
|--------|------|-------|
| LLM Provider API | Generates responses and tool calls | External; controlled via API key |
| Local File System | Source of truth for code context | Trusted; pi operates in the developer's working directory |
| Shell / Bash | Executes agent-requested commands | **High risk** — runs as the developer's user |
| npm Registry | Distributes Extensions and Skills | Semi-trusted; verify packages before installation |
| IDE (RPC mode) | Drives pi programmatically | Trusted process on localhost |

---

## Security Note

`pi` runs bash commands **as the user who launched it**, with no sandboxing by default. This is intentional for a developer tool. Be aware when running `pi` in CI or automated pipelines.

---

**Back to:** [README](./README.md) | [Container View →](./c4-02-container.md)
